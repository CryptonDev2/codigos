local Players = game:GetService("Players")
local LogService = game:GetService("LogService")

local MAX_WALKSPEED = 16 
local SPEED_THRESHOLD = 8 
local REMOTE_LIMIT_PER_SECOND = 20 

local player_data = {}

local function monitorCharacter(player, character)
	local rootPart = character:WaitForChild("HumanoidRootPart", 5)
	local humanoid = character:WaitForChild("Humanoid", 5)
	
	if not rootPart or not humanoid then return end
	
	player_data[player.UserId].lastPosition = rootPart.Position
	player_data[player.UserId].lastCheck = tick()

	while character:IsDescendantOf(workspace) do
		task.wait(1) 
		
		if not rootPart or not rootPart.Parent then break end
		
		local currentPos = rootPart.Position
		local currentTime = tick()
		
		local lastPos = player_data[player.UserId].lastPosition
		local lastTime = player_data[player.UserId].lastCheck
		
		if lastPos and lastTime then
			local distance = (currentPos - Vector3.new(lastPos.X, currentPos.Y, lastPos.Z)).Magnitude 
			local timePassed = currentTime - lastTime
			
			local calculatedSpeed = distance / timePassed
			local maxAllowedDistance = (MAX_WALKSPEED + SPEED_THRESHOLD) * timePassed
			
			if distance > maxAllowedDistance and humanoid.SeatPart == nil then
				player_data[player.UserId].violations = player_data[player.UserId].violations + 1
				warn(("[Anticheat] %s sospecha de Teleport/Speed. Velocidad: %.2f"):format(player.Name, calculatedSpeed))
				
				if player_data[player.UserId].violations >= 3 then
					player:Kick("Detectado por comportamiento anómalo de movimiento (Desincronización).")
					break
				else
					rootPart.AssemblyLinearVelocity = Vector3.zero
					rootPart:PivotTo(CFrame.new(lastPos))
				end
			else
				player_data[player.UserId].lastPosition = currentPos
				player_data[player.UserId].lastCheck = currentTime
			end
		end
	end
end

local function monitorHealth(player, character)
	local humanoid = character:WaitForChild("Humanoid", 5)
	if not humanoid then return end
	
	humanoid.HealthChanged:Connect(function(health)
		if health > humanoid.MaxHealth then
			player:Kick("Modificación ilegal de estadísticas de salud.")
		end
	end)
end

Players.PlayerAdded:Connect(function(player)
	player_data[player.UserId] = {
		lastPosition = nil,
		lastCheck = nil,
		violations = 0,
		remotesCount = {},
		lastRemoteReset = tick()
	}
	
	player.CharacterAdded:Connect(function(character)
		task.spawn(monitorCharacter, player, character)
		task.spawn(monitorHealth, player, character)
	end)
end)

Players.PlayerRemoving:Connect(function(player)
	player_data[player.UserId] = nil
end)

local function verifyRemoteUsage(player, remoteName)
	local data = player_data[player.UserId]
	if not data then return false end
	
	local now = tick()
	if now - data.lastRemoteReset >= 1 then
		data.remotesCount = {}
		data.lastRemoteReset = now
	end
	
	data.remotesCount[remoteName] = (data.remotesCount[remoteName] or 0) + 1
	
	if data.remotesCount[remoteName] > REMOTE_LIMIT_PER_SECOND then
		player:Kick("Ejecución excesiva de eventos remotos (Spam/Crash Attempt).")
		return false
	end
	
	return true
end
