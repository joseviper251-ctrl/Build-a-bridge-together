# Build-a-bridge-together
--[[
	Auto-Chop + Auto-Place · Build a Bridge Together
	Platform Edition - Fixed Position
]]

-- ============================ CONFIG ============================
local CHOP_INTERVAL     = 0.85
local WALK_TO_TREES     = true
local WALK_RANGE        = 6
local WALK_TIMEOUT      = 10

local PLACE_COOLDOWN    = 0.05
local PLACE_LOOP_WAIT   = 0.05
local PLACE_REFIRE      = 0.3
local F_SPAM_INTERVAL   = 0.05
local STAND_HEIGHT      = 3
local MOVE_TIMEOUT      = 8

local PLATFORM_SIZE     = Vector3.new(8, 2, 8)
local PLATFORM_OFFSET   = 5  -- Studs below player (was 3.5, now lower)
-- ================================================================

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualInputManager = game:GetService("VirtualInputManager")
local RunService = game:GetService("RunService")

local lp = Players.LocalPlayer

-- Cleanup previous
if _G.AUTO_BRIDGE_CLEANUP then
	pcall(_G.AUTO_BRIDGE_CLEANUP)
end

-- State
local chopEnabled = false
local placeEnabled = false
local running = true
local treesDone = 0
local planksDone = 0

-- Services
local PlacePlank = ReplicatedStorage:WaitForChild("PlacePlank")
local Placed = workspace:WaitForChild("Planks"):WaitForChild("Placed")
local NotPlaced = workspace.Planks:WaitForChild("NotPlaced")

-- Platform reference
local safetyPlatform = nil
local platformConnection = nil

-- Load Rayfield
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
	Name = "Auto Bridge",
	LoadingTitle = "Build a bridge together",
	LoadingSubtitle = "Made By Nightshade",
	ConfigurationSaving = { Enabled = false },
	Discord = { Enabled = false },
	KeySystem = false
})

local MainTab = Window:CreateTab("Main", 4483362458)

-- ==================== PLATFORM FUNCTIONS ====================
local function removePlatform()
	if platformConnection then
		platformConnection:Disconnect()
		platformConnection = nil
	end
	if safetyPlatform then
		safetyPlatform:Destroy()
		safetyPlatform = nil
		print("[AutoBridge] Platform removed")
	end
end

local function createPlatform()
	removePlatform()
	
	local char = lp.Character
	local root = char and char:FindFirstChild("HumanoidRootPart")
	if not root then 
		print("[AutoBridge] ERROR: No character found")
		return 
	end
	
	-- Calculate position BELOW feet (not at feet)
	-- Platform top will be at feet level, center is lower
	local platformPos = root.Position - Vector3.new(0, PLATFORM_OFFSET, 0)
	
	-- Create platform
	safetyPlatform = Instance.new("Part")
	safetyPlatform.Name = "SafetyPlatform_" .. lp.Name
	safetyPlatform.Size = PLATFORM_SIZE
	safetyPlatform.Anchored = true
	safetyPlatform.CanCollide = true
	safetyPlatform.Transparency = 0.5
	safetyPlatform.Color = Color3.fromRGB(0, 255, 100)
	safetyPlatform.Material = Enum.Material.Neon
	safetyPlatform.CastShadow = false
	safetyPlatform.CFrame = CFrame.new(platformPos)
	safetyPlatform.Parent = workspace
	
	print("[AutoBridge] Platform created at: " .. tostring(platformPos))
	
	-- Update platform position to follow player (X and Z only, keep same Y offset)
	local initialY = platformPos.Y
	
	platformConnection = RunService.Heartbeat:Connect(function()
		if not safetyPlatform or not placeEnabled then 
			removePlatform()
			return 
		end
		
		local char = lp.Character
		local root = char and char:FindFirstChild("HumanoidRootPart")
		if root then
			-- Only follow X and Z, keep Y at fixed offset below initial spawn
			-- This prevents the platform from pushing player up when they jump
			local newPos = Vector3.new(root.Position.X, initialY, root.Position.Z)
			safetyPlatform.CFrame = CFrame.new(newPos)
		else
			removePlatform()
		end
	end)
