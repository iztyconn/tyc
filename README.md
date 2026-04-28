--[[
    AIMBOT + HITBOX
    - Aimbot só funciona com arma na mão ou mirando
    - Mira apenas em alvos visíveis (anti-parede)
    - Hitbox expansível (5-20)
    UI moderna, arrastável e minimizável
    Atalhos: E (aimbot) | H (hitbox) | RightAlt (minimizar)
]]

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- ===== CONFIGURAÇÕES =====
local AIMBOT_ENABLED = false
local HITBOX_ENABLED = false
local HITBOX_SIZE = 10
local AIMBOT_FOV = 120
local WEAPON_REQUIRED = true  -- só mira se tiver tool ou botão direito
local CHECK_WALL = true       -- verifica obstáculos

-- ===== VARIÁVEIS DE UI E DRAG =====
local isMinimized = false
local draggingUI = false
local dragStart = Vector2.new()
local startPos = UDim2.new()
local draggingSlider = false

local hitboxConnection = nil
local isAiming = false  -- botão direito pressionado

-- ===== FUNÇÕES AUXILIARES =====
local function hasWeapon()
    local char = LocalPlayer.Character
    return char and char:FindFirstChildOfClass("Tool") ~= nil
end

local function canAimbot()
    return AIMBOT_ENABLED and (not WEAPON_REQUIRED or hasWeapon() or isAiming)
end

local function isVisible(part)
    local origin = Camera.CFrame.Position
    local dir = (part.Position - origin).Unit
    local ray = RaycastParams.new()
    ray.FilterType = Enum.RaycastFilterType.Blacklist
    ray.FilterDescendantsInstances = {LocalPlayer.Character, Camera}
    local result = Workspace:Raycast(origin, dir * (origin - part.Position).Magnitude, ray)
    return not result or result.Instance:IsDescendantOf(part.Parent)
end

local function getBestTarget()
    local best, bestAngle = nil, AIMBOT_FOV
    local center = Camera.ViewportSize / 2
    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character then
            local head = plr.Character:FindFirstChild("Head")
            if head then
                local vec, onScreen = Camera:WorldToViewportPoint(head.Position)
                if onScreen then
                    local angle = (Vector2.new(vec.X, vec.Y) - center).Magnitude
                    if angle < bestAngle and (not CHECK_WALL or isVisible(head)) then
                        best, bestAngle = head, angle
                    end
                end
            end
        end
    end
    return best
end

-- ===== HITBOX =====
local function updateHitboxes()
    if not HITBOX_ENABLED then return end
    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character then
            local head = plr.Character:FindFirstChild("Head")
            if head then
                head.Size = Vector3.new(HITBOX_SIZE, HITBOX_SIZE, HITBOX_SIZE)
                head.Transparency = 0.65
                head.CanCollide = false
                head.Massless = true
                head.Color = Color3.fromRGB(255, 70, 110)
            end
        end
    end
end

local function resetHitboxes()
    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character then
            local head = plr.Character:FindFirstChild("Head")
            if head then
                head.Size = Vector3.new(2, 1, 1)
                head.Transparency = 1
                head.CanCollide = true
                head.Massless = false
            end
        end
    end
end

local function toggleHitbox()
    HITBOX_ENABLED = not HITBOX_ENABLED
    if HITBOX_ENABLED then
        if not hitboxConnection then
            hitboxConnection = RunService.RenderStepped:Connect(updateHitboxes)
        end
        updateHitboxes()
    else
        if hitboxConnection then
            hitboxConnection:Disconnect()
            hitboxConnection = nil
        end
        resetHitboxes()
    end
end

-- ===== AIMBOT LOOP =====
RunService.RenderStepped:Connect(function()
    if canAimbot() then
        local target = getBestTarget()
        if target then
            Camera.CFrame = CFrame.new(Camera.CFrame.Position, target.Position)
        end
    end
end)

-- ===== DETECÇÃO DE MIRA (BOTÃO DIREITO) =====
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = true
    end
end)
UserInputService.InputEnded:Connect(function(input, gp)
    if gp then return end
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = false
    end
end)

