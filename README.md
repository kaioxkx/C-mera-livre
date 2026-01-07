local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local cam = workspace.CurrentCamera

if player.PlayerGui:FindFirstChild("FreeCamGui") then
	player.PlayerGui.FreeCamGui:Destroy()
end

local isMobile = UIS.TouchEnabled and not UIS.KeyboardEnabled
local freecam = false
local speed = 1
local vel = Vector3.zero
local rotVel = Vector2.zero
local yaw, pitch = 0,0
local lastCF
local humanoid
local moveDir = Vector3.zero

-- Área de arraste da câmera
local allowRotate = false
local rotateArea = 0.40

-- GUI -------------------------------------------------
local gui = Instance.new("ScreenGui", player.PlayerGui)
gui.Name = "FreeCamGui"
gui.ResetOnSpawn = false

local btn = Instance.new("TextButton", gui)
btn.Size = UDim2.fromOffset(60,60)
btn.Position = UDim2.fromScale(0.03,0.6)
btn.BackgroundColor3 = Color3.new(0,0,0)
btn.BackgroundTransparency = 0.5
btn.Text = "CÂMERA\nLIVRE\nOFF"
btn.TextScaled = true
btn.Font = Enum.Font.FredokaOne
btn.TextColor3 = Color3.new(1,1,1)
btn.TextStrokeColor3 = Color3.new(0,0,0)
btn.TextStrokeTransparency = 0
btn.Active = true
btn.Draggable = true
Instance.new("UICorner", btn).CornerRadius = UDim.new(1,0)

-- MOBILE KEYBOARD -----------------------------------
local mobileKeys = Instance.new("Frame", gui)
mobileKeys.Visible = false
mobileKeys.Size = UDim2.fromOffset(180,180)
mobileKeys.Position = UDim2.fromScale(0.03,0.38)
mobileKeys.BackgroundTransparency = 1

local function createKey(text,pos)
	local frame = Instance.new("Frame", mobileKeys)
	frame.Size = UDim2.fromOffset(55,55)
	frame.Position = pos
	frame.BackgroundColor3 = Color3.new(0,0,0)
	frame.BackgroundTransparency = 0.15
	frame.BorderSizePixel = 0
	Instance.new("UICorner", frame).CornerRadius = UDim.new(0.1,0)

	local stroke = Instance.new("UIStroke", frame)
	stroke.Thickness = 2

	-- rainbow border
	task.spawn(function()
		local h=0
		while frame.Parent do
			h = (h+1)%360
			stroke.Color = Color3.fromHSV(h/360,1,1)
			task.wait(0.03)
		end
	end)

	local txt = Instance.new("TextButton", frame)
	txt.Size = UDim2.fromScale(1,1)
	txt.BackgroundTransparency = 1
	txt.Text = text
	txt.Font = Enum.Font.FredokaOne
	txt.TextScaled = true
	txt.TextColor3 = Color3.new(1,1,1)
	txt.TextStrokeColor3 = Color3.new(0,0,0)
	txt.TextStrokeTransparency = 0

	return txt
end

local keyW = createKey("W", UDim2.fromOffset(60,0))
local keyA = createKey("A", UDim2.fromOffset(0,60))
local keyS = createKey("S", UDim2.fromOffset(60,60))
local keyD = createKey("D", UDim2.fromOffset(120,60))

local function bindKey(btn,vec)
	btn.MouseButton1Down:Connect(function() moveDir += vec end)
	btn.MouseButton1Up:Connect(function() moveDir -= vec end)
end

bindKey(keyW, Vector3.new(0,0,-1))
bindKey(keyS, Vector3.new(0,0,1))
bindKey(keyA, Vector3.new(-1,0,0))
bindKey(keyD, Vector3.new(1,0,0))

-- VELOCIDADE ----------------------------------------
local title = Instance.new("TextLabel", gui)
title.Size = UDim2.fromOffset(140,30)
title.Position = btn.Position + UDim2.fromOffset(-35,70)
title.Text = "VELOCIDADE"
title.Visible = false
title.BackgroundTransparency = 1
title.Font = Enum.Font.FredokaOne
title.TextColor3 = Color3.new(1,1,1)
title.TextStrokeColor3 = Color3.new(0,0,0)
title.TextStrokeTransparency = 0
title.TextScaled = true