end
-- =============================================================

-- Toggles
local ChopToggle = MainTab:CreateToggle({
	Name = "Auto Chop (Fast)",
	CurrentValue = false,
	Flag = "AutoChop",
	Callback = function(Value)
		chopEnabled = Value
	end
})

local PlaceToggle = MainTab:CreateToggle({
	Name = "Auto Place (Platform)",
	CurrentValue = false,
	Flag = "AutoPlace",
	Callback = function(Value)
		placeEnabled = Value
		if Value then
			print("[AutoBridge] Place enabled - creating platform...")
			createPlatform()
		else
			print("[AutoBridge] Place disabled - removing platform...")
			removePlatform()
		end
	end
})

-- Button: Instant Proximity Prompts
MainTab:CreateButton({
	Name = "Instant Prompts (No Hold)",
	Callback = function()
		for i, v in pairs(game:GetService("Workspace"):GetDescendants()) do
			if v:IsA("ProximityPrompt") then
				v.HoldDuration = 0
			end
		end
		
		game:GetService("ProximityPromptService").PromptButtonHoldBegan:Connect(function(v)
			v.HoldDuration = 0
		end)
		
		print("[AutoBridge] Instant Proximity Prompts activated!")
	end,
})

-- Stats
MainTab:CreateSection("Stats")
local TreesLabel = MainTab:CreateLabel("Trees: 0")
local PlanksLabel = MainTab:CreateLabel("Planks: 0")

-- Helper functions
local function getAxe(char)
	local axe = char:FindFirstChild("Axe") or (lp.Backpack and lp.Backpack:FindFirstChild("Axe"))
	if axe and axe.Parent ~= char and char:FindFirstChild("Humanoid") then
		char.Humanoid:EquipTool(axe)
		task.wait(0.2)
		axe = char:FindFirstChild("Axe")
	end
	return axe
end

local function nearestTree(root)
	local best, bestDist = nil, math.huge
	local rootPos = root.Position
	
	for _, tree in ipairs(workspace.Trees:GetChildren()) do
		local standing = tree:FindFirstChild("Standing")
		local trunk = tree:FindFirstChild("Trunk")
		if standing and trunk and standing.Value then
			local pivot = trunk:GetPivot().Position
			local d = (pivot - rootPos).Magnitude
			if d < bestDist then
				best, bestDist = tree, d
			end
		end
	end
	return best, bestDist
end

local function pressF()
	VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.F, false, game)
	task.wait(0.01)
	VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.F, false, game)
end

-- Simple MoveTo function
local function moveToTarget(humanoid, targetPos, timeout)
	local startTime = os.clock()
	humanoid:MoveTo(targetPos)
	
	while running and placeEnabled do
		task.wait(0.05)
		
		if not humanoid or not humanoid.RootPart then return false end
		
		local dist = (humanoid.RootPart.Position - targetPos).Magnitude
		if dist <= 1.5 then return true end
		
		if os.clock() - startTime > timeout then return false end
		
		if (os.clock() - startTime) % 1.0 < 0.06 then
			humanoid:MoveTo(targetPos)
		end
	end
	
	return false
end

-- Place logic helpers
local firedRecently = {}
local currentTarget = nil

local function getPlaceable()
	local list = {}
	for _, p in ipairs(NotPlaced:GetChildren()) do
		local prev = p:FindFirstChild("Previous")
		if prev and prev.Value and prev.Value.Parent == Placed then
			local t = firedRecently[p]
			if not t or os.clock() - t > PLACE_REFIRE then
				table.insert(list, p)
			end
		end
	end
	return list
end

-- Cleanup
_G.AUTO_BRIDGE_CLEANUP = function()
	running = false
	chopEnabled = false
	placeEnabled = false
	removePlatform()
	Rayfield:Destroy()
	_G.AUTO_BRIDGE_CLEANUP = nil
end

-- F SPAM LOOP
task.spawn(function()
	while running do
		if placeEnabled then
			pressF()
		end
		task.wait(F_SPAM_INTERVAL)
	end
end)

