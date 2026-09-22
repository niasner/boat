-- ========================================================
-- Blox Fruits TAKO BOAT
-- ========================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- 控制參數
local BoatFlyEnabled = false
local AutoSea6Enabled = false
local NoclipEnabled = true             -- 預設開啟穿牆
local BoatSpeed = 350                  -- 預設船速
local TargetHeight = 150               -- 固定飛行高度
local LockedDirection = nil            -- 鏡頭鎖定方向

local UpPressed = false
local DownPressed = false

-- 取得當前船隻模型並解鎖零件
local function getBoatData()
    local char = LocalPlayer.Character
    if not char then return nil, nil end
    
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum or not hum.SeatPart then return nil, nil end
    
    local seat = hum.SeatPart
    if not seat:IsA("VehicleSeat") and not seat:IsA("Seat") then return nil, nil end
    
    local boat = seat.Parent
    if boat and boat:IsA("Model") then
        return boat, seat
    end
    return nil, nil
end

-- 核心 1：物理計算前（Stepped）強制關閉船隻與角色的碰撞體 (Noclip)
RunService.Stepped:Connect(function()
    if not NoclipEnabled and not BoatFlyEnabled and not AutoSea6Enabled then return end
    
    -- 1. 船隻零件穿牆
    local boat, _ = getBoatData()
    if boat then
        for _, part in ipairs(boat:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
                part.Anchored = false
                part.AssemblyLinearVelocity = Vector3.zero
                part.AssemblyAngularVelocity = Vector3.zero
            end
        end
    end
    
    -- 2. 角色身體零件穿牆（防止人物卡牆帶動船隻）
    local char = LocalPlayer.Character
    if char then
        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
            end
        end
    end
end)

-- 核心 2：位置與航行更新（Heartbeat）
RunService.Heartbeat:Connect(function(dt)
    if not BoatFlyEnabled and not AutoSea6Enabled then return end
    
    local boat, seat = getBoatData()
    if not boat or not seat then return end
    
    local currentPos = seat.Position
    
    if AutoSea6Enabled then
        -- 鏡頭方向鎖定開航
        if not LockedDirection then
            local camLook = Camera.CFrame.LookVector
            local flatLook = Vector3.new(camLook.X, 0, camLook.Z)
            LockedDirection = flatLook.Magnitude > 0 and flatLook.Unit or Vector3.new(0, 0, -1)
        end
        
        local moveOffset = LockedDirection * (BoatSpeed * dt)
        local nextPos = Vector3.new(currentPos.X + moveOffset.X, TargetHeight, currentPos.Z + moveOffset.Z)
        
        boat:PivotTo(CFrame.new(nextPos, nextPos + LockedDirection))
        
    elseif BoatFlyEnabled then
        -- WASD 手動飛行
        local camCF = Camera.CFrame
        local moveDir = Vector3.zero
        
        if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDir = moveDir + camCF.LookVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDir = moveDir - camCF.LookVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDir = moveDir + camCF.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDir = moveDir - camCF.RightVector end
        
        if UpPressed or UserInputService:IsKeyDown(Enum.KeyCode.Space) or UserInputService:IsKeyDown(Enum.KeyCode.E) then
            moveDir = moveDir + Vector3.new(0, 1, 0)
        end
        if DownPressed or UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) or UserInputService:IsKeyDown(Enum.KeyCode.Q) then
            moveDir = moveDir - Vector3.new(0, 1, 0)
        end
        
        if moveDir.Magnitude > 0 then
            local nextPos = currentPos + (moveDir.Unit * BoatSpeed * dt)
            boat:PivotTo(CFrame.new(nextPos, nextPos + camCF.LookVector))
        end
    end
end)

-- ========================================================
-- UI 控制介面
-- ========================================================

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "XenoBoatFlyV8"
screenGui.ResetOnSpawn = false
local coreGui = game:GetService("CoreGui")
if coreGui:FindFirstChild("XenoBoatFlyV8") then coreGui.XenoBoatFlyV8:Destroy() end
screenGui.Parent = coreGui

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 220, 0, 250)
frame.Position = UDim2.new(0.05, 0, 0.25, 0)
frame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
frame.Active = true
frame.Draggable = true
frame.Parent = screenGui

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 30)
title.Text = "🚢 Xeno 船航 (穿牆無敵版)"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
title.Font = Enum.Font.SourceSansBold
title.TextSize = 13
title.Parent = frame

-- WASD 按鈕
local flyBtn = Instance.new("TextButton")
flyBtn.Size = UDim2.new(0.9, 0, 0, 28)
flyBtn.Position = UDim2.new(0.05, 0, 0.14, 0)
flyBtn.Text = "WASD 手動飛行: 關閉"
flyBtn.BackgroundColor3 = Color3.fromRGB(180, 50, 50)
flyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
flyBtn.Font = Enum.Font.SourceSansBold
flyBtn.Parent = frame

-- 鏡頭開航按鈕
local autoBtn = Instance.new("TextButton")
autoBtn.Size = UDim2.new(0.9, 0, 0, 28)
autoBtn.Position = UDim2.new(0.05, 0, 0.27, 0)
autoBtn.Text = "🌊 朝鏡頭方向開航: 關閉"
autoBtn.BackgroundColor3 = Color3.fromRGB(180, 50, 50)
autoBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
autoBtn.Font = Enum.Font.SourceSansBold
autoBtn.Parent = frame

