-- PunchServer (Script inside ServerScriptService)
-- This handles destruction, wreck points, and size scaling

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local PunchHit = ReplicatedStorage:WaitForChild("PunchHit")

-- How many wreck points each destroyed part gives
local POINTS_PER_PART = 10

-- How much size increases per point milestone
local SIZE_PER_MILESTONE = 0.05 -- added to scale each milestone
local MILESTONE_EVERY = 50       -- every 50 points, you grow

-- Store each player's points
local playerPoints = {}

Players.PlayerAdded:Connect(function(player)
	playerPoints[player.UserId] = 0

	-- Create a leaderstat so points show on the leaderboard
	local leaderstats = Instance.new("Folder")
	leaderstats.Name = "leaderstats"
	leaderstats.Parent = player

	local wrecks = Instance.new("IntValue")
	wrecks.Name = "Wreck Points"
	wrecks.Value = 0
	wrecks.Parent = leaderstats
end)

Players.PlayerRemoving:Connect(function(player)
	playerPoints[player.UserId] = nil
end)

local function growPlayer(player)
	local character = player.Character
	if not character then return end
	local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
	if not humanoidRootPart then return end

	-- Scale every body part up slightly
	for _, part in pairs(character:GetDescendants()) do
		if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
			part.Size = part.Size + Vector3.new(SIZE_PER_MILESTONE, SIZE_PER_MILESTONE, SIZE_PER_MILESTONE)
		end
	end
end

PunchHit.OnServerEvent:Connect(function(player, targetPart)
	-- Safety checks
	if not targetPart then return end
	if not targetPart:IsA("BasePart") then return end

	-- Check the part is tagged as destructible
	-- In Studio, add a BoolValue named "Destructible" inside any part you want breakable
	if not targetPart:FindFirstChild("Destructible") then return end

	-- Check punch range (prevent exploiting from far away)
	local character = player.Character
	if not character then return end
	local rootPart = character:FindFirstChild("HumanoidRootPart")
	if not rootPart then return end

	local distance = (rootPart.Position - targetPart.Position).Magnitude
	if distance > 15 then return end -- must be within 15 studs

	-- Destroy the part
	targetPart:Destroy()

	-- Add points
	local userId = player.UserId
	playerPoints[userId] = (playerPoints[userId] or 0) + POINTS_PER_PART

	-- Update leaderboard
	local leaderstats = player:FindFirstChild("leaderstats")
	if leaderstats then
		local wrecks = leaderstats:FindFirstChild("Wreck Points")
		if wrecks then
			wrecks.Value = playerPoints[userId]
		end
	end

	-- Check if player hit a growth milestone
	if playerPoints[userId] % MILESTONE_EVERY == 0 then
		growPlayer(player)
	end
end)
