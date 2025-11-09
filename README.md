-- VALENSUELA MODDER v8 - Integrated UI with Auto Chest & Auto Store & Auto Skills (Z X C V E B N)
-- Combines previous refactor with chest/store features and per-key auto-skill toggles.
-- Drop into your executor. Works reactively on respawn and provides the same draggable UI.

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local VirtualInputManager = game:GetService("VirtualInputManager")
local CoreGui = game:GetService("CoreGui")
local RunService = game:GetService("RunService")
local task = task

local player = Players.LocalPlayer

-- Settings (exposed internally)
local settings = {
    autoClick = false,
    autoSword = false,
    autoSkillMaster = false, -- master switch to enable per-key skill threads
    autoFarmAllBosses = false,
    autoFindChest = true,    -- corresponds to "Find Chest"
    autoStoreItems = true,   -- corresponds to "auto store item"
    -- per-key skill toggles
    skillZ = false,
    skillX = false,
    skillC = false,
    skillV = false,
    skillE = false,
    skillB = false,
    skillN = false,
}

local BOSS_NAMES = {
    ["Meteor"] = true, ["Toji"] = true, ["Madara"] = true, ["Femto"] = true, ["Ichigo"] = true,
    ["Anos"] = true, ["Rimuru"] = true, ["Isagi"] = true, ["Vergil"] = true, ["All Might"] = true,
    ["Gojo"] = true, ["Shadow King"] = true, ["VergilV2"] = true, ["Sans"] = true, ["Kj's"] = true,
}

-- Key mapping for skills
local SKILL_KEYMAP = {
    Z = Enum.KeyCode.Z,
    X = Enum.KeyCode.X,
    C = Enum.KeyCode.C,
    V = Enum.KeyCode.V,
    E = Enum.KeyCode.E,
    B = Enum.KeyCode.B,
    N = Enum.KeyCode.N,
}

-- Connection and loop management
local loops = {} -- name -> cancel function
local function startLoop(name, fn)
    if loops[name] and type(loops[name]) == "function" then
        loops[name]() -- cancel existing
    end
    local cancelled = false
    loops[name] = function() cancelled = true end
    task.spawn(function()
        local ok, err = pcall(function()
            while not cancelled do
                fn(function() return cancelled end)
            end
        end)
        if not ok then
            warn(("Loop '%s' errored: %s"):format(name, tostring(err)))
        end
    end)
end
local function stopLoop(name)
    if loops[name] and type(loops[name]) == "function" then
        loops[name]() -- set cancelled
        loops[name] = nil
    end
end

-- Helpers
local function getCharacterHrp()
    local char = player.Character
    if char then
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if hrp then return char, hrp end
    end
    return nil, nil
end

local function clickOnce()
    VirtualInputManager:SendMouseButtonEvent(0, 0, 0, true, game, 0)
    task.wait(0.06)
    VirtualInputManager:SendMouseButtonEvent(0, 0, 0, false, game, 0)
end

local function pressKeyOnce(key)
    VirtualInputManager:SendKeyEvent(true, key, false, game)
    task.wait(0.05)
    VirtualInputManager:SendKeyEvent(false, key, false, game)
end

-- Equip sword helper
local function equipSword()
    stopLoop("equipSword")
    if not settings.autoSword then return end
    startLoop("equipSword", function(isCancelled)
        local attempts = 0
        while not isCancelled() and attempts < 20 do
            attempts = attempts + 1
            local backpack = player:FindFirstChildOfClass("Backpack")
            local humanoid = player.Character and player.Character:FindFirstChild("Humanoid")
            if backpack and humanoid then
                for _, tool in ipairs(backpack:GetChildren()) do
                    if tool:IsA("Tool") and tool.Name:lower():find("sword") then
                        pcall(function() humanoid:EquipTool(tool) end)
                        return
                    end
                end
            end
            task.wait(0.4)
        end
    end)
