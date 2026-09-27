-- =====================================================
-- BKZ Drive | 打所有实体 | 不打玩家/别人车 | X -55~55 不锁
-- Toggle UI: RightControl
-- =====================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local UIS = game:GetService("UserInputService")
local LP = Players.LocalPlayer

local Z_MIN, Z_MAX = -771, -407
local Z_MID = (Z_MIN + Z_MAX) * 0.5

local CONFIG = {
	Enabled = false,
	Speed = 220,
	RideHeight = 3.5,
	StickForce = 0.4,
	GroundRay = 150,
	NoGroundLift = 35,
	VoidTime = 0.35,
	VoidPullSpeed = 1.4,
	ZBand = 80,
	ZCorrectMax = 0.18,
	ZSmooth = 0.06,
	Mode = "ZOMBIE", -- = 打实体
	NpcRadius = 400,
	Noclip = true,
	DebugNPC = true,
	ChaseBoost = 1.8,
	UseNoLockX = true,
	NoLockXMin = -55,
	NoLockXMax = 55,
	ToggleKey = Enum.KeyCode.RightControl,
	-- 实体最小体积，过滤地面碎屑
	MinPartSize = 1.2,
}

getgenv().BKZ_CFG = CONFIG

local KEY_MAP = {
	RightControl = Enum.KeyCode.RightControl,
	LeftControl = Enum.KeyCode.LeftControl,
	RightShift = Enum.KeyCode.RightShift,
	LeftShift = Enum.KeyCode.LeftShift,
	RightAlt = Enum.KeyCode.RightAlt,
	LeftAlt = Enum.KeyCode.LeftAlt,
	P = Enum.KeyCode.P,
	K = Enum.KeyCode.K,
	Insert = Enum.KeyCode.Insert,
	End = Enum.KeyCode.End,
	F8 = Enum.KeyCode.F8,
	F9 = Enum.KeyCode.F9,
}

local savedCollide = {}
local startX, lastZComp = nil, 0
local lastGroundPos, noGroundSince = nil, nil
local npcCache = { part = nil, untilT = 0, name = nil }
local lastNpcLog = 0
local Window, uiVisible = nil, true

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
		or m:FindFirstChild("Chassis")
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

-- 是否属于「某个玩家的角色」
local function isPlayerCharacter(model)
	if not model then return false end
	if Players:GetPlayerFromCharacter(model) then return true end
	local m = model:FindFirstAncestorOfClass("Model")
	while m do
		if Players:GetPlayerFromCharacter(m) then return true end
		m = m.Parent and m.Parent:FindFirstAncestorOfClass("Model")
	end
	return false
end

-- 是否是载具（有座椅 / VehicleSeat / 常见车名）
local function isVehicleModel(m)
	if not m or not m:IsA("Model") then return false end
	if m:FindFirstChildWhichIsA("VehicleSeat", true) then return true end
	if m:FindFirstChildWhichIsA("Seat", true) then
		-- 有 Seat + 多零件，当车
		local parts = 0
		for _, d in ipairs(m:GetDescendants()) do
			if d:IsA("BasePart") then parts += 1 if parts > 3 then break end end
		end
		if parts > 3 then return true end
	end
	local n = string.lower(m.Name)
	if n:find("car") or n:find("vehicle") or n:find("chassis") or n:find("kart") or n:find("truck") then
		return true
	end
	return false
end

-- 别人的车 / 自己的车 / 玩家
local function isProtected(inst)
	if not inst then return true end

	-- 自己角色
	if LP.Character and (inst == LP.Character or inst:IsDescendantOf(LP.Character)) then
		return true
	end

	-- 自己的车
	local myCar = select(1, findCar())
	if myCar and (inst == myCar or inst:IsDescendantOf(myCar)) then
		return true
	end

	-- 任意玩家角色
	local model = inst:IsA("Model") and inst or inst:FindFirstAncestorOfClass("Model")
	if model and isPlayerCharacter(model) then
		return true
	end
	for _, plr in ipairs(Players:GetPlayers()) do
		if plr.Character and inst:IsDescendantOf(plr.Character) then
			return true
		end
	end

	-- 别人的车：有座位的 Model，且不是自己的车
	if model and isVehicleModel(model) then
		if myCar and model == myCar then return true end
		-- 有玩家坐在上面
		for _, d in ipairs(model:GetDescendants()) do
			if d:IsA("VehicleSeat") or d:IsA("Seat") then
				local occ = d.Occupant
				if occ then
					local ch = occ.Parent
					if ch and Players:GetPlayerFromCharacter(ch) then
						return true
					end
				end
			end
		end
		-- 名字带玩家名 / 在玩家文件夹下
		for _, plr in ipairs(Players:GetPlayers()) do
			if plr ~= LP and string.find(string.lower(model.Name), string.lower(plr.Name), 1, true) then
				return true
			end
		end
		-- 保险：所有载具都不当攻击目标（只碾怪实体，不追别人车）
		return true
	end

	return false
