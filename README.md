-- PunchServer (Script inside ServerScriptService)

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local PunchHit = ReplicatedStorage:WaitForChild("PunchHit")

local POINTS_PER_PART = 10
local SIZE_PER_MILESTONE = 0.05
local MILESTONE_EVERY = 50
local RESPAWN_TIME = 120 -- seconds before part comes back

local playerPoints = {}

Players.PlayerAdded:Connect(function(player)
	playerPoints[player.UserId] = 0

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

	for _, part in pairs(character:GetDescendants()) do
		if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
			part.Size = part.Size + Vector3.new(SIZE_PER_MILESTONE, SIZE_PER_MILESTONE, SIZE_PER_MILESTONE)
		end
	end
end

local function destroyWithPhysics(part, puncher)
	-- Save original state and clone the part for respawning later
	local originalParent = part.Parent
	local originalAnchored = part.Anchored

	local clone = part:Clone()
	clone.Anchored = true
	clone.Transparency = 1
	clone.CanCollide = false
	clone.Parent = originalParent

	-- Unanchor and launch the part
	part.Anchored = false

	-- Direction away from the player who punched
	local direction = Vector3.new(0, 1, 0)
	local character = puncher.Character
	if character and character:FindFirstChild("HumanoidRootPart") then
		local punchDir = (part.Position - character.HumanoidRootPart.Position).Unit
		direction = punchDir + Vector3.new(0, 0.4, 0) -- slight upward angle
	end

	-- Apply launch velocity and random spin
	part.AssemblyLinearVelocity = direction * 60
	part.AssemblyAngularVelocity = Vector3.new(
		math.random(-8, 8),
		math.random(-8, 8),
		math.random(-8, 8)
	)

	-- Remove the flying part after a few seconds so it doesn't clutter the map
	task.delay(5, function()
		if part and part.Parent then
			part:Destroy()
		end
	end)

	-- Respawn the original part after RESPAWN_TIME seconds
	task.delay(RESPAWN_TIME, function()
		if clone and clone.Parent then
			clone.Transparency = 0
			clone.CanCollide = true
			clone.Anchored = originalAnchored
		end
	end)
end

PunchHit.OnServerEvent:Connect(function(player, targetPart)
	if not targetPart then return end
	if not targetPart:IsA("BasePart") then return end
	if not targetPart:FindFirstChild("Destructible") then return end

	local character = player.Character
	if not character then return end
	local rootPart = character:FindFirstChild("HumanoidRootPart")
	if not rootPart then return end

	local distance = (rootPart.Position - targetPart.Position).Magnitude
	if distance > 15 then return end

	-- Physics destruction instead of instant delete
	destroyWithPhysics(targetPart, player)

	-- Points
	local userId = player.UserId
	playerPoints[userId] = (playerPoints[userId] or 0) + POINTS_PER_PART

	local leaderstats = player:FindFirstChild("leaderstats")
	if leaderstats then
		local wrecks = leaderstats:FindFirstChild("Wreck Points")
		if wrecks then
			wrecks.Value = playerPoints[userId]
		end
	end

	if playerPoints[userId] % MILESTONE_EVERY == 0 then
		growPlayer(player)
	end
end)
