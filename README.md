--打压狗屎 支持剑客就完事了
--泛滥人3578176440
--群聊347724155
--进群获取其他缝合源码
local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local Lighting = game:GetService("Lighting")
local StarterGui = game:GetService("StarterGui")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local SoundService = game:GetService("SoundService")
local TweenService = game:GetService("TweenService")
local TeleportService = game:GetService("TeleportService")
local TextChatService = game:GetService("TextChatService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local MarketplaceService = game:GetService("MarketplaceService")
local ProximityPromptService = game:GetService("ProximityPromptService")
local VirtualInputManager = Instance.new("VirtualInputManager")
local VirtualUser = cloneref and cloneref(game:GetService("VirtualUser")) or game:GetService("VirtualUser")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse()

local ESPLibrary = loadstring(game:HttpGet("https://raw.githubusercontent.com/Xingtaiduan/Script/refs/heads/main/ESPLibrary.lua"))()
local UI = loadstring(game:HttpGet("https://raw.githubusercontent.com/Xingtaiduan/Script/refs/heads/main/Library/Lua_Ware.lua"))()
local Notify = loadstring(game:HttpGet("https://raw.githubusercontent.com/Xingtaiduan/Script/refs/heads/main/Library/Notification.lua"))()

local flags = UI.flags
local connections = {}
local guiConnections = {}

local function addConnection(conn)
    table.insert(connections, conn)
end

local function cleanup()
    for _, conn in ipairs(connections) do
        pcall(function() conn:Disconnect() end)
    end
    connections = {}
end

local function notification(title, text, duration)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = title,
            Text = text,
            Duration = duration or 5
        })
    end)
end

local screenOffsetX, screenOffsetY = (function()
    local gui = Instance.new("ScreenGui", CoreGui)
    gui.Enabled = false
    local frame = Instance.new("Frame", gui)
    local clone = gui:Clone()
    gui.SafeAreaCompatibility = Enum.SafeAreaCompatibility.FullscreenExtension
    gui.ScreenInsets = Enum.ScreenInsets.None
    local dx, dy = clone.AbsolutePosition.X - gui.AbsolutePosition.X, clone.AbsolutePosition.Y - gui.AbsolutePosition.Y
    gui:Destroy()
    clone:Destroy()
    return dx, dy
end)()

local isMobile = table.find({Enum.Platform.Android, Enum.Platform.IOS}, UserInputService:GetPlatform()) ~= nil

local friendList = {}
task.spawn(function()
    local success, friends = pcall(function()
        return LocalPlayer:GetFriendsAsync()
    end)
    if success and friends then
        while true do
            local page = friends:GetCurrentPage()
            for _, friend in ipairs(page) do
                table.insert(friendList, friend.Id)
            end
            if friends.IsFinished then break end
            friends:AdvanceToNextPageAsync()
        end
    end
end)

local proximityPrompts = {}
local clickDetectors = {}
local touchTransmitters = {}
local npcModels = {}
local unanchoredParts = {}
local partCallbacks = {}

local function isNPC(model)
    return model:IsA("Model") and model.Parent and model.Parent:IsA("Model")
end

local function canNetworkOwn(part)
    return not part:IsGrounded() and not part.Anchored and part.ReceiveAge == 0
end

task.spawn(function()
    for _, descendant in ipairs(workspace:GetDescendants()) do
        if isNPC(descendant) then
            npcModels[descendant.Parent] = true
        end
        if descendant:IsA("BasePart") and not descendant.Anchored then
            table.insert(unanchoredParts, descendant)
        else
            if not proximityPrompts[descendant.ClassName] then
                proximityPrompts[descendant.ClassName] = {}
            end
            table.insert(proximityPrompts[descendant.ClassName], descendant)
        end
    end
end)

workspace.DescendantAdded:Connect(function(instance)
    if isNPC(instance) then
        npcModels[instance.Parent] = true
    elseif instance:IsA("BasePart") and not instance.Anchored then
        table.insert(unanchoredParts, instance)
    else
        if not proximityPrompts[instance.ClassName] then
            proximityPrompts[instance.ClassName] = {}
        end
        table.insert(proximityPrompts[instance.ClassName], instance)
        for _, callback in pairs(partCallbacks) do
            task.spawn(callback, instance)
        end
    end
end)

workspace.DescendantRemoving:Connect(function(instance)
    if isNPC(instance) then
        npcModels[instance.Parent] = nil
    end
end)

local function getTargets(includePlayers, includeNPCs, teamCheck)
    local targets = {}
    if includePlayers then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and not teamCheck then
                table.insert(targets, player.Character)
            end
        end
    end
    if includeNPCs then
        for model, _ in pairs(npcModels) do
            if model ~= LocalPlayer.Character then
                table.insert(targets, model)
            end
        end
    end
    return targets