-- FAST CHOP LOOP
task.spawn(function()
	while running do
		if not chopEnabled then
			task.wait(0.1)
			continue
		end
		
		local char = lp.Character
		local root = char and char:FindFirstChild("HumanoidRootPart")
		local humanoid = char and char:FindFirstChild("Humanoid")
		if not (root and humanoid) then
			task.wait(0.1)
			continue
		end

		local tree, dist = nearestTree(root)
		if not tree then
			task.wait(0.2)
			continue
		end
		
		local trunk = tree:FindFirstChild("Trunk")
		if not trunk then continue end

		-- Fast walk to tree
		if WALK_TO_TREES and dist > WALK_RANGE then
			local t0 = os.clock()
			local targetPos = trunk:GetPivot().Position
			humanoid:MoveTo(targetPos)
			
			while running and chopEnabled and tree.Parent do
				task.wait(0.05)
				dist = (trunk:GetPivot().Position - root.Position).Magnitude
				if dist <= WALK_RANGE then break end
				if os.clock() - t0 > WALK_TIMEOUT then break end
				if (os.clock() - t0) % 1.5 < 0.06 then
					humanoid:MoveTo(trunk:GetPivot().Position)
				end
			end
		end

		-- Rapid fire chopping
		while running and chopEnabled and tree.Parent do
			local standing = tree:FindFirstChild("Standing")
			if not standing or not standing.Value then break end
			
			local axe = getAxe(char)
			if not axe then
				task.wait(0.5)
				continue
			end
			
			axe.Hit:FireServer()
			
			VirtualInputManager:SendMouseButtonEvent(0, 0, 0, true, game, 1)
			VirtualInputManager:SendMouseButtonEvent(0, 0, 0, false, game, 1)
			
			task.wait(CHOP_INTERVAL)
		end

		if tree:FindFirstChild("Standing") and not tree.Standing.Value then
			treesDone += 1
			TreesLabel:Set("Trees: " .. treesDone)
		end
	end
end)

-- PLACE LOOP (Simple MoveTo with Platform)
task.spawn(function()
	local lastFireAt = 0
	
	while running do
		task.wait(PLACE_LOOP_WAIT)

		if not placeEnabled then
			currentTarget = nil
			continue
		end

		local char = lp.Character
		local root = char and char:FindFirstChild("HumanoidRootPart")
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if not root or not hum or hum.Health <= 0 then continue end

		local myPos = root.Position
		local placeable = getPlaceable()

		if #placeable > 0 then
			-- Get nearest ghost plank
			table.sort(placeable, function(a, b)
				return (a.Position - myPos).Magnitude < (b.Position - myPos).Magnitude
			end)
			
			local target = placeable[1]
			currentTarget = target
			
			local prevPlank = target:FindFirstChild("Previous")
			prevPlank = prevPlank and prevPlank.Value
			if not prevPlank or not prevPlank.Parent then continue end

			-- Calculate position ON TOP of the ghost plank
			local standPos = target.Position + Vector3.new(0, STAND_HEIGHT, 0)
			local dist = (standPos - myPos).Magnitude

			-- Simple MoveTo
			if dist > 1 then
				moveToTarget(hum, standPos, MOVE_TIMEOUT)
			else
				-- We're standing on it - FIRE!
				if os.clock() - lastFireAt >= PLACE_COOLDOWN and target.Parent == NotPlaced then
					PlacePlank:FireServer(target)
					firedRecently[target] = os.clock()
					lastFireAt = os.clock()
					planksDone += 1
					PlanksLabel:Set("Planks: " .. planksDone)
				end
			end
			
		elseif currentTarget and currentTarget.Parent then
			-- Keep standing on current target until it places
			local standPos = currentTarget.Position + Vector3.new(0, STAND_HEIGHT, 0)
			if (root.Position - standPos).Magnitude > 2 then
				moveToTarget(hum, standPos, MOVE_TIMEOUT)
			end
		else
			currentTarget = nil
		end
	end
end)

print("[AutoBridge] Loaded - Platform won't push you up")