-- 穿牆開關按鈕
local noclipBtn = Instance.new("TextButton")
noclipBtn.Size = UDim2.new(0.9, 0, 0, 28)
noclipBtn.Position = UDim2.new(0.05, 0, 0.40, 0)
noclipBtn.Text = "👻 船隻/角色穿牆: 開啟"
noclipBtn.BackgroundColor3 = Color3.fromRGB(50, 180, 80)
noclipBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
noclipBtn.Font = Enum.Font.SourceSansBold
noclipBtn.Parent = frame

noclipBtn.MouseButton1Click:Connect(function()
    NoclipEnabled = not NoclipEnabled
    noclipBtn.Text = NoclipEnabled and "👻 船隻/角色穿牆: 開啟" or "👻 船隻/角色穿牆: 關閉"
    noclipBtn.BackgroundColor3 = NoclipEnabled and Color3.fromRGB(50, 180, 80) or Color3.fromRGB(180, 50, 50)
end)

autoBtn.MouseButton1Click:Connect(function()
    AutoSea6Enabled = not AutoSea6Enabled
    if AutoSea6Enabled then
        BoatFlyEnabled = false
        local camLook = Camera.CFrame.LookVector
        local flatLook = Vector3.new(camLook.X, 0, camLook.Z)
        LockedDirection = flatLook.Magnitude > 0 and flatLook.Unit or Vector3.new(0, 0, -1)
    else
        LockedDirection = nil
    end
    
    autoBtn.Text = AutoSea6Enabled and "🌊 朝鏡頭方向開航: 開啟" or "🌊 朝鏡頭方向開航: 關閉"
    autoBtn.BackgroundColor3 = AutoSea6Enabled and Color3.fromRGB(50, 180, 80) or Color3.fromRGB(180, 50, 50)
    flyBtn.Text = "WASD 手動飛行: 關閉"
    flyBtn.BackgroundColor3 = Color3.fromRGB(180, 50, 50)
end)

flyBtn.MouseButton1Click:Connect(function()
    BoatFlyEnabled = not BoatFlyEnabled
    if BoatFlyEnabled then 
        AutoSea6Enabled = false 
        LockedDirection = nil
    end
    flyBtn.Text = BoatFlyEnabled and "WASD 手動飛行: 開啟" or "WASD 手動飛行: 關閉"
    flyBtn.BackgroundColor3 = BoatFlyEnabled and Color3.fromRGB(50, 180, 80) or Color3.fromRGB(180, 50, 50)
    autoBtn.Text = "🌊 朝鏡頭方向開航: 關閉"
    autoBtn.BackgroundColor3 = Color3.fromRGB(180, 50, 50)
end)

-- 船速控制 UI
local speedLabel = Instance.new("TextLabel")
speedLabel.Size = UDim2.new(0.5, 0, 0, 26)
speedLabel.Position = UDim2.new(0.05, 0, 0.54, 0)
speedLabel.Text = "目前船速: " .. tostring(BoatSpeed)
speedLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
speedLabel.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
speedLabel.Font = Enum.Font.SourceSans
speedLabel.TextSize = 13
speedLabel.Parent = frame

local speedSubBtn = Instance.new("TextButton")
speedSubBtn.Size = UDim2.new(0.18, 0, 0, 26)
speedSubBtn.Position = UDim2.new(0.57, 0, 0.54, 0)
speedSubBtn.Text = "-50"
speedSubBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
speedSubBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
speedSubBtn.Font = Enum.Font.SourceSansBold
speedSubBtn.Parent = frame

local speedAddBtn = Instance.new("TextButton")
speedAddBtn.Size = UDim2.new(0.18, 0, 0, 26)
speedAddBtn.Position = UDim2.new(0.77, 0, 0.54, 0)
speedAddBtn.Text = "+50"
speedAddBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
speedAddBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
speedAddBtn.Font = Enum.Font.SourceSansBold
speedAddBtn.Parent = frame

speedSubBtn.MouseButton1Click:Connect(function()
    BoatSpeed = math.max(50, BoatSpeed - 50)
    speedLabel.Text = "目前船速: " .. tostring(BoatSpeed)
end)

speedAddBtn.MouseButton1Click:Connect(function()
    BoatSpeed = BoatSpeed + 50
    speedLabel.Text = "目前船速: " .. tostring(BoatSpeed)
end)

-- 上升 / 下降按鈕
local upBtn = Instance.new("TextButton")
upBtn.Size = UDim2.new(0.42, 0, 0, 28)
upBtn.Position = UDim2.new(0.05, 0, 0.68, 0)
upBtn.Text = "▲ 上升 (Space)"
upBtn.Parent = frame
upBtn.MouseButton1Down:Connect(function() UpPressed = true end)
upBtn.MouseButton1Up:Connect(function() UpPressed = false end)

local downBtn = Instance.new("TextButton")
downBtn.Size = UDim2.new(0.42, 0, 0, 0.68, 0)
downBtn.Position = UDim2.new(0.53, 0, 0.68, 0)
downBtn.Text = "▼ 下降 (Shift)"
downBtn.Parent = frame
downBtn.MouseButton1Down:Connect(function() DownPressed = true end)
downBtn.MouseButton1Up:Connect(function() DownPressed = false end)
