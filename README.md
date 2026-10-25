--[[
    🥚 STEAL EGG SCRIPT - v3.0
    Autor: (seu nome)
    Features: AutoSteal, AutoFusion, AutoEvent, BestEgg, ESP, GUI
--]]

-- ============================================================
-- 1. CONFIGURAÇÕES GLOBAIS
-- ============================================================
local Config = {
    -- Auto Steal
    AutoSteal        = true,
    StealDistance    = 15,
    StealDelay       = 0.3,
    OnlyStealBest    = false,
    IgnoreRarities   = {},          -- ex: {"Common", "Uncommon"}

    -- Auto Fusion
    AutoFusion       = true,
    FusionMode       = "Best",      -- "Best", "Random", "Custom"
    CustomFusions    = {
        -- {base1, base2, resultEsperado}
        {"Common", "Common", "Uncommon"},
        {"Uncommon", "Uncommon", "Rare"},
        {"Rare", "Rare", "Epic"},
        {"Epic", "Epic", "Legendary"},
        {"Legendary", "Legendary", "Mythic"},
        {"Mythic", "Mythic", "Secret"},
        {"Secret", "Secret", "Divine"},
        {"Divine", "Divine", "Celestial"},
        {"Celestial", "Celestial", "Godly"},
    },

    -- Auto Events
    AutoEvent        = true,
    EventPriority    = {"Meteor", "Golden", "Rainbow", "Lucky", "Boss"},
    EventDelay       = 0.5,

    -- Auto Hatch
    AutoHatch        = true,
    HatchBestOnly    = true,

    -- Auto Upgrade
    AutoUpgrade      = true,
    UpgradePriority  = {"Speed", "Capacity", "Steal", "Fusion"},

    -- Loja
    AutoBuy          = true,
    BuyOnlyAffordable= true,

    -- ESP
    ESP_Eggs         = true,
    ESP_Players      = false,
    ESP_Color        = Color3.fromRGB(0, 255, 150),

    -- Segurança
    AntiAFK          = true,
    Notify           = true,
}

-- ============================================================
-- 2. UTILITÁRIOS
-- ============================================================
local Players   = game:GetService("Players")
local RS        = game:GetService("ReplicatedStorage")
local WS        = game:GetService("Workspace")
local RunService= game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local LP        = Players.LocalPlayer

local function log(msg, color)
    print(("[EGG] %s"):format(msg))
    if Config.Notify and syn and syn.request then
        -- placeholder; use seu notificador aqui
    end
end

local function getChar()
    return LP.Character or LP.CharacterAdded:Wait()
end

local function getHRP()
    local c = getChar()
    return c:FindFirstChild("HumanoidRootPart")
end

local function distance(a, b)
    if not a or not b then return math.huge end
    return (a.Position - b.Position).Magnitude
end

local function getRarity(obj)
    -- detecta raridade por nome do objeto (ajuste conforme o jogo)
    local name = obj.Name:lower()
    local rarities = {
        common = 1, uncommon = 2, rare = 3, epic = 4,
        legendary = 5, mythic = 6, secret = 7, divine = 8,
        celestial = 9, godly = 10, transcendent = 11,
    }
    for r, v in pairs(rarities) do
        if name:find(r) then return r, v end
    end
    return "unknown", 0
end

-- ============================================================
-- 3. SISTEMA DE OVOS
-- ============================================================
local EggSystem = {}

function EggSystem.getEggs()
    local eggs = {}
    for _, obj in pairs(WS:GetDescendants()) do
        if obj:IsA("Model") or obj:IsA("Part") then
            local n = obj.Name:lower()
            if n:find("egg") or n:find("ovo") or obj:FindFirstChild("EggData") then
                table.insert(eggs, obj)
            end
        end
    end
    return eggs
end

function EggSystem.getBestEgg()
    local best, bestScore = nil, 0
    for _, egg in ipairs(EggSystem.getEggs()) do
        local rar, score = getRarity(egg)
        -- prioriza ovos raros e longe de outros players
        local distBonus = 0
        for _, p in pairs(Players:GetPlayers()) do
            if p ~= LP and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                local d = distance(egg.PrimaryPart or egg, p.Character.HumanoidRootPart)
                if d < 20 then distBonus = distBonus + 5 end
            end
        end
        local final = score * 100 + distBonus
        if final > bestScore then
            best, bestScore = egg, final
        end
    end
    return best
end

-- ============================================================
-- 4. AUTO STEAL
-- ============================================================
local Steal = {}