-- ===== CRIAÇÃO DA UI =====
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AimbotGUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = PlayerGui

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 240, 0, 280)
MainFrame.Position = UDim2.new(0.5, -120, 0.5, -140)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
MainFrame.BackgroundTransparency = 0.1
MainFrame.BorderSizePixel = 0
MainFrame.ClipsDescendants = true
MainFrame.Parent = ScreenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 12)
UICorner.Parent = MainFrame

local UIStroke = Instance.new("UIStroke")
UIStroke.Thickness = 1.2
UIStroke.Color = Color3.fromRGB(255, 80, 120)
UIStroke.Transparency = 0.6
UIStroke.Parent = MainFrame

-- Header (arrastável)
local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 45)
Header.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
Header.BackgroundTransparency = 0.3
Header.BorderSizePixel = 0
Header.Parent = MainFrame
Instance.new("UICorner", Header).CornerRadius = UDim.new(0, 12)

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -70, 1, 0)
Title.Position = UDim2.new(0, 10, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "⚡ AIMBOT + HITBOX"
Title.TextColor3 = Color3.fromRGB(255, 100, 150)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 12
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Header

-- Botões do header
local function makeHeaderBtn(text, xOffset, color)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 25, 0, 25)
    btn.Position = UDim2.new(1, xOffset, 0.5, -12.5)
    btn.BackgroundColor3 = color
    btn.Text = text
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = text == "✕" and 14 or 18
    btn.BorderSizePixel = 0
    btn.Parent = Header
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    return btn
end

local MinimizeBtn = makeHeaderBtn("−", -60, Color3.fromRGB(50, 50, 60))
local CloseBtn = makeHeaderBtn("✕", -30, Color3.fromRGB(200, 50, 50))

-- Conteúdo principal
local Content = Instance.new("Frame")
Content.Size = UDim2.new(1, 0, 1, -45)
Content.Position = UDim2.new(0, 0, 0, 45)
Content.BackgroundTransparency = 1
Content.Parent = MainFrame

-- Status boxes
local function makeStatusBox(y, label)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 200, 0, 36)
    frame.Position = UDim2.new(0.5, -100, 0, y)
    frame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
    frame.BackgroundTransparency = 0.5
    frame.BorderSizePixel = 0
    frame.Parent = Content
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)
    
    local led = Instance.new("Frame")
    led.Size = UDim2.new(0, 10, 0, 10)
    led.Position = UDim2.new(0, 12, 0.5, -5)
    led.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
    led.BorderSizePixel = 0
    led.Parent = frame
    Instance.new("UICorner", led).CornerRadius = UDim.new(1, 0)
    
    local text = Instance.new("TextLabel")
    text.Size = UDim2.new(1, -30, 1, 0)
    text.Position = UDim2.new(0, 28, 0, 0)
    text.BackgroundTransparency = 1
    text.Text = label .. ": OFF"
    text.TextColor3 = Color3.fromRGB(180, 180, 180)
    text.Font = Enum.Font.GothamSemibold
    text.TextSize = 11
    text.TextXAlignment = Enum.TextXAlignment.Left
    text.Parent = frame
    
    return {frame = frame, led = led, text = text}
end

local aimbotBox = makeStatusBox(10, "AIMBOT")
local hitboxBox = makeStatusBox(55, "HITBOX")

-- Slider de tamanho da hitbox
local SliderLabel = Instance.new("TextLabel")
SliderLabel.Size = UDim2.new(1, -20, 0, 20)
SliderLabel.Position = UDim2.new(0, 10, 0, 105)
SliderLabel.BackgroundTransparency = 1
SliderLabel.Text = "TAMANHO HITBOX:"
SliderLabel.TextColor3 = Color3.fromRGB(160, 160, 180)
SliderLabel.Font = Enum.Font.GothamBold
SliderLabel.TextSize = 11
SliderLabel.TextXAlignment = Enum.TextXAlignment.Left
SliderLabel.Parent = Content

local SizeValue = Instance.new("TextLabel")
SizeValue.Size = UDim2.new(0, 40, 0, 20)
SizeValue.Position = UDim2.new(1, -50, 0, 105)
SizeValue.BackgroundTransparency = 1
SizeValue.Text = "10"
SizeValue.TextColor3 = Color3.fromRGB(255, 100, 150)
SizeValue.Font = Enum.Font.GothamBold
SizeValue.TextSize = 13
SizeValue.TextXAlignment = Enum.TextXAlignment.Right
SizeValue.Parent = Content

