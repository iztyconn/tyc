local WindUI = loadstring(game:HttpGet("https://raw.githubusercontent.com/Footagesus/WindUI/main/dist/main.lua"))()

local AIMBOT_ENABLED = false
local FOV_ENABLED = true
local FOV_RADIUS = 120
local HITBOX_ENABLED = false
local HITBOX_SIZE = 10
local WEAPON_REQUIRED = true
local CHECK_WALL = true
local AIMBOT_SMOOTH = 0.35

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

local isAiming = false
local hitboxConnection = nil
local aimbotConnection = nil
local fovCircle = nil

local function setupFOVCircle()
    if not Drawing or not Drawing.new then return false end
    local success, circle = pcall(function() return Drawing.new("Circle") end)
    if success and circle then
        fovCircle = circle
        fovCircle.Visible = FOV_ENABLED
        fovCircle.Radius = FOV_RADIUS
        fovCircle.Thickness = 2
        fovCircle.Color = Color3.fromRGB(255, 100, 150)
        fovCircle.Filled = false
        fovCircle.NumSides = 64
        fovCircle.Transparency = 0.7
        return true
    end
    return false
end
setupFOVCircle()

local function hasWeapon()
    local char = LocalPlayer.Character
    return char and char:FindFirstChildOfClass("Tool") ~= nil
end

local function isVisible(part)
    local origin = Camera.CFrame.Position
    local dir = (part.Position - origin).Unit
    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Blacklist
    rayParams.FilterDescendantsInstances = {LocalPlayer.Character, Camera}
    local result = Workspace:Raycast(origin, dir * (origin - part.Position).Magnitude, rayParams)
    return not result or result.Instance:IsDescendantOf(part.Parent)
end

local function getBestTarget()
    if not FOV_ENABLED then return nil end
    local best, bestAngle = nil, FOV_RADIUS
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

local function doAimbot(target)
    if not target then return end
    if AIMBOT_SMOOTH > 0 then
        local current = Camera.CFrame
        local targetCF = CFrame.new(current.Position, target.Position)
        local newCF = current:Lerp(targetCF, AIMBOT_SMOOTH)
        Camera.CFrame = newCF
    else
        Camera.CFrame = CFrame.new(Camera.CFrame.Position, target.Position)
    end
end

local function startAimbotLoop()
    if aimbotConnection then aimbotConnection:Disconnect() end
    aimbotConnection = RunService.RenderStepped:Connect(function()
        if not AIMBOT_ENABLED then return end
        if WEAPON_REQUIRED and not (hasWeapon() or isAiming) then return end
        local target = getBestTarget()
        if target then
            doAimbot(target)
        end
    end)
end

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

RunService.RenderStepped:Connect(function()
    if fovCircle then
        fovCircle.Visible = FOV_ENABLED
        if FOV_ENABLED and Camera and Camera.ViewportSize then
            fovCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
            fovCircle.Radius = FOV_RADIUS
        end
    end
end)

startAimbotLoop()

local userId = LocalPlayer.UserId
local avatarThumbnail = "https://www.roblox.com/headshot-thumbnail/image?userId=" .. userId .. "&width=150&height=150&format=png"

local Window = WindUI:CreateWindow({
    Title = "My Super Hub",
    Icon = "boxes",
    Author = "by Tycoon",
    Folder = "MySuperHub_Aimbot",
    Size = UDim2.fromOffset(580, 460),
    MinSize = Vector2.new(560, 350),
    MaxSize = Vector2.new(850, 560),
    ToggleKey = Enum.KeyCode.LeftShift,
    Transparent = true,
    Theme = "Dark",
    Resizable = true,
    SideBarWidth = 200,
    BackgroundImageTransparency = 0.42,
    HideSearchBar = true,
    ScrollBarEnabled = false,
    User = {
        Enabled = true,
        Anonymous = false,
        Image = avatarThumbnail,
        Callback = function() end,
    },
})

Window:Tag({
    Title = "v2.0.0",
    Icon = "aura",
    Color = Color3.fromHex("#30ff6a"),
    Radius = 20,
})

task.wait(0.2)

local AimbotTab = Window:Tab({
    Title = "Aimbot",
    Icon = "crosshair",
})

