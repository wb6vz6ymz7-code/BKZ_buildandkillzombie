-- =====================================================
-- BKZ Drive | HUMAN 锁敌（恢复可打怪）| Drive UI | X -55~55 不锁
-- =====================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local LP = Players.LocalPlayer

local Z_MIN, Z_MAX = -771, -407
local Z_MID = (Z_MIN + Z_MAX) * 0.5

local CONFIG = {
	Enabled = false,
	Speed = 200,
	RideHeight = 3.5,
	StickForce = 0.4,
	GroundRay = 150,
	NoGroundLift = 35,
	VoidTime = 0.35,
	VoidPullSpeed = 1.4,
	ZBand = 80,
	ZCorrectMax = 0.18,
	ZSmooth = 0.06,
	Mode = "ZOMBIE",
	NpcRadius = 350,
	Noclip = true,
	DebugNPC = true,
	ChaseBoost = 1.4,
	UseNoLockX = true,
	NoLockXMin = -55,
	NoLockXMax = 55,
}

getgenv().BKZ_CFG = CONFIG

local savedCollide = {}
local startX = nil
local lastZComp = 0
local lastGroundPos = nil
local noGroundSince = nil
local npcCache = { part = nil, untilT = 0 }
local lastNpcLog = 0

local function getModelPart(m)
	if not m then return nil end
	if m:IsA("BasePart") then return m end
	return m.PrimaryPart
		or m:FindFirstChild("HumanoidRootPart")
		or m:FindFirstChild("Torso")
		or m:FindFirstChild("UpperTorso")
		or m:FindFirstChild("Head")
		or m:FindFirstChild("Root")
		or m:FindFirstChild("RootPart")
		or m:FindFirstChildWhichIsA("BasePart", true)
end

local function findCar()
	local char = LP.Character
	if not char then return nil, nil end
	local hum = char:FindFirstChildOfClass("Humanoid")
	if hum and hum.SeatPart then
		local seat = hum.SeatPart
		local model = seat:FindFirstAncestorOfClass("Model")
		if model and model ~= char then
			return model, getModelPart(model) or seat
		end
		return seat.Parent, seat
	end
	return nil, nil
end

local function isMyCar(m)
	local car = select(1, findCar())
	return car and (m == car or (m and m:IsDescendantOf(car)))
end

-- 与「能打到所有怪物」相同：任意非玩家 Humanoid
local function isEnemyModel(m)
	if not m or not m:IsA("Model") then return false end
	if m == LP.Character or (LP.Character and m:IsDescendantOf(LP.Character)) then return false end
	if Players:GetPlayerFromCharacter(m) then return false end
	if isMyCar(m) then return false end

	local hum = m:FindFirstChildOfClass("Humanoid")
	if hum and hum.Health > 0 then return true end
	for _, d in ipairs(m:GetDescendants()) do
		if d:IsA("Humanoid") and d.Health > 0 then
			local owner = Players:GetPlayerFromCharacter(d.Parent)
			if not owner then return true end
		end
	end
	return false
end

local function inNoLockX(pos)
	if not CONFIG.UseNoLockX then return false end
	return pos.X >= CONFIG.NoLockXMin and pos.X <= CONFIG.NoLockXMax
end

local function nearestEnemy(origin, radius)
	if npcCache.part and os.clock() < npcCache.untilT then
		local p = npcCache.part
		if p.Parent and (p.Position - origin).Magnitude <= radius and not inNoLockX(p.Position) then
			return p
		end
	end

	local best, bestD, bestName = nil, radius, nil
	local scanned, skippedX = 0, 0

	local function consider(m)
		if not isEnemyModel(m) then return end
		local p = getModelPart(m)
		if not p then return end
		if inNoLockX(p.Position) then
			skippedX += 1
			return
		end
		scanned += 1
		local d = (p.Position - origin).Magnitude
		if d < bestD and d > 1.5 then
			bestD, best, bestName = d, p, m.Name
		end
	end

	-- 全图 Humanoid（与可打怪版相同）
	for _, inst in ipairs(Workspace:GetDescendants()) do
		if inst:IsA("Humanoid") and inst.Health > 0 then
			local m = inst:FindFirstAncestorOfClass("Model")
			if m then consider(m) end
		end
	end

	if CONFIG.DebugNPC and os.clock() - lastNpcLog > 0.7 then
		lastNpcLog = os.clock()
		if best then
			print(string.format("[BKZ] LOCK %s dist=%.1f x=%.1f", bestName, bestD, best.Position.X))
		else
			print(string.format("[BKZ] NO LOCK r=%.0f found=%d skipX=%d", radius, scanned, skippedX))
		end
	end

	npcCache.part = best
	npcCache.untilT = os.clock() + 0.08
	return best, bestD