end

-- Auto Attack
local function autoAttack()
    stopLoop("autoAttack")
    if not settings.autoClick then return end
    startLoop("autoAttack", function(isCancelled)
        local _, hrp = getCharacterHrp()
        while not hrp do
            if isCancelled() then return end
            task.wait(0.2)
            _, hrp = getCharacterHrp()
        end
        while not isCancelled() and settings.autoClick and player.Character and player.Character:FindFirstChild("HumanoidRootPart") do
            clickOnce()
            task.wait(0.17)
        end
    end)
end

-- Auto Skill master & per-key loops
local function startSkillLoopForKey(name)
    stopLoop("skill_"..name)
    if not settings.autoSkillMaster then return end
    if not settings["skill"..name] then return end
    local key = SKILL_KEYMAP[name]
    startLoop("skill_"..name, function(isCancelled)
        while not isCancelled() do
            if not settings.autoSkillMaster or not settings["skill"..name] then return end
            pressKeyOnce(key)
            task.wait(0.6) -- reasonable delay between presses (adjustable)
        end
    end)
end

local function refreshSkillLoops()
    for _, key in ipairs({"Z","X","C","V","E","B","N"}) do
        startSkillLoopForKey(key)
    end
end

-- Auto Farm Bosses (keeps original behavior)
local function findBosses()
    local bosses = {}
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("Model") and obj:FindFirstChild("Humanoid") and obj:FindFirstChild("HumanoidRootPart") and BOSS_NAMES[obj.Name] then
            table.insert(bosses, obj)
        end
    end
    return bosses
end

local function autoFarmAllBosses()
    stopLoop("autoFarm")
    if not settings.autoFarmAllBosses then return end
    startLoop("autoFarm", function(isCancelled)
        local _, hrp = getCharacterHrp()
        while not hrp do
            if isCancelled() then return end
            task.wait(0.5)
            _, hrp = getCharacterHrp()
        end
        while not isCancelled() and settings.autoFarmAllBosses and player.Character and player.Character:FindFirstChild("HumanoidRootPart") do
            local bossList = findBosses()
            for _, boss in ipairs(bossList) do
                if isCancelled() or not settings.autoFarmAllBosses then return end
                local bossHum = boss:FindFirstChild("Humanoid")
                local bossHrp = boss:FindFirstChild("HumanoidRootPart")
                local _, myHrp = getCharacterHrp()
                if bossHum and bossHrp and myHrp and bossHum.Health > 0 then
                    while not isCancelled() and settings.autoFarmAllBosses and bossHum and bossHum.Health > 0 and player.Character and player.Character:FindFirstChild("HumanoidRootPart") do
                        local _, myHrp2 = getCharacterHrp()
                        if not myHrp2 then break end
                        myHrp2.CFrame = bossHrp.CFrame + Vector3.new(0, 2, 0)
                        clickOnce()
                        -- use skills sequence
                        for _, k in ipairs({"Z","X","C","V","E","B","N"}) do
                            if isCancelled() or not settings.autoFarmAllBosses then break end
                            pressKeyOnce(SKILL_KEYMAP[k])
                            task.wait(0.12)
                        end
                        task.wait(0.28)
                    end
                    task.wait(0.5)
                end
            end
            task.wait(1.5)
        end
    end)
end