local HitboxTab = Window:Tab({
    Title = "Hitbox",
    Icon = "user",
})

AimbotTab:Toggle({ Title = "Aimbot", Value = AIMBOT_ENABLED, Callback = function(s) AIMBOT_ENABLED = s end })
AimbotTab:Toggle({ Title = "Anti-parede", Value = CHECK_WALL, Callback = function(s) CHECK_WALL = s end })
AimbotTab:Toggle({ Title = "Exigir arma/mira", Value = WEAPON_REQUIRED, Callback = function(s) WEAPON_REQUIRED = s end })

AimbotTab:Divider()

AimbotTab:Slider({ Title = "Raio do FOV", Step = 5, Value = { Min = 40, Max = 400, Default = FOV_RADIUS }, Callback = function(v) FOV_RADIUS = v end })
AimbotTab:Slider({ Title = "Suavização", Step = 0.05, Value = { Min = 0, Max = 1, Default = AIMBOT_SMOOTH }, Callback = function(v) AIMBOT_SMOOTH = v end })

AimbotTab:Divider()

AimbotTab:Toggle({ Title = "Mostrar círculo FOV", Value = FOV_ENABLED, Callback = function(s) FOV_ENABLED = s end })

HitboxTab:Toggle({ Title = "Hitbox", Value = HITBOX_ENABLED, Callback = function(s) toggleHitbox() end })
HitboxTab:Divider()
HitboxTab:Slider({ Title = "Tamanho da hitbox", Step = 1, Value = { Min = 5, Max = 20, Default = HITBOX_SIZE }, Callback = function(v)
    HITBOX_SIZE = v
    if HITBOX_ENABLED then updateHitboxes() end
end })

local OpenButton = Instance.new("TextButton")
OpenButton.Size = UDim2.new(0, 120, 0, 40)
OpenButton.Position = UDim2.new(0.8, 0, 0.85, 0)
OpenButton.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
OpenButton.Text = "Open UI"
OpenButton.TextColor3 = Color3.fromRGB(255, 255, 255)
OpenButton.Font = Enum.Font.GothamBold
OpenButton.TextSize = 14
OpenButton.BackgroundTransparency = 0.2
OpenButton.BorderSizePixel = 0
OpenButton.Visible = false
OpenButton.Parent = PlayerGui

local OpenButtonCorner = Instance.new("UICorner")
OpenButtonCorner.CornerRadius = UDim.new(0, 12)
OpenButtonCorner.Parent = OpenButton

local OpenButtonStroke = Instance.new("UIStroke")
OpenButtonStroke.Thickness = 1.2
OpenButtonStroke.Color = Color3.fromRGB(255, 100, 150)
OpenButtonStroke.Transparency = 0.5
OpenButtonStroke.Parent = OpenButton

local draggingOpen = false
local dragStartOpen = Vector2.new()
local startPosOpen = UDim2.new()

OpenButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingOpen = true
        dragStartOpen = input.Position
        startPosOpen = OpenButton.Position
    end
end)

OpenButton.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingOpen = false
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if draggingOpen and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStartOpen
        OpenButton.Position = UDim2.new(startPosOpen.X.Scale, startPosOpen.X.Offset + delta.X, startPosOpen.Y.Scale, startPosOpen.Y.Offset + delta.Y)
    end
end)

local function minimizeWindow()
    Window.Frame.Visible = false
    OpenButton.Visible = true
end

local function restoreWindow()
    Window.Frame.Visible = true
    OpenButton.Visible = false
end

OpenButton.MouseButton1Click:Connect(restoreWindow)

task.wait(0.2)
local topbar = Window.Frame:FindFirstChild("Topbar")
if topbar then
    for _, btn in pairs(topbar:GetChildren()) do
        if btn:IsA("TextButton") and btn.Text == "−" then
            btn.MouseButton1Click:Connect(minimizeWindow)
            break
        end
    end
end

local closeButton = topbar and topbar:FindFirstChild("Close")
if closeButton and closeButton:IsA("TextButton") then
    closeButton.MouseButton1Click:Connect(function()
        if hitboxConnection then hitboxConnection:Disconnect() end
        resetHitboxes()
        if fovCircle then fovCircle:Destroy() end
        OpenButton:Destroy()
        Window:Destroy()
    end)
end