end

flags.typeofName = "用户名(UserName)"
local playerNames = {}

local function updatePlayerNames()
    playerNames = {"All"}
    if flags.typeofName == "用户名(UserName)" then
        for _, player in ipairs(Players:GetChildren()) do
            table.insert(playerNames, player.Name)
        end
    elseif flags.typeofName == "昵称(DisplayName)" then
        for _, player in ipairs(Players:GetChildren()) do
            table.insert(playerNames, player.DisplayName)
        end
    end
    task.wait()
    table.sort(playerNames, function(a, b) return a:lower() < b:lower() end)
end

updatePlayerNames()

local function findPlayer(name, showError)
    local searchName = name:gsub("%s+", "")
    for _, player in ipairs(Players:GetPlayers()) do
        if player.Name:lower():match("^" .. searchName:lower()) or player.DisplayName:lower():match("^" .. searchName:lower()) then
            return player
        end
    end
    if showError then
        notification("XA：错误", "未找到玩家", 5)
    end
    return nil
end

local flingQueue = {}

local function startFling()
    if not selectedPlayer then
        return
    end
    local targetPlayers = {}
    if selectedPlayer == "All" then
        targetPlayers = Players:GetPlayers()
    else
        targetPlayers = {findPlayer(selectedPlayer)}
    end
    
    local originalCFrame = LocalPlayer.Character.HumanoidRootPart.CFrame
    local originalVelocity = LocalPlayer.Character.HumanoidRootPart.Velocity
    local originalDestroyHeight = workspace.FallenPartsDestroyHeight
    
    LocalPlayer.Character.Humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, false)
    workspace.FallenPartsDestroyHeight = 0/0
    
    for _, target in pairs(targetPlayers) do
        local targetChar = target.Character
        if targetChar then
            local humanoid = targetChar:FindFirstChildOfClass("Humanoid")
            local hrp = targetChar:FindFirstChild("HumanoidRootPart")
            if humanoid and hrp and humanoid.Health > 0 and target ~= LocalPlayer then
                local startTime = tick()
                local startPos = hrp.Position
                Camera.CameraSubject = targetChar
                
                if not target[getPlayerCheck] then
                    local function flingSequence(cframeOffset, angle)
                        LocalPlayer.Character.HumanoidRootPart.CFrame = CFrame.new(hrp.Position) * cframeOffset * angle
                        LocalPlayer.Character.HumanoidRootPart.Velocity = Vector3.new(0, 1000000, 0)
                    end
                    
                    task.wait()
                    local speed = hrp.Velocity.Magnitude
                    local moveDir = targetChar.Humanoid.MoveDirection
                    local angles = CFrame.Angles(math.rad(100), 0, 0)
                    
                    flingSequence(CFrame.new(0, 1.5, 0) + moveDir * speed / 1.25, angles)
                    task.wait()
                    flingSequence(CFrame.new(0, -1.5, 0) + moveDir * speed / 1.25, angles)
                    task.wait()
                    flingSequence(CFrame.new(2.25, 1.5, -2.25) + moveDir * speed / 1.25, angles)
                    task.wait()
                    flingSequence(CFrame.new(-2.25, -1.5, 2.25) + moveDir * speed / 1.25, angles)
                    task.wait()
                    flingSequence(CFrame.new(0, 1.5, 0) + moveDir, angles)
                    task.wait()
                    flingSequence(CFrame.new(0, -1.5, 0) + moveDir, angles)
                end
            end
        end
    end
    
    LocalPlayer.Character.Humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, true)
    if (LocalPlayer.Character.HumanoidRootPart.Position - originalCFrame.Position).Magnitude < 20 then
        LocalPlayer.Character.HumanoidRootPart.CFrame = originalCFrame * CFrame.new(0, 0.5, 0)
    end
    LocalPlayer.Character.Humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
    Camera.CameraSubject = LocalPlayer.Character.Humanoid
    
    for _, part in ipairs(LocalPlayer.Character:GetChildren()) do
        if part:IsA("BasePart") then
            part.Velocity = Vector3.zero
            part.RotVelocity = Vector3.zero
        end
    end
    task.wait()
    if not LocalPlayer.Character.HumanoidRootPart then
        workspace.FallenPartsDestroyHeight = originalDestroyHeight
        return true
    end
    return (LocalPlayer.Character.HumanoidRootPart.Position - originalCFrame.Position).Magnitude < 20
end

local function validatePlayer()
    if not selectedPlayer then
        notification("XA：错误", "请先选择玩家", 5)
        return true
    end
    if selectedPlayer ~= "All" and not Players:FindFir