-- Auto Store Items
local function autoStoreLoop()
    stopLoop("autoStore")
    if not settings.autoStoreItems then return end
    startLoop("autoStore", function(isCancelled)
        while not isCancelled() do
            task.wait(0.6)
            if isCancelled() then return end
            if settings.autoStoreItems then
                pcall(function()
                    local pl = player
                    local backpack = pl:FindFirstChildOfClass("Backpack")
                    if backpack then
                        for _, v in pairs(backpack:GetChildren()) do
                            if v:IsA("Tool") and v.ToolTip == "" then
                                -- Move equipped tools with ToolTip ~= "" back to backpack
                                if pl.Character then
                                    for _, x in ipairs(pl.Character:GetChildren()) do
                                        if x:IsA("Tool") and x.ToolTip ~= "" then
                                            x.Parent = pl.Backpack
                                        end
                                    end
                                end
                                v.Parent = pl.Character
                                pcall(function()
                                    if game:GetService("ReplicatedStorage"):FindFirstChild("RemoteFunctions") and game:GetService("ReplicatedStorage").RemoteFunctions:FindFirstChild("Inventory") then
                                        game:GetService("ReplicatedStorage").RemoteFunctions.Inventory:InvokeServer("Add")
                                    end
                                end)
                            end
                        end
                    end
                end)
            end
        end
    end)
end

-- Find Chest (auto find & pickup)
local function fireProximityPromptsIn(root)
    for _, desc in ipairs(root:GetDescendants()) do
        if desc:IsA("ProximityPrompt") then
            pcall(function() fireproximityprompt(desc) end)
        end
    end
end

local function autoFindChestLoop()
    stopLoop("autoChest")
    if not settings.autoFindChest then return end
    startLoop("autoChest", function(isCancelled)
        while not isCancelled() do
            task.wait(0.6)
            if isCancelled() then return end
            if settings.autoFindChest then
                pcall(function()
                    local chest = Workspace:FindFirstChild("Drop") and Workspace.Drop:FindFirstChildOfClass("Part")
                    if chest then
                        -- teleport to chest safely
                        local char = player.Character
                        if char and char:FindFirstChild("HumanoidRootPart") then
                            char.HumanoidRootPart.CFrame = chest.CFrame
                        end
                        for _, item in ipairs(Workspace.Drop:GetChildren()) do
                            if item:IsA("Part") then
                                local char2 = player.Character
                                if char2 and char2:FindFirstChild("HumanoidRootPart") then
                                    char2.HumanoidRootPart.CFrame = item.CFrame
                                end
                                task.wait(0.12)
                                local prompt = item:FindFirstChildOfClass("ProximityPrompt")
                                if prompt then
                                    pcall(function()
                                        fireproximityprompt(prompt)
                                    end)
                                end
                                task.wait(0.12)
                            end
                        end
                    end
                end)
            end
        end
    end)
end

-- Central respawn handling to restart features that need character
local function onCharacterAdded()
    task.wait(0.2)
    if settings.autoClick then autoAttack() end
    if settings.autoSkillMaster then refreshSkillLoops() end
    if settings.autoFarmAllBosses then autoFarmAllBosses() end
    if settings.autoSword then equipSword() end
    if settings.autoStoreItems then autoStoreLoop() end
    if settings.autoFindChest then autoFindChestLoop() end
end
player.CharacterAdded:Connect(onCharacterAdded)
if player.Character then task.spawn(onCharacterAdded) end

-- UI (keeps visual style from your earlier script, draggable)
local function newCol(r,g,b) return Color3.fromRGB(r,g,b) end
local mainColor = newCol(26, 26, 32)
local frameBorder = newCol(66,66,66)
local buttonOn = newCol(200,50,50)
local buttonOff = newCol(50,200,200)
local buttonTextActive = newCol(255,255,230)
local buttonTextOff = newCol(255,255,255)

-- Clean previous GUI if exists
pcall(function()
    local existing = CoreGui:FindFirstChild("ValensuelaModderUI")
    if existing then existing:Destroy() end
end)

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "ValensuelaModderUI"
screenGui.Parent = CoreGui
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 460, 0, 420)
mainFrame.Position = UDim2.new(0, 320, 0, 150)
mainFrame.BackgroundColor3 = mainColor
mainFrame.BorderColor3 = frameBorder
mainFrame.BorderSizePixel = 4
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Parent = screenGui