local SliderBg = Instance.new("Frame")
SliderBg.Size = UDim2.new(1, -20, 0, 3)
SliderBg.Position = UDim2.new(0, 10, 0, 128)
SliderBg.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
SliderBg.BorderSizePixel = 0
SliderBg.Parent = Content
Instance.new("UICorner", SliderBg).CornerRadius = UDim.new(1, 0)

local SliderFill = Instance.new("Frame")
SliderFill.Size = UDim2.new(0.33, 0, 1, 0)
SliderFill.BackgroundColor3 = Color3.fromRGB(255, 80, 120)
SliderFill.BorderSizePixel = 0
SliderFill.Parent = SliderBg
Instance.new("UICorner", SliderFill).CornerRadius = UDim.new(1, 0)

local SliderBtn = Instance.new("TextButton")
SliderBtn.Size = UDim2.new(0, 14, 0, 14)
SliderBtn.Position = UDim2.new(0.33, -7, 0, -5.5)
SliderBtn.BackgroundColor3 = Color3.fromRGB(255, 100, 150)
SliderBtn.Text = ""
SliderBtn.BorderSizePixel = 0
SliderBtn.Parent = SliderBg
Instance.new("UICorner", SliderBtn).CornerRadius = UDim.new(1, 0)

-- Labels min/max
local minLbl = Instance.new("TextLabel")
minLbl.Size = UDim2.new(0, 15, 0, 15)
minLbl.Position = UDim2.new(0, 10, 0, 135)
minLbl.BackgroundTransparency = 1
minLbl.Text = "5"
minLbl.TextColor3 = Color3.fromRGB(100, 100, 120)
minLbl.Font = Enum.Font.Gotham
minLbl.TextSize = 9
minLbl.Parent = Content

local maxLbl = Instance.new("TextLabel")
maxLbl.Size = UDim2.new(0, 20, 0, 15)
maxLbl.Position = UDim2.new(1, -30, 0, 135)
maxLbl.BackgroundTransparency = 1
maxLbl.Text = "20"
maxLbl.TextColor3 = Color3.fromRGB(100, 100, 120)
maxLbl.Font = Enum.Font.Gotham
maxLbl.TextSize = 9
maxLbl.TextXAlignment = Enum.TextXAlignment.Right
maxLbl.Parent = Content

-- Botões de toggle
local function makeToggleBtn(text, xPos, yPos, bgColor)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 85, 0, 30)
    btn.Position = UDim2.new(xPos, 0, yPos, 0)
    btn.BackgroundColor3 = bgColor
    btn.Text = text
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 11
    btn.BorderSizePixel = 0
    btn.Parent = Content
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 8)
    return btn
end

local aimbotBtn = makeToggleBtn("AIMBOT", 0.15, 1, Color3.fromRGB(255, 60, 90))
aimbotBtn.Position = UDim2.new(0, 15, 1, -45)
local hitboxBtn = makeToggleBtn("HITBOX", 1, 1, Color3.fromRGB(255, 60, 90))
hitboxBtn.Position = UDim2.new(1, -100, 1, -45)

-- ===== FUNÇÕES DE UI =====
local function updateAimbotUI()
    local state = AIMBOT_ENABLED
    aimbotBox.led.BackgroundColor3 = state and Color3.fromRGB(80, 255, 80) or Color3.fromRGB(255, 50, 50)
    aimbotBox.text.Text = "AIMBOT: " .. (state and "ON" or "OFF")
    aimbotBtn.BackgroundColor3 = state and Color3.fromRGB(80, 180, 80) or Color3.fromRGB(255, 60, 90)
end

local function updateHitboxUI()
    local state = HITBOX_ENABLED
    hitboxBox.led.BackgroundColor3 = state and Color3.fromRGB(80, 255, 80) or Color3.fromRGB(255, 50, 50)
    hitboxBox.text.Text = "HITBOX: " .. (state and "ON" or "OFF")
    hitboxBtn.BackgroundColor3 = state and Color3.fromRGB(80, 180, 80) or Color3.fromRGB(255, 60, 90)