end

local function inNoLockX(pos)
	return CONFIG.UseNoLockX and pos.X >= CONFIG.NoLockXMin and pos.X <= CONFIG.NoLockXMax
end

-- 从 Model 取一个代表 Part
local function entityPart(m)
	if isProtected(m) then return nil end
	local p = getModelPart(m)
	if not p then return nil end
	if inNoLockX(p.Position) then return nil end
	-- 过滤太小的装饰
	local s = p.Size
	if math.max(s.X, s.Y, s.Z) < CONFIG.MinPartSize then return nil end
	return p
end

--[[
  所有实体策略：
  1) 优先：带 Humanoid 的非玩家 Model
  2) 其次：名字像怪的 Model
  3) 再：其它「像活物」的 Model（有多个 Part、不在地面层）
]]
local function nearestEntity(origin, radius)
	if npcCache.part and os.clock() < npcCache.untilT then
		local p = npcCache.part
		if p and p.Parent and (p.Position - origin).Magnitude <= radius and not inNoLockX(p.Position) then
			return p
		end
	end

	local best, bestD, bestName, bestPri = nil, radius, nil, 99
	local scanned, skipped = 0, 0

	local function consider(model, priority)
		if not model or not model:IsA("Model") then return end
		if isProtected(model) then skipped += 1 return end
		local p = entityPart(model)
		if not p then return end
		local d = (p.Position - origin).Magnitude
		if d > radius or d < 0.5 then return end
		scanned += 1
		-- 优先级数字越小越好；同优先级比距离
		if priority < bestPri or (priority == bestPri and d < bestD) then
			bestPri, bestD, best, bestName = priority, d, p, model.Name
		end
	end

	for _, inst in ipairs(Workspace:GetDescendants()) do
		if inst:IsA("Humanoid") and inst.Health > 0 then
			local m = inst:FindFirstAncestorOfClass("Model")
			if m then consider(m, 1) end
		elseif inst:IsA("Model") then
			local n = string.lower(inst.Name)
			if n:find("zomb") or n:find("npc") or n:find("enemy") or n:find("mob")
				or n:find("walker") or n:find("infected") or n:find("runner")
				or n:find("undead") or n:find("monster") then
				consider(inst, 2)
			end
		end
	end

	-- 若上面一个都没有：扫「非保护、有 PrimaryPart/多 Part」的 Model 当实体
	if not best then
		for _, inst in ipairs(Workspace:GetChildren()) do
			if inst:IsA("Model") then
				consider(inst, 5)
			elseif inst:IsA("Folder") then
				for _, c in ipairs(inst:GetChildren()) do
					if c:IsA("Model") then consider(c, 5) end
				end
			end
		end
	end

	if CONFIG.DebugNPC and os.clock() - lastNpcLog > 0.55 then
		lastNpcLog = os.clock()
		if best then
			print(string.format("[BKZ] ENTITY %s dist=%.1f x=%.1f pri=%d", bestName, bestD, best.Position.X, bestPri))
		else
			print(string.format("[BKZ] NO ENTITY scanned=%d skipProtect=%d r=%.0f", scanned, skipped, radius))
		end
	end

	npcCache.part = best
	npcCache.name = bestName
	npcCache.untilT = os.clock() + 0.1
	return best
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
		Vector3.new(0, 4, 0), Vector3.new(8, 4, 0), Vector3.new(-8, 4, 0),
		Vector3.new(0, 4, 8), Vector3.new(0, 4, -8),
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
	end
	lastZComp, noGroundSince = 0, nil
	npcCache.untilT = 0
end

local function setUIVisible(vis)
	uiVisible = vis
	pcall(function()
		local list = { LP:FindFirstChild("PlayerGui"), game:GetService("CoreGui") }
		if gethui then table.insert(list, gethui()) end
		for _, parent in ipairs(list) do
			if parent then
				for _, g in ipairs(parent:GetChildren()) do
					if g:IsA("ScreenGui") then
						local n = string.lower(g.Name)
						if n:find("wind") or n:find("bkz") then g.Enabled = vis end
					end
				end
			end
		end
	end)
end

