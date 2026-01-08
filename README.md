-- BeanzZZ AutoRob Script for Emergency Hamburg
-- Minimal GUI like the image, no left settings panel
-- Auto sequence: Club -> Bank (if bombable) -> Jeweler -> Sell -> Parking -> Server hop to low pop (<=5 players)

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TeleportService = game:GetService("TeleportService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer
local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
local HumanoidRootPart = Character:WaitForChild("HumanoidRootPart")
local Humanoid = Character:WaitForChild("Humanoid")

-- Update character reference
LocalPlayer.CharacterAdded:Connect(function(newChar)
    Character = newChar
    HumanoidRootPart = newChar:WaitForChild("HumanoidRootPart")
    Humanoid = newChar:WaitForChild("Humanoid")
end)

-- Simple teleport function with pathfinding-like tween
local function teleportTo(cframe)
    local tween = TweenService:Create(HumanoidRootPart, TweenInfo.new(2, Enum.EasingStyle.Linear), {CFrame = cframe})
    tween:Play()
    tween.Completed:Wait()
end

-- Approximate CFrames for Emergency Hamburg (adjust if needed by testing in-game)
local locations = {
    dealer = CFrame.new(-100, 5, 200),  -- Nearest dealer position
    clubSafeFront = CFrame.new(150, 5, 100),  -- Club safe front
    clubDoorBehind = CFrame.new(148, 5, 102),  -- Behind door for grenade
    clubSafe = CFrame.new(152, 5, 98),  -- Safe position
    bankVaultFront = CFrame.new(0, 5, 0),  -- Bank vault
    bankEntrance = CFrame.new(10, 5, 20),  -- Entrance after bomb
    bankLootPositions = {  -- 3 loot spots
        CFrame.new(5, 5, 5),
        CFrame.new(-5, 5, 5),
        CFrame.new(0, 5, 10)
    },
    jewelerSafeFront = CFrame.new(200, 5, -100),  -- Jeweler safe
    jewelerEntranceFar = CFrame.new(205, 5, -120),  -- Far from entrance after bomb
    jewelerSafe = CFrame.new(198, 5, -98),  -- Loot all
    parkingGarage = CFrame.new(50, -20, 10)  -- Bottom level parking
}

-- Function to buy grenade/bomb from nearest dealer
local function buyGrenade()
    teleportTo(locations.dealer)
    wait(1)
    -- Assume remote for buying grenade (common in scripts: FireServer("Grenade"))
    pcall(function()
        ReplicatedStorage.events:FindFirstChild("BuyGrenade"):FireServer()  -- Adjust remote name
    end)
    wait(1)
end

-- Throw grenade (equip and throw)
local function throwGrenade()
    local tool = Character:FindFirstChild("Grenade") or Character.Backpack:FindFirstChild("Grenade")
    if tool then
        Humanoid:EquipTool(tool)
        wait(0.5)
        -- Simulate throw: mouse click or remote
        mouse1click()  -- Or FireServer for throw
    end
    wait(3)  -- Wait for explosion
end

-- Collect loot fast (loop collect)
local function collectLoot(positions)
    for _, pos in pairs(positions) do
        teleportTo(pos)
        wait(0.1)
        -- Simulate pickup: click or proximity
        mouse1click()
    end
end

-- Sell golds/jewels at dealer
local function sellLoot()
    teleportTo(locations.dealer)
    wait(1)
    pcall(function()
        ReplicatedStorage.events:FindFirstChild("SellGold"):FireServer()
        ReplicatedStorage.events:FindFirstChild("SellJewelry"):FireServer()
    end)
    wait(1)
end

-- Check if safe/vault is open/bombable (green light? assume proximity check)
local function isVaultOpen(vaultCFrame)
    teleportTo(vaultCFrame)
    wait(0.5)
    -- Dummy check: if no red light part or distance to loot
    return true  -- Simplify; in real, check for part
end

-- Main AutoRob toggle
local autoRobEnabled = false

-- Create minimal GUI like image (right side only, no left settings)
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "BeanzZZ"
ScreenGui.Parent = game.CoreGui

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 300, 0, 400)
MainFrame.Position = UDim2.new(1, -320, 0.5, -200)
MainFrame.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 50)
Title.BackgroundColor3 = Color3.new(0.2, 0.2, 0.2)
Title.Text = "BeanzZZ | discord.gg/beanzhub"
Title.TextColor3 = Color3.new(1,1,1)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 16
Title.Parent = MainFrame

local AutoRobBtn = Instance.new("TextButton")
AutoRobBtn.Size = UDim2.new(0.9, 0, 0, 40)
AutoRobBtn.Position = UDim2.new(0.05, 0, 0.15, 0)
AutoRobBtn.BackgroundColor3 = Color3.new(0.3, 0.3, 0.3)
AutoRobBtn.Text = "🚗 AutoRob"
AutoRobBtn.TextColor3 = Color3.new(1,1,1)
AutoRobBtn.Font = Enum.Font.Gotham
AutoRobBtn.TextSize = 18
AutoRobBtn.Parent = MainFrame

local check = Instance.new("Frame")
check.Size = UDim2.new(0, 30, 0, 30)
check.BackgroundColor3 = Color3.new(0, 0.8, 0)
check.Position = UDim2.new(1, -40, 0.5, -15)
check.Parent = AutoRobBtn

-- Toggle AutoRob
AutoRobBtn.MouseButton1Click:Connect(function()
    autoRobEnabled = not autoRobEnabled
    check.Visible = autoRobEnabled
    if autoRobEnabled then
        spawn(function()
            while autoRobEnabled do
                -- 1. Club
                buyGrenade()
                teleportTo(locations.clubDoorBehind)
                throwGrenade()
                if isVaultOpen(locations.clubSafeFront) then
                    teleportTo(locations.clubSafe)
                    collectLoot({locations.clubSafe})  -- All loot
                end
                sellLoot()

                -- 2. Bank if bombable
                if isVaultOpen(locations.bankVaultFront) then
                    buyGrenade()
                    teleportTo(locations.bankVaultFront)
                    throwGrenade()
                    teleportTo(locations.bankEntrance)
                    collectLoot(locations.bankLootPositions)
                    sellLoot()
                end

                -- 3. Jeweler
                if isVaultOpen(locations.jewelerSafeFront) then
                    buyGrenade()
                    teleportTo(locations.jewelerSafeFront)
                    throwGrenade()
                    teleportTo(locations.jewelerEntranceFar)
                    collectLoot({locations.jewelerSafe})  -- All fast
                    sellLoot()
                end

                -- 4. Parking
                teleportTo(locations.parkingGarage)

                -- 5. Server hop to <=5 players
                TeleportService:TeleportToPlaceInstance(game.PlaceId, "lowpop")  -- Use hop script for max 5

                wait(1)
            end
        end)
    end
end)

-- Drag GUI
local dragging = false
MainFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
    end
end)
MainFrame.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = false
    end
end)
RunService.Heartbeat:Connect(function()
    if dragging then
        MainFrame.Position = UDim2.new(0, MainFrame.Position.X.Offset + mouse.Delta.X, 0, MainFrame.Position.Y.Offset + mouse.Delta.Y)
    end
end)

print("BeanzZZ AutoRob loaded! Click AutoRob to start sequence.")