local box = Instance.new("TextBox", gui)
box.Size = UDim2.fromOffset(80,35)
box.Position = title.Position + UDim2.fromOffset(30,35)
box.Text = "1"
box.Visible = false
box.BackgroundColor3 = Color3.fromRGB(30,30,30)
box.Font = Enum.Font.FredokaOne
box.TextColor3 = Color3.new(1,1,1)
box.TextStrokeColor3 = Color3.new(0,0,0)
box.TextStrokeTransparency = 0
box.TextScaled = true

box.FocusLost:Connect(function()
	local v = tonumber(box.Text)
	if v then speed = math.clamp(v,0.1,10) end
	box.Text = tostring(speed)
end)

-- RESET ----------------------------------------
local function resetCam()
	freecam = false
	btn.Text = "CÂMERA\nLIVRE\nOFF"
	cam.CameraType = Enum.CameraType.Custom
	mobileKeys.Visible = false
	moveDir = Vector3.zero
	rotVel = Vector2.zero
end

player.CharacterAdded:Connect(function(char)
	resetCam()
	humanoid = char:WaitForChild("Humanoid")
	humanoid.Died:Connect(resetCam)
end)

btn.MouseButton1Click:Connect(function()
	freecam = not freecam
	btn.Text = freecam and "CÂMERA\nLIVRE\nON" or "CÂMERA\nLIVRE\nOFF"
	mobileKeys.Visible = freecam and isMobile
	title.Visible = freecam
	box.Visible = freecam

	if freecam then
		lastCF = cam.CFrame
		local look = lastCF.LookVector
		yaw = math.atan2(-look.X, -look.Z)
		pitch = math.asin(look.Y)
		cam.CameraType = Enum.CameraType.Scriptable
		if humanoid then
			humanoid.WalkSpeed = 0
			humanoid.JumpPower = 0
		end
	else
		resetCam()
	end
end)

-- ROTACAO TOUCH -------------------------------
UIS.TouchStarted:Connect(function(t)
	if t.Position.X > cam.ViewportSize.X * rotateArea then
		allowRotate = true
	end
end)

UIS.TouchEnded:Connect(function()
	allowRotate = false
end)

-- FREECAM ---------------------------------------
RunService:BindToRenderStep("FreeCamCinema", 301, function(dt)
	if not freecam then return end

	-- ROTACAO
	if allowRotate or not isMobile then
		local delta = UIS:GetMouseDelta()
		-- aumenta a velocidade e mantém inércia realista
		rotVel = rotVel:Lerp(Vector2.new(-delta.Y, -delta.X) * 0.6, 0.25)
	else
		-- desacelera suavemente até zero, tipo efeito cinema
		rotVel = rotVel:Lerp(Vector2.zero, 0.1)
	end

	pitch = math.clamp(pitch + math.rad(rotVel.X), math.rad(-80), math.rad(80))
	yaw += math.rad(rotVel.Y)

	local cf = CFrame.new(cam.CFrame.Position) * CFrame.Angles(0,yaw,0) * CFrame.Angles(pitch,0,0)

	-- MOVIMENTO
	local inputDir = moveDir
	if not isMobile then
		inputDir = Vector3.new(
			(UIS:IsKeyDown(Enum.KeyCode.D) and 1 or 0) - (UIS:IsKeyDown(Enum.KeyCode.A) and 1 or 0),
			0,
			(UIS:IsKeyDown(Enum.KeyCode.S) and 1 or 0) - (UIS:IsKeyDown(Enum.KeyCode.W) and 1 or 0)
		)
	end

	local move = (cf.RightVector * inputDir.X + cf.LookVector * -inputDir.Z)
	vel = vel:Lerp(move * speed * dt * 60, 0.18)
	vel *= 0.9

	cam.CFrame = cf + vel
end)
-- =============================
-- RESTAURA WALK/JUMP APÓS RESPAWN
-- =============================

local defaultWalkSpeed = 16 -- valor padrão do Roblox
local defaultJumpPower = 50 -- valor padrão do Roblox

-- Função para restaurar humanoide
local function restoreHumanoid(char)
	local hum = char:FindFirstChildOfClass("Humanoid")
	if hum then
		-- Se não estiver em FreeCam, restaura normalmente
		if not freecam then
			hum.WalkSpeed = defaultWalkSpeed
			hum.JumpPower = defaultJumpPower
		end

		-- Quando morrer, garante reset
		hum.Died:Connect(function()
			resetCam()
		end)
	end
end

-- Quando o personagem spawna
player.CharacterAdded:Connect(function(char)
	restoreHumanoid(char)
end)
