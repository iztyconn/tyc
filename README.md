local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- CONFIG
local AIMBOT_ENABLED = false
local HITBOX_ENABLED = false
local HOLDING = false

local HITBOX_SIZE = 10

local hitboxConnection

-- ================= INPUT (SEGURAR BOTÃO) =================
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        HOLDING = true
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        HOLDING = false
    end
end)

-- ================= ANTI WALL =================
local function isVisible(targetPart)
    local origin = Camera.CFrame.Position
    local direction = (targetPart.Position - origin)

    local params = RaycastParams.new()
    params.FilterDescendantsInstances = {LocalPlayer.Character}
    params.FilterType = Enum.RaycastFilterType.Blacklist

    local result = Workspace:Raycast(origin, direction, params)

    if result then
        return result.Instance:IsDescendantOf(targetPart.Parent)
    end

    return true
end

-- ================= TARGET =================
local function getTarget()
    local closest = nil
    local shortest = math.huge

    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild("Head") then
            local head = plr.Character.Head

            local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
            if not onScreen then continue end

            if not isVisible(head) then continue end

            local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
            local dist = (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude

            if dist < shortest then
                shortest = dist
                closest = head
            end
        end
    end

    return closest
end

-- ================= HITBOX =================
local function toggleHitbox()
    HITBOX_ENABLED = not HITBOX_ENABLED

    if HITBOX_ENABLED then
        hitboxConnection = RunService.RenderStepped:Connect(function()
            for _, p in pairs(Players:GetPlayers()) do
                if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("Head") then
                    local h = p.Character.Head
                    h.Size = Vector3.new(HITBOX_SIZE, HITBOX_SIZE, HITBOX_SIZE)
                    h.Transparency = 0.7
                    h.CanCollide = false
                    h.Massless = true
                end
            end
        end)
    else
        if hitboxConnection then
            hitboxConnection:Disconnect()
            hitboxConnection = nil
        end
    end
end

-- ================= AIMBOT LOOP =================
RunService.RenderStepped:Connect(function()
    if not AIMBOT_ENABLED or not HOLDING then return end

    local target = getTarget()
    if target then
        local current = Camera.CFrame
        local targetCF = CFrame.new(current.Position, target.Position)

        -- SMOOTH FORTE + LEGIT
        local distance = (target.Position - current.Position).Magnitude
        local baseSmooth = 0.05
        local dynamic = math.clamp(distance / 1000, 0, 0.05)
        local smooth = baseSmooth + dynamic

        Camera.CFrame = current:Lerp(targetCF, smooth)
    end
end)

-- ================= UI TOP =================
local ScreenGui = Instance.new("ScreenGui", PlayerGui)
ScreenGui.Name = "ProAimUI"
ScreenGui.ResetOnSpawn = false

local Frame = Instance.new("Frame", ScreenGui)
Frame.Size = UDim2.new(0, 230, 0, 190)
Frame.Position = UDim2.new(0.35, 0, 0.3, 0)
Frame.BackgroundColor3 = Color3.fromRGB(25,25,35)
Frame.Active = true
Frame.Draggable = true
Frame.BorderSizePixel = 0
Instance.new("UICorner", Frame).CornerRadius = UDim.new(0, 12)

local gradient = Instance.new("UIGradient", Frame)
gradient.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(30,30,45)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(20,20,30))
}

local Title = Instance.new("TextLabel", Frame)
Title.Size = UDim2.new(1,0,0,40)
Title.BackgroundTransparency = 1
Title.Text = "🔥 AIM PANEL"
Title.TextColor3 = Color3.fromRGB(0,255,150)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 18

local function createToggle(text, posY)
    local btn = Instance.new("TextButton", Frame)
    btn.Size = UDim2.new(0,180,0,35)
    btn.Position = UDim2.new(0.1,0,0,posY)
    btn.BackgroundColor3 = Color3.fromRGB(40,40,55)
    btn.Text = text.." : OFF"
    btn.TextColor3 = Color3.new(1,1,1)
    btn.Font = Enum.Font.GothamSemibold
    btn.TextSize = 14
    btn.BorderSizePixel = 0
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0,8)
    return btn
end

local AimbotBtn = createToggle("AIMBOT", 60)
local HitboxBtn = createToggle("HITBOX", 105)

local SizeBox = Instance.new("TextBox", Frame)
SizeBox.Size = UDim2.new(0,180,0,30)
SizeBox.Position = UDim2.new(0.1,0,0,145)
SizeBox.PlaceholderText = "Hitbox Size"
SizeBox.Text = "10"
SizeBox.BackgroundColor3 = Color3.fromRGB(40,40,55)
SizeBox.TextColor3 = Color3.new(1,1,1)
SizeBox.Font = Enum.Font.Gotham
SizeBox.TextSize = 14
Instance.new("UICorner", SizeBox).CornerRadius = UDim.new(0,8)

local function animate(btn, state)
    if state then
        btn.BackgroundColor3 = Color3.fromRGB(0,170,100)
        btn.Text = btn.Text:gsub("OFF","ON")
    else
        btn.BackgroundColor3 = Color3.fromRGB(40,40,55)
        btn.Text = btn.Text:gsub("ON","OFF")
    end
end

AimbotBtn.MouseButton1Click:Connect(function()
    AIMBOT_ENABLED = not AIMBOT_ENABLED
    animate(AimbotBtn, AIMBOT_ENABLED)
end)

HitboxBtn.MouseButton1Click:Connect(function()
    HITBOX_SIZE = tonumber(SizeBox.Text) or 10
    toggleHitbox()
    animate(HitboxBtn, HITBOX_ENABLED)
end)