function Steal.stealEgg(egg)
    if not egg then return end
    local hrp = getHRP()
    if not hrp then return end

    local target = egg.PrimaryPart or egg
    local pos = target.Position

    -- teleporte suave
    local dist = distance(hrp, target)
    if dist > Config.StealDistance then
        hrp.CFrame = CFrame.new(pos + Vector3.new(0, 3, 0))
    end

    -- tenta trigger via remote (genérico)
    for _, remote in pairs(RS:GetDescendants()) do
        if remote:IsA("RemoteEvent") or remote:IsA("RemoteFunction") then
            local n = remote.Name:lower()
            if n:find("steal") or n:find("grab") or n:find("pick") or n:find("interact") then
                pcall(function()
                    if remote:IsA("RemoteEvent") then
                        remote:FireServer(egg)
                    else
                        remote:InvokeServer(egg)
                    end
                end)
            end
        end
    end

    -- fallback: ClickDetector
    local cd = egg:FindFirstChildWhichIsA("ClickDetector", true)
    if cd then
        pcall(function() fireclickdetector(cd) end)
    end
end

function Steal.loop()
    while Config.AutoSteal do
        local egg
        if Config.OnlyStealBest then
            egg = EggSystem.getBestEgg()
        else
            local eggs = EggSystem.getEggs()
            table.sort(eggs, function(a, b)
                local _, sa = getRarity(a); local _, sb = getRarity(b)
                return sa > sb
            end)
            egg = eggs[1]
        end
        if egg then
            local _, score = getRarity(egg)
            if not table.find(Config.IgnoreRarities, (getRarity(egg))) then
                Steal.stealEgg(egg)
            end
        end
        task.wait(Config.StealDelay)
    end
end

-- ============================================================
-- 5. AUTO FUSION
-- ============================================================
local Fusion = {}

function Fusion.findFusionRemote()
    for _, r in pairs(RS:GetDescendants()) do
        if (r:IsA("RemoteEvent") or r:IsA("RemoteFunction")) then
            local n = r.Name:lower()
            if n:find("fus") or n:find("combine") or n:find("merge") then
                return r
            end
        end
    end
    return nil
end

function Fusion.tryFusion(a, b)
    local remote = Fusion.findFusionRemote()
    if not remote then return false end
    pcall(function()
        if remote:IsA("RemoteEvent") then
            remote:FireServer(a, b)
        else
            remote:InvokeServer(a, b)
        end
    end)
    return true
end

function Fusion.getOwnedEggs()
    local owned = {}
    -- ajuste conforme o jogo (Backpack, Character, etc.)
    local c = getChar()
    if c then
        for _, obj in pairs(c:GetChildren()) do
            if obj.Name:lower():find("egg") or obj:FindFirstChild("EggData") then
                table.insert(owned, obj)
            end
        end
    end
    local bp = LP:FindFirstChildOfClass("Backpack")
    if bp then
        for _, obj in pairs(bp:GetChildren()) do
            if obj.Name:lower():find("egg") then table.insert(owned, obj) end
        end
    end
    return owned
end