end

local function setNoClip(model, char, on)
	local function apply(inst)
		if not inst then return end
		for _, p in ipairs(inst:GetDescendants()) do
			if p:IsA("BasePart") then
				if on then
					if savedCollide[p] == nil then savedCollide[p] = p.CanCollide end
					p.CanCollide = false
				else
					if savedCollide[p] ~= nil then
						p.CanCollide = savedCollide[p]
						savedCollide[p] = nil
					end
				end
			end
		end
	end
	apply(model)
	apply(char)
end

local function restoreCollide()
	for p, val in pairs(savedCollide) do
		pcall(function()
			if p and p.Parent then p.CanCollide = val end
		end)
		savedCollide[p] = nil
	end
end

local function probeGround(part, car, char)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { car, char }
	local best
	for _, off in ipairs({
		Vector3.new(0, 4, 0), Vector3.new(6, 4, 0), Vector3.new(-6, 4, 0),
		Vector3.new(0, 4, 6), Vector3.new(0, 4, -6),
	}) do
		local hit = Workspace:Raycast(part.Position + off, Vector3.new(0, -CONFIG.GroundRay, 0), params)
		if hit and (not best or hit.Position.Y > best.Position.Y) then best = hit end
	end
	return best
end

local function captureStartX()
	local _, part = findCar()
	local hrp = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
	local p = (part and part.Position) or (hrp and hrp.Position)
	if p then
		startX = p.X
		lastGroundPos = p
		print(string.format("[BKZ] start X=%.1f | noLock X [%.0f,%.0f]", startX, CONFIG.NoLockXMin, CONFIG.NoLockXMax))
	end
	lastZComp = 0
	noGroundSince = nil
	npcCache.untilT = 0
end