end

local function updateSlider()
    local p = (HITBOX_SIZE - 5) / 15
    SliderFill.Size = UDim2.new(p, 0, 1, 0)
    SliderBtn.Position = UDim2.new(p, -7, 0, -5.5)
    SizeValue.Text = tostring(HITBOX_SIZE)
end

-- ===== EVENTOS =====
aimbotBtn.MouseButton1Click:Connect(function()
    AIMBOT_ENABLED = not AIMBOT_ENABLED
    updateAimbotUI()
end)

hitboxBtn.MouseButton1Click:Connect(function()
    toggleHitbox()
    updateHitboxUI()
end)

CloseBtn.MouseButton1Click:Connect(function()
    if hitboxConnection then hitboxConnection:Disconnect() end
    resetHitboxes()
    ScreenGui:Destroy()
end)

MinimizeBtn.MouseButton1Click:Connect(function()
    isMinimized = not isMinimized
    local targetSize = isMinimized and UDim2.new(0, 240, 0, 45) or UDim2.new(0, 240, 0, 280)
    TweenService:Create(MainFrame, TweenInfo.new(0.3), {Size = targetSize}):Play()
    Content.Visible = not isMinimized
    MinimizeBtn.Text = isMinimized and "+" or "−"
end)

-- Slider drag
SliderBtn.MouseButton1Down:Connect(function()
    draggingSlider = true
end)

UserInputService.InputChanged:Connect(function(input)
    if draggingSlider and input.UserInputType == Enum.UserInputType.MouseMovement then
        local w = SliderBg.AbsoluteSize.X
        if w == 0 then return end
        local percent = math.clamp((input.Position.X - SliderBg.AbsolutePosition.X) / w, 0, 1)
        HITBOX_SIZE = math.floor(5 + percent * 15)
        HITBOX_SIZE = math.clamp(HITBOX_SIZE, 5, 20)
        updateSlider()
        if HITBOX_ENABLED then updateHitboxes() end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingSlider = false
    end
end)

-- Drag da janela
Header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingUI = true
        dragStart = input.Position
        startPos = MainFrame.Position
    end
end)

Header.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingUI = false
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if draggingUI and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

-- Atalhos de teclado
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == Enum.KeyCode.E then
        AIMBOT_ENABLED = not AIMBOT_ENABLED
        updateAimbotUI()
    elseif input.KeyCode == Enum.KeyCode.H then
        toggleHitbox()
        updateHitboxUI()
    elseif input.KeyCode == Enum.KeyCode.RightAlt then
        MinimizeBtn.MouseButton1Click:Fire()
    end
end)

-- ===== INICIALIZAÇÃO =====
updateAimbotUI()
updateHitboxUI()
updateSlider()

-- Animação de entrada
MainFrame.BackgroundTransparency = 1
MainFrame.Size = UDim2.new(0, 200, 0, 0)
task.wait(0.05)
TweenService:Create(MainFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back), {
    Size = UDim2.new(0, 240, 0, 280),
    BackgroundTransparency = 0.1
}):Play()

-- Notificação
local notify = Instance.new("Frame")
notify.Size = UDim2.new(0, 240, 0, 35)
notify.Position = UDim2.new(0.5, -120, 1, -45)
notify.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
notify.BackgroundTransparency = 0.15
notify.Parent = ScreenGui
Instance.new("UICorner", notify).CornerRadius = UDim.new(0, 8)

local notifText = Instance.new("TextLabel")
notifText.Size = UDim2.new(1, 0, 1, 0)
notifText.BackgroundTransparency = 1
notifText.Text = "✓ AIMBOT + HITBOX | E (aimbot) | H (hitbox)"
notifText.TextColor3 = Color3.fromRGB(255, 255, 255)
notifText.Font = Enum.Font.GothamSemibold
notifText.TextSize = 10
notifText.Parent = notify

task.wait(3)
notify:Destroy()

-- Detectar novos personagens para hitbox
Players.PlayerAdded:Connect(function(p)
    p.CharacterAdded:Connect(function()
        task.wait(0.5)
        if HITBOX_ENABLED then updateHitboxes() end
    end)
end)