function Fusion.loop()
    while Config.AutoFusion do
        local owned = Fusion.getOwnedEggs()
        if #owned >= 2 then
            if Config.FusionMode == "Best" then
                -- ordena por raridade
                table.sort(owned, function(a, b)
                    local _, sa = getRarity(a); local _, sb = getRarity(b)
                    return sa > sb
                end)
                Fusion.tryFusion(owned[1], owned[2])
            elseif Config.FusionMode == "Random" then
                local a = owned[math.random(#owned)]
                local b = owned[math.random(#owned)]
                if a ~= b then Fusion.tryFusion(a, b) end
            else -- Custom
                for _, combo in ipairs(Config.CustomFusions) do
                    local base1, base2 = combo[1], combo[2]
                    local a, b
                    for _, o in pairs(owned) do
                        if getRarity(o) == base1 then a = o break end
                    end
                    for _, o in pairs(owned) do
                        if getRarity(o) == base2 and o ~= a then b = o break end
                    end
                    if a and b then Fusion.tryFusion(a, b) break end
                end
            end
        end
        task.wait(1)
    end
end

-- ============================================================
-- 6. AUTO EVENT
-- ============================================================
local Events = {}

function Events.findActive()
    local active = {}
    for _, obj in pairs(WS:GetDescendants()) do
        local n = obj.Name:lower()
        for _, ev in ipairs(Config.EventPriority) do
            if n:find(ev:lower()) then
                table.insert(active, {obj = obj, type = ev, priority = table.find(Config.EventPriority, ev)})
                break
            end
        end
    end
    table.sort(active, function(a, b) return a.priority < b.priority end)
    return active
end

function Events.loop()
    while Config.AutoEvent do
        local active = Events.findActive()
        if #active > 0 then
            local ev = active[1]
            local hrp = getHRP()
            if hrp and ev.obj then
                local target = ev.obj.PrimaryPart or ev.obj
                if target then
                    hrp.CFrame = CFrame.new(target.Position + Vector3.new(0, 5, 0))
                    log("Evento detectado: " .. ev.type)
                end
            end
        end
        task.wait(Config.EventDelay)
    end
end

-- ============================================================
-- 7. AUTO HATCH + UPGRADE + BUY
-- ============================================================
local Utility = {}

function Utility.hatchLoop()
    while Config.AutoHatch do
        for _, r in pairs(RS:GetDescendants()) do
            if (r:IsA("RemoteEvent") or r:IsA("RemoteFunction")) then
                local n = r.Name:lower()
                if n:find("hatch") or n:find("open") or n:find("unbox") then
                    pcall(function()
                        if r:IsA("RemoteEvent") then r:FireServer()
                        else r:InvokeServer() end
                    end)
                end
            end
        end
        task.wait(2)
    end
end

function Utility.upgradeLoop()
    while Config.AutoUpgrade do
        for _, r in pairs(RS:GetDescendants()) do
            if r:IsA("RemoteEvent") then
                local n = r.Name:lower()
                if n:find("upgrade") or n:find("level") then
                    pcall(function() r:FireServer() end)
                end
            end
        end
        task.wait(3)
    end
end

function Utility.buyLoop()
    while Config.AutoBuy do
        for _, r in pairs(RS:GetDescendants()) do
            if r:IsA("RemoteEvent") then
                local n = r.Name:lower()
                if n:find("buy") or n:find("purchase") then
                    pcall(function() r:FireServer() end)
                end
            end
        end
        task.wait(5)
    end
end

-- ============================================================
-- 8. ESP DE OVOS
-- ============================================================
local ESP = {connections = {}}

function ESP.add(obj, labelText)
    if not obj or not obj:IsA("BasePart") then return end
    local bb = Instance.new("BillboardGui")
    bb.Name = "EggESP"
    bb.AlwaysOnTop = true
    bb.Size = UDim2.new(0, 100, 0, 30)
    bb.StudsOffset = Vector3.new(0, 3, 0)
    bb.Parent = obj

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, 0, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = labelText
    lbl.TextColor3 = Config.ESP_Color
    lbl.TextStrokeTransparency = 0
    lbl.TextScaled = true
    lbl.Font = Enum.Font.GothamBold
    lbl.Parent = bb
end

function ESP.clear()
    for _, obj in pairs(WS:GetDescendants()) do
        local g = obj:FindFirstChild("EggESP")
        if g then g:Destroy() end
    end
end

function ESP.loop()
    while Config.ESP_Eggs do
        ESP.clear()
        for _, egg in ipairs(EggSystem.getEggs()) do
            local rar = getRarity(egg)
            ESP.add(egg.PrimaryPart or (egg:IsA("BasePart") and egg), "🥚 " .. rar)
        end
        task.wait(2)
    end
end

-- ============================================================
-- 9. ANTI-AFK
-- ============================================================
if Config.AntiAFK then
    LP.Idled:Connect(function()
        local vim = game:GetService("VirtualUser")
        vim:CaptureController()
        vim:ClickButton2(Vector2.new())
    end)
end

-- ============================================================
-- 10. GUI (Rayfield-like minimal)
-- ============================================================
local function createGUI()
    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "EggStealGUI"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.Parent = (gethui and gethui()) or LP:WaitForChild("PlayerGui")

    local Main = Instance.new("Frame")
    Main.Size = UDim2.new(0, 300, 0, 420)
    Main.Position = UDim2.new(0, 20, 0, 100)
    Main.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
    Main.BorderSizePixel = 0
    Main.Active = true
    Main.Draggable = true
    Main.Parent = ScreenGui

    local UICorner = Instance.new("UICorner")
    UICorner.CornerRadius = UDim.new(0, 12)
    UICorner.Parent = Main

    local Title = Instance.new("TextLabel")
    Title.Size = UDim2.new(1, 0, 0, 40)
    Title.BackgroundColor3 = Color3.fromRGB(0, 200, 120)
    Title.Text = "🥚 EGG STEAL v3.0"
    Title.TextColor3 = Color3.new(1,1,1)
    Title.Font = Enum.Font.GothamBold
    Title.TextSize = 18
    Title.Parent = Main
    UICorner:Clone().Parent = Title

    local ToggleList = Instance.new("ScrollingFrame")
    ToggleList.Size = UDim2.new(1, -20, 1, -50)
    ToggleList.Position = UDim2.new(0, 10, 0, 45)
    ToggleList.BackgroundTransparency = 1
    ToggleList.BorderSizePixel = 0
    ToggleList.ScrollBarThickness = 4
    ToggleList.CanvasSize = UDim2.new(0, 0, 0, 700)
    ToggleList.Parent = Main

    local UIListLayout = Instance.new("UIListLayout")
    UIListLayout.Padding = UDim.new(0, 5)
    UIListLayout.Parent = ToggleList

    local function addToggle(name, key)
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(1, 0, 0, 32)
        btn.BackgroundColor3 = Config[key] and Color3.fromRGB(0,150,80) or Color3.fromRGB(50,50,70)
        btn.TextColor3 = Color3.new(1,1,1)
        btn.Font = Enum.Font.GothamSemibold
        btn.TextSize = 14
        btn.Text = (Config[key] and "✅ " or "❌ ") .. name
        btn.Parent = ToggleList
        UICorner:Clone().Parent = btn
        btn.MouseButton1Click:Connect(function()
            Config[key] = not Config[key]
            btn.BackgroundColor3 = Config[key] and Color3.fromRGB(0,150,80) or Color3.fromRGB(50,50,70)
            btn.Text = (Config[key] and "✅ " or "❌ ") .. name
            if key == "AutoSteal" and Config.AutoSteal then task.spawn(Steal.loop) end
            if key == "AutoFusion" and Config.AutoFusion then task.spawn(Fusion.loop) end
            if key == "AutoEvent" and Config.AutoEvent then task.spawn(Events.loop) end
            if key == "ESP_Eggs" and Config.ESP_Eggs then task.spawn(ESP.loop) end
        end)
    end

    addToggle("Auto Steal",  "AutoSteal")
    addToggle("Auto Fusion", "AutoFusion")
    addToggle("Auto Event",  "AutoEvent")
    addToggle("Auto Hatch",  "AutoHatch")
    addToggle("Auto Upgrade","AutoUpgrade")
    addToggle("Auto Buy",    "AutoBuy")
    addToggle("ESP Ovos",    "ESP_Eggs")
    addToggle("Anti-AFK",    "AntiAFK")
    addToggle("Só Melhores", "OnlyStealBest")

    -- Botão de modo de fusão
    local modeBtn = Instance.new("TextButton")
    modeBtn.Size = UDim2.new(1, 0, 0, 32)
    modeBtn.BackgroundColor3 = Color3.fromRGB(80,40,150)
    modeBtn.TextColor3 = Color3.new(1,1,1)
    modeBtn.Font = Enum.Font.GothamSemibold
    modeBtn.TextSize = 14
    modeBtn.Text = "🔀 Modo Fusão: " .. Config.FusionMode
    modeBtn.Parent = ToggleList
    UICorner:Clone().Parent = modeBtn
    modeBtn.MouseButton1Click:Connect(function()
        local modes = {"Best", "Random", "Custom"}
        local idx = table.find(modes, Config.FusionMode) or 1
        Config.FusionMode = modes[idx % #modes + 1]
        modeBtn.Text = "🔀 Modo Fusão: " .. Config.FusionMode
    end)

    -- Info
    local info = Instance.new("TextLabel")
    info.Size = UDim2.new(1, 0, 0, 60)
    info.BackgroundTransparency = 1
    info.TextColor3 = Color3.fromRGB(180,180,200)
    info.Font = Enum.Font.Gotham
    info.TextSize = 12
    info.TextWrapped = true
    info.Text = "Fusões configuradas: "..#Config.CustomFusions.."\nEventos: "..table.concat(Config.EventPriority, ", ")
    info.Parent = ToggleList
end

-- ============================================================
-- 11. INICIALIZAÇÃO
-- ============================================================
log("Iniciando script...")

createGUI()

-- inicia loops conforme config
if Config.AutoSteal  then task.spawn(Steal.loop)  end
if Config.AutoFusion then task.spawn(Fusion.loop) end
if Config.AutoEvent  then task.spawn(Events.loop) end
if Config.AutoHatch  then task.spawn(Utility.hatchLoop) end
if Config.AutoUpgrade then task.spawn(Utility.upgradeLoop) end
if Config.AutoBuy    then task.spawn(Utility.buyLoop) end
if Config.ESP_Eggs   then task.spawn(ESP.loop)    end