local titleBar = Instance.new("TextLabel", mainFrame)
titleBar.Size = UDim2.new(1, 0, 0, 46)
titleBar.BackgroundColor3 = frameBorder
titleBar.BorderSizePixel = 0
titleBar.Font = Enum.Font.GothamBlack
titleBar.Text = "VALENSUELA MODDER // ZENO PIECE 2"
titleBar.TextColor3 = buttonOn
titleBar.TextSize = 21
titleBar.BackgroundTransparency = 0

local toggleButton = Instance.new("TextButton", mainFrame)
toggleButton.Size = UDim2.new(0, 34, 0, 32)
toggleButton.Position = UDim2.new(1, -42, 0, 8)
toggleButton.BackgroundColor3 = mainColor
toggleButton.BorderColor3 = buttonOn
toggleButton.Font = Enum.Font.GothamBold
toggleButton.TextSize = 19
toggleButton.Text = "X"
toggleButton.TextColor3 = buttonOn

toggleButton.MouseButton1Click:Connect(function()
    mainFrame.Visible = false
    if not screenGui:FindFirstChild("reopenBtn") then
        local reopenBtn = Instance.new("TextButton", screenGui)
        reopenBtn.Name = "reopenBtn"
        reopenBtn.Size = UDim2.new(0, 55, 0, 55)
        reopenBtn.Position = UDim2.new(0, 18, 0, 18)
        reopenBtn.BackgroundColor3 = mainColor
        reopenBtn.Text = "MOD"
        reopenBtn.Font = Enum.Font.GothamBold
        reopenBtn.TextSize = 17
        reopenBtn.TextColor3 = buttonOff
        reopenBtn.BorderColor3 = buttonOff
        reopenBtn.MouseButton1Click:Connect(function()
            mainFrame.Visible = true
            reopenBtn:Destroy()
        end)
    end
end)

local y = 54
local buttons = {}

local function makeBtn(frame, text, posY, onClick, btnCol, txtCol, sizeX)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, sizeX or 380, 0, 36)
    btn.Position = UDim2.new(0, 24, 0, posY)
    btn.BackgroundColor3 = btnCol
    btn.TextColor3 = txtCol
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 16
    btn.Text = text
    btn.BorderColor3 = mainColor
    btn.Parent = frame
    btn.MouseButton1Click:Connect(onClick)
    return btn
end

local function renderBtnState(btn, on, baseText)
    local btnCol = on and buttonOn or buttonOff
    local txtCol = on and buttonTextActive or buttonTextOff
    btn.Text = baseText..": "..(on and "ON" or "OFF")
    btn.BackgroundColor3 = btnCol
    btn.TextColor3 = txtCol
end

-- Setup main toggles using the existing layout style
local function setupToggle(varName, baseText, actionOnTrue, oneShot)
    buttons[varName] = makeBtn(mainFrame, baseText..": OFF", y, function()
        settings[varName] = not settings[varName]
        renderBtnState(buttons[varName], settings[varName], baseText)
        if settings[varName] and actionOnTrue then
            actionOnTrue()
        elseif not settings[varName] then
            -- stop corresponding loop by varName
            if varName == "autoClick" then stopLoop("autoAttack")
            elseif varName == "autoSkillMaster" then
                -- stop all skill loops
                for _, k in ipairs({"Z","X","C","V","E","B","N"}) do stopLoop("skill_"..k) end
            elseif varName == "autoStoreItems" then stopLoop("autoStore")
            elseif varName == "autoFindChest" then stopLoop("autoChest")
            elseif varName == "autoFarmAllBosses" then stopLoop("autoFarm")
            elseif varName == "autoSword" then stopLoop("equipSword") end
        end
        if oneShot and settings[varName] then
            -- revert if it was meant to be one-shot
            settings[varName] = false
            renderBtnState(buttons[varName], settings[varName], baseText)
        end
    end, buttonOff, buttonTextOff, 380)
    renderBtnState(buttons[varName], settings[varName], baseText)
    y = y + 44
