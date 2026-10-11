
--// NEON HUB | Standalone Lua UI
--// UI + Speed + Jump + Crosshair + Aim Training
--// Best used in your own Roblox experience or test place.

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
if not player then return end

local playerGui = player:WaitForChild("PlayerGui")

local old = playerGui:FindFirstChild("NeonHub")
if old then old:Destroy() end

local gui = Instance.new("ScreenGui")
gui.Name = "NeonHub"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.Parent = playerGui

local neon = Color3.fromRGB(0, 255, 220)
local dark = Color3.fromRGB(15, 18, 30)
local white = Color3.fromRGB(240, 250, 255)

local function make(class, props, parent)
    local obj = Instance.new(class)
    for k, v in pairs(props) do
        obj[k] = v
    end
    obj.Parent = parent
    return obj
end

local main = make("Frame", {
    Name = "Main",
    Size = UDim2.fromOffset(300, 340),
    Position = UDim2.new(0.5, -150, 0.5, -170),
    BackgroundColor3 = dark,
    BorderSizePixel = 0,
}, gui)

make("UICorner", {CornerRadius = UDim.new(0, 12)}, main)
make("UIStroke", {Color = neon, Thickness = 2}, main)

local header = make("TextButton", {
    Name = "Header",
    Size = UDim2.new(1, 0, 0, 44),
    BackgroundColor3 = Color3.fromRGB(20, 45, 55),
    Text = "✦  NEON HUB",
    TextColor3 = neon,
    TextSize = 19,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
}, main)
make("UICorner", {CornerRadius = UDim.new(0, 10)}, header)

local content = make("Frame", {
    Name = "Content",
    Position = UDim2.fromOffset(12, 54),
    Size = UDim2.new(1, -24, 1, -66),
    BackgroundTransparency = 1,
}, main)

make("UIListLayout", {
    Padding = UDim.new(0, 7),
    SortOrder = Enum.SortOrder.LayoutOrder,
}, content)

-- Drag window
local dragging = false
local dragStart, startPos

header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
    or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = main.Position
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and (
        input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch
    ) then
        local delta = input.Position - dragStart
        main.Position = UDim2.new(
            startPos.X.Scale, startPos.X.Offset + delta.X,
            startPos.Y.Scale, startPos.Y.Offset + delta.Y
        )
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
    or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)

local function addButton(text, callback)
    local button = make("TextButton", {
        Size = UDim2.new(1, 0, 0, 36),
        BackgroundColor3 = Color3.fromRGB(25, 34, 48),
        Text = text,
        TextColor3 = white,
        TextSize = 14,
        Font = Enum.Font.GothamSemibold,
        AutoButtonColor = false,
    }, content)

    make("UICorner", {CornerRadius = UDim.new(0, 8)}, button)
    make("UIStroke", {Color = Color3.fromRGB(50, 100, 110), Thickness = 1}, button)

    button.Activated:Connect(function()
        callback(button)
    end)
    return button
end

local function getHumanoid()
    local character = player.Character
    return character and character:FindFirstChildOfClass("Humanoid")
end

local speedOn = false
local jumpOn = false
local speedValue = 24
local jumpValue = 65

local function refreshSpeed()
    local hum = getHumanoid()
    if hum then
        hum.WalkSpeed = speedOn and speedValue or 16
    end
end

local function refreshJump()
    local hum = getHumanoid()
    if hum then
        hum.UseJumpPower = true
        hum.JumpPower = jumpOn and jumpValue or 50
    end
end

addButton("SPEED: OFF", function(button)
    speedOn = not speedOn
    button.Text = speedOn and "SPEED: ON (" .. speedValue .. ")" or "SPEED: OFF"
    button.TextColor3 = speedOn and neon or white
    refreshSpeed()
end)

addButton("SPEED + 4", function()
    speedValue = math.clamp(speedValue + 4, 16, 100)
    if speedOn then refreshSpeed() end
end)

addButton("JUMP: OFF", function(button)
    jumpOn = not jumpOn
    button.Text = jumpOn and "JUMP: ON (" .. jumpValue .. ")" or "JUMP: OFF"
    button.TextColor3 = jumpOn and neon or white
    refreshJump()
end)

addButton("JUMP + 10", function()
    jumpValue = math.clamp(jumpValue + 10, 50, 150)
    if jumpOn then refreshJump() end
end)

-- Toggle a simple screen crosshair
local crosshairOn = false
local crosshair

addButton("CROSSHAIR: OFF", function(button)
    crosshairOn = not crosshairOn
    button.Text = crosshairOn and "CROSSHAIR: ON" or "CROSSHAIR: OFF"
    button.TextColor3 = crosshairOn and neon or white

    if crosshair then
        crosshair:Destroy()
        crosshair = nil
    end

    if crosshairOn then
        crosshair = make("TextLabel", {
            Name = "TrainingCrosshair",
            AnchorPoint = Vector2.new(0.5, 0.5),
            Position = UDim2.fromScale(0.5, 0.5),
            Size = UDim2.fromOffset(36, 36),
            BackgroundTransparency = 1,
            Text = "+",
            TextColor3 = neon,
            TextStrokeTransparency = 0.25,
            TextSize = 32,
            Font = Enum.Font.GothamBold,
        }, gui)
    end
end)

-- Practice target: click the moving target to score points
local trainingOn = false
local target
local score = 0
local scoreLabel
local targetConnection

local function stopTraining()
    trainingOn = false
    if targetConnection then
        targetConnection:Disconnect()
        targetConnection = nil
    end
    if target then target:Destroy(); target = nil end
    if scoreLabel then scoreLabel:Destroy(); scoreLabel = nil end
end

addButton("AIM TRAINING: START", function(button)
    if trainingOn then
        stopTraining()
        button.Text = "AIM TRAINING: START"
        button.TextColor3 = white
        return
    end

    trainingOn = true
    score = 0

    scoreLabel = make("TextLabel", {
        Position = UDim2.new(0.5, -100, 0, 55),
        Size = UDim2.fromOffset(200, 32),
        BackgroundColor3 = dark,
        BackgroundTransparency = 0.15,
        TextColor3 = neon,
        TextSize = 18,
        Font = Enum.Font.GothamBold,
        Text = "SCORE: 0",
    }, gui)
    make("UICorner", {CornerRadius = UDim.new(0, 8)}, scoreLabel)

    target = make("TextButton", {
        Name = "PracticeTarget",
        Size = UDim2.fromOffset(48, 48),
        Position = UDim2.new(0.7, 0, 0.5, 0),
        BackgroundColor3 = Color3.fromRGB(255, 55, 120),
        Text = "●",
        TextColor3 = white,
        TextSize = 28,
        AutoButtonColor = true,
    }, gui)
    make("UICorner", {CornerRadius = UDim.new(1, 0)}, target)

    target.Activated:Connect(function()
        if not trainingOn then return end
        score = score + 1
        if scoreLabel then
            scoreLabel.Text = "SCORE: " .. score
        end

        target.Position = UDim2.new(
            math.random(15, 80) / 100, 0,
            math.random(20, 75) / 100, 0
        )
    end)

    targetConnection = RunService.RenderStepped:Connect(function()
        if trainingOn and target and target.Parent then
            target.Rotation = (target.Rotation + 1) % 360
        end
    end)

    button.Text = "AIM TRAINING: STOP"
    button.TextColor3 = neon
end)

addButton("CLOSE MENU", function()
    gui:Destroy()
end)

player.CharacterAdded:Connect(function()
    task.wait(1)
    refreshSpeed()
    refreshJump()
end)