RunService.Heartbeat:Connect(function(dt)
	if not CONFIG.Enabled or CONFIG.Speed <= 0 then return end

	local car, part = findCar()
	local char = LP.Character
	if not part then return end

	if startX == nil then captureStartX() end
	if CONFIG.Noclip then setNoClip(car, char, true) end

	local pos = part.Position
	local hit = probeGround(part, car, char)

	local inVoid = false
	if hit then
		noGroundSince = nil
		lastGroundPos = Vector3.new(pos.X, hit.Position.Y + CONFIG.RideHeight, math.clamp(pos.Z, Z_MIN, Z_MAX))
	else
		if not noGroundSince then
			noGroundSince = os.clock()
		elseif os.clock() - noGroundSince >= CONFIG.VoidTime then
			inVoid = true
		end
	end

	local desired
	local chasing = false

	if inVoid and lastGroundPos then
		local flat = Vector3.new(lastGroundPos.X - pos.X, 0, lastGroundPos.Z - pos.Z)
		desired = flat.Magnitude > 2 and flat.Unit or Vector3.new(-1, 0, 0)
		lastZComp = 0
	else
		if CONFIG.Mode == "ZOMBIE" then
			local enemy = nearestEnemy(pos, CONFIG.NpcRadius)
			if enemy then
				chasing = true
				local to = Vector3.new(enemy.Position.X - pos.X, 0, enemy.Position.Z - pos.Z)
				if to.Magnitude > 0.3 then
					desired = to.Unit
				else
					desired = Vector3.new(1, 0, 0)
				end
				lastZComp = 0
			end
		end

		if not chasing then
			local zErr = pos.Z - Z_MID
			local half = CONFIG.ZBand
			local targetZComp = 0
			if zErr > half then
				targetZComp = -math.clamp((zErr - half) / math.max(half, 1), 0, 1) * CONFIG.ZCorrectMax
			elseif zErr < -half then
				targetZComp = math.clamp((-half - zErr) / math.max(half, 1), 0, 1) * CONFIG.ZCorrectMax
			end
			lastZComp = lastZComp + (targetZComp - lastZComp) * CONFIG.ZSmooth
			desired = Vector3.new(1, 0, lastZComp)
			if desired.Magnitude > 0 then desired = desired.Unit end
		end
	end

	local vy, speedMul = 0, 1
	if hit then
		vy = ((hit.Position.Y + CONFIG.RideHeight) - pos.Y) * 8
	else
		vy = CONFIG.NoGroundLift
		if inVoid then
			vy = CONFIG.NoGroundLift * 2.5
			speedMul = CONFIG.VoidPullSpeed
		end
	end

	local boost = chasing and CONFIG.ChaseBoost or 1
	local vx = desired.X * CONFIG.Speed * speedMul * boost
	local vz = desired.Z * CONFIG.Speed * speedMul * boost
	if not chasing and not inVoid then
		vx = math.max(vx, CONFIG.Speed * 0.35)
	end

	local vel = Vector3.new(vx, vy, vz)
	local look = Vector3.new(desired.X, 0, desired.Z)
	if look.Magnitude < 0.05 then look = Vector3.new(1, 0, 0) else look = look.Unit end

	local y = pos.Y
	if hit then
		y = pos.Y + ((hit.Position.Y + CONFIG.RideHeight) - pos.Y) * CONFIG.StickForce
	elseif inVoid and lastGroundPos then
		y = pos.Y + (math.max(lastGroundPos.Y, pos.Y + 5) - pos.Y) * 0.25
	end

	local z
	if chasing then
		z = math.clamp(pos.Z + desired.Z * CONFIG.Speed * dt * 0.5, Z_MIN, Z_MAX)
	else
		z = math.clamp(pos.Z, Z_MIN, Z_MAX)
	end

	local usePos = Vector3.new(pos.X, y, z)
	if inVoid and lastGroundPos then
		local dist = (Vector3.new(pos.X, 0, pos.Z) - Vector3.new(lastGroundPos.X, 0, lastGroundPos.Z)).Magnitude
		if dist < 12 then
			usePos = Vector3.new(lastGroundPos.X - 8, math.max(pos.Y, lastGroundPos.Y), math.clamp(lastGroundPos.Z, Z_MIN, Z_MAX))
		end
	end

	pcall(function()
		part.AssemblyLinearVelocity = vel
		part.AssemblyAngularVelocity = Vector3.zero
		part.CFrame = CFrame.new(usePos, usePos + look)
	end)
	if car and car.PrimaryPart and car.PrimaryPart ~= part then
		pcall(function()
			car.PrimaryPart.AssemblyLinearVelocity = vel
			car.PrimaryPart.CFrame = CFrame.new(usePos, usePos + look)
		end)
	end
end)

-- UI：仅 Drive
local WindUI
pcall(function()
	WindUI = loadstring(game:HttpGet("https://raw.githubusercontent.com/Footagesus/WindUI/main/dist/main.lua"))()
end)
if not WindUI then
	pcall(function()
		WindUI = loadstring(game:HttpGet("https://raw.githubusercontent.com/kiciahook/WindUI/refs/heads/main/dist/main.lua"))()
	end)
end

if WindUI then
	local Window = WindUI:CreateWindow({
		Title = "BKZ Drive",
		Icon = "car",
		Author = "HUMAN lock fixed",
		Folder = "BKZDrive",
		Size = UDim2.fromOffset(480, 360),
		Transparent = true,
		Theme = "Dark",
		SideBarWidth = 120,
	})

	local Main = Window:Tab({ Title = "Drive", Icon = "gauge" })

	Main:Toggle({
		Title = "Enable Drive",
		Value = false,
		Callback = function(v)
			CONFIG.Enabled = v
			if v then captureStartX() else restoreCollide(); startX = nil end
		end,
	})
	Main:Slider({
		Title = "Speed",
		Value = { Min = 0, Max = 1000, Default = 200 },
		Callback = function(v) CONFIG.Speed = v end,
	})
	Main:Dropdown({
		Title = "Mode",
		Values = { "FORWARD", "ZOMBIE" },
		Value = "ZOMBIE",
		Callback = function(v) CONFIG.Mode = v end,
	})
	Main:Slider({
		Title = "NPC Radius",
		Value = { Min = 20, Max = 500, Default = 350 },
		Callback = function(v) CONFIG.NpcRadius = v end,
	})
	Main:Toggle({
		Title = "Noclip",
		Value = true,
		Callback = function(v) CONFIG.Noclip = v end,
	})
else
	warn("[BKZ] WindUI failed")
end

print("[BKZ] HUMAN restored | X -55~55 skip | Drive UI only")