end

-- main toggles
setupToggle("autoClick", "Auto Attack", autoAttack, false)
setupToggle("autoSword", "Select Sword", equipSword, false)
setupToggle("autoSkillMaster", "Auto Skills (Master)", function() refreshSkillLoops() end, false)
setupToggle("autoFarmAllBosses", "Auto Farm Bosses Automático", autoFarmAllBosses, false)
setupToggle("autoFindChest", "Find Chest (AutoChest)", autoFindChestLoop, false)
setupToggle("autoStoreItems", "Auto Store Items", autoStoreLoop, false)

-- Per-key skill toggles (smaller buttons, in two rows)
local skillNames = {"Z","X","C","V","E","B","N"}
local startX = 24
local perWidth = 60
local perGap = 8
local rowY = y + 10

for i, name in ipairs(skillNames) do
    local colIndex = (i-1) % 7
    local posX = startX + (colIndex * (perWidth + perGap))
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, perWidth, 0, 32)
    btn.Position = UDim2.new(0, posX, 0, rowY)
    btn.BackgroundColor3 = buttonOff
    btn.TextColor3 = buttonTextOff
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 14
    btn.Text = name..": OFF"
    btn.BorderColor3 = mainColor
    btn.Parent = mainFrame
    btn.MouseButton1Click:Connect(function()
        settings["skill"..name] = not settings["skill"..name]
        local on = settings["skill"..name]
        btn.BackgroundColor3 = on and buttonOn or buttonOff
        btn.TextColor3 = on and buttonTextActive or buttonTextOff
        btn.Text = name..": "..(on and "ON" or "OFF")
        -- start or stop loop for this skill if master is enabled
        if settings.autoSkillMaster and on then
            startSkillLoopForKey(name)
        else
            stopLoop("skill_"..name)
        end
    end)
end

y = rowY + 44

local infoLabel = Instance.new("TextLabel", mainFrame)
infoLabel.Size = UDim2.new(1, -22, 0, 120)
infoLabel.Position = UDim2.new(0, 12, 0, y+12)
infoLabel.BackgroundTransparency = 1
infoLabel.Font = Enum.Font.Gotham
infoLabel.TextSize = 13
infoLabel.TextXAlignment = Enum.TextXAlignment.Left
infoLabel.TextColor3 = newCol(210,210,210)
infoLabel.Text =
    "Painel arrastável, selecione opções:\n- Auto Attack: cliques automáticos\n- Select Sword: equipa espada do backpack\n- Auto Skills (Master) + teclas Z X C V E B N (liga cada tecla)\n- Find Chest: procura e pega drops (auto chest)\n- Auto Store Items: guarda itens automaticamente\n\nScript reinicia funções ao respawn automaticamente."

-- Initialize UI text state for skill buttons based on settings (in case defaults changed)
for _, child in ipairs(mainFrame:GetChildren()) do
    if child:IsA("TextButton") and child.Text ~= "X" and child.Name ~= "reopenBtn" then
        -- display already set during creation; this loop kept for compatibility
    end
end

-- Keep NoClip behaviour similar to original (optional)
RunService.Stepped:Connect(function()
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if hrp then
        if settings.autoClick or settings.autoFarmAllBosses then
            if not hrp:FindFirstChild("BodyClip") then
                local noclip = Instance.new("BodyVelocity")
                noclip.Name = "BodyClip"
                noclip.Parent = hrp
                noclip.MaxForce = Vector3.new(1e5, 1e5, 1e5)
                noclip.Velocity = Vector3.new(0, 0, 0)
            end
        else
            if hrp:FindFirstChild("BodyClip") then
                hrp.BodyClip:Destroy()
            end
        end
    end
end)

-- Expose settings to getgenv for compatibility with other scripts if needed
pcall(function()
    local g = getgenv and getgenv() or _G
    g.valensuela_modder = settings
end)