UIS.InputBegan:Connect(function(input)
	if input.KeyCode == CONFIG.ToggleKey then setUIVisible(not uiVisible) end
end)

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
		if not noGroundSince then noGroundSince = os.clock()
		elseif os.clock() - noGroundSince >= CONFIG.VoidTime then inVoid = true end
	end

	local desired, chasing = nil, false

	if inVoid and lastGroundPos then
		local flat = Vector3.new(lastGroundPos.X - pos.X, 0, lastGroundPos.Z - pos.Z)
		desired = flat.Magnitude > 2 and flat.Unit or Vector3.new(-1, 0, 0)
	else
		if CONFIG.Mode == "ZOMBIE" then
			local ent = nearestEntity(pos, CONFIG.NpcRadius)
			if ent then
				chasing = true
				local to = Vector3.new(ent.Position.X - pos.X, 0, ent.Position.Z - pos.Z)
				desired = to.Magnitude > 0.15 and to.Unit or Vector3.new(1, 0, 0)
				lastZComp = 0
			end
		end
		if not chasing then
			local zErr = pos.Z - Z_MID
			local half = CONFIG.ZBand
			local t = 0
			if zErr > half then t = -math.clamp((zErr - half) / half, 0, 1) * CONFIG.ZCorrectMax
			elseif zErr < -half then t = math.clamp((-half - zErr) / half, 0, 1) * CONFIG.ZCorrectMax end
			lastZComp = lastZComp + (t - lastZComp) * CONFIG.ZSmooth
			desired = Vector3.new(1, 0, lastZComp)
			if desired.Magnitude > 0 then desired = desired.Unit end
		end
	end

	local vy, speedMul = 0, 1
	if hit then
		vy = ((hit.Position.Y + CONFIG.RideHeight) - pos.Y) * 8
	else
		vy = CONFIG.NoGroundLift
		if inVoid then vy = CONFIG.NoGroundLift * 2.5; speedMul = CONFIG.VoidPullSpeed end
	end

	local boost = chasing and CONFIG.ChaseBoost or 1
	local spd = CONFIG.Speed * speedMul * boost
	local vx, vz = desired.X * spd, desired.Z * spd
	if not chasing and not inVoid then vx = math.max(vx, CONFIG.Speed * 0.4) end

	local look = Vector3.new(desired.X, 0, desired.Z)
	if look.Magnitude < 0.05 then look = Vector3.new(1, 0, 0) else look = look.Unit end

	local y = pos.Y
	if hit then y = pos.Y + ((hit.Position.Y + CONFIG.RideHeight) - pos.Y) * CONFIG.StickForce end

	local usePos = Vector3.new(pos.X, y, math.clamp(pos.Z, Z_MIN, Z_MAX))
	if chasing then
		usePos = Vector3.new(
			pos.X + desired.X * spd * dt * 0.9,
			y,
			math.clamp(pos.Z + desired.Z * spd * dt * 0.9, Z_MIN, Z_MAX)
		)
	end
	if inVoid and lastGroundPos then
		local dist = (Vector3.new(pos.X, 0, pos.Z) - Vector3.new(lastGroundPos.X, 0, lastGroundPos.Z)).Magnitude
		if dist < 12 then
			usePos = Vector3.new(lastGroundPos.X - 8, math.max(pos.Y, lastGroundPos.Y), math.clamp(lastGroundPos.Z, Z_MIN, Z_MAX))
		end
	end

	local vel = Vector3.new(vx, vy, vz)
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
	Window = WindUI:CreateWindow({
		Title = "BKZ Drive",
		Icon = "car",
		Author = "all entities | no players/cars",
		Folder = "BKZDrive",
		Size = UDim2.fromOffset(480, 400),
		Transparent = true,
		Theme = "Dark",
		SideBarWidth = 120,
		ToggleUIKeybind = "RightControl",
	})
	local Main = Window:Tab({ Title = "Drive", Icon = "gauge" })
	local Settings = Window:Tab({ Title = "Settings", Icon = "settings" })

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
		Value = { Min = 0, Max = 1000, Default = 220 },
		Callback = function(v) CONFIG.Speed = v end,
	})
	Main:Dropdown({
		Title = "Mode",
		Values = { "FORWARD", "ZOMBIE" },
		Value = "ZOMBIE",
		Callback = function(v) CONFIG.Mode = v end,
	})
	Main:Slider({
		Title = "Entity Radius",
		Value = { Min = 20, Max = 600, Default = 400 },
		Callback = function(v) CONFIG.NpcRadius = v end,
	})
	Main:Toggle({
		Title = "Noclip",
		Value = true,
		Callback = function(v) CONFIG.Noclip = v end,
	})
	Main:Toggle({
		Title = "Debug",
		Value = true,
		Callback = function(v) CONFIG.DebugNPC = v end,
	})

	Settings:Dropdown({
		Title = "Toggle UI Key",
		Values = { "RightControl", "LeftControl", "RightShift", "LeftShift", "P", "K", "F8", "F9" },
		Value = "RightControl",
		Callback = function(name)
			CONFIG.ToggleKey = KEY_MAP[name] or Enum.KeyCode.RightControl
		end,
	})
else
	warn("[BKZ] WindUI failed")
end

print("[BKZ] 打所有实体 | 跳过玩家/所有载具 | X[-55,55]不锁")