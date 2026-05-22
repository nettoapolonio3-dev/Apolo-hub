-- ============================================================
-- apolo Hub  v1 (Auto-save button positions + all visuals)
-- Protection -> Visuals: ESP Players, Custom FOV
-- Auto Steal, Infinite Jump, Anti Ragdoll ON by default
-- ============================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local Stats = game:GetService("Stats")
local LP = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local _isfile = isfile or (syn and syn.isfile) or (getgenv and getgenv().isfile) or function() return false end
local _readfile = readfile or (syn and syn.readfile) or (getgenv and getgenv().readfile) or function() return nil end
local _writefile = writefile or (syn and syn.writefile) or (getgenv and getgenv().writefile) or function() end
local getconnections = getconnections or get_signal_cons or getconnects or (syn and syn.get_signal_cons)

-- ============================================================
-- STATE
-- ============================================================
local State = {
    normalSpeed = 55, carrySpeed = 28, laggerSpeed = 20.1,
    speedToggled = false, laggerEnabled = false,
    infJumpEnabled = true,
    antiRagdollEnabled = true,
    fpsBoostEnabled = false,
    guiVisible = true, uiLocked = false,
    isStealing = false,
    autoLeftEnabled = false, autoRightEnabled = false,
    autoLeftPhase = 1, autoRightPhase = 1,
    medusaLastUsed = 0, medusaDebounce = false, medusaCounterEnabled = false,
    batAimbotToggled = false, autoSwingEnabled = false,
    hittingCooldown = false,
    batCounterEnabled = false, batCounterDebounce = false,
    batAimbotSpeed = 52,
    dropEnabled = false, _tpInProgress = false,
    lastMoveDir = Vector3.new(0, 0, 0),
    stackButtonsHidden = false,
    stackButtonScale = 1,
    _prevCarry = 30, _prevSpeed = false,
    autoTPDownEnabled = false,
    -- Auto Medusa
    autoMedusaEnabled = false,
    medusaRange = 10,
    medusaCooldown = 0.12,
    medusaAttacking = false,
    -- ESP
    espEnabled = false,
    -- Custom FOV
    customFOVEnabled = false,
    originalFOV = Camera.FieldOfView,
}

local Keys = {
    speed = Enum.KeyCode.Q, guiHide = Enum.KeyCode.LeftControl,
    autoLeft = Enum.KeyCode.L, autoRight = Enum.KeyCode.R,
    lagger = Enum.KeyCode.Unknown, tpDown = Enum.KeyCode.Unknown,
    drop = Enum.KeyCode.H, aimbot = Enum.KeyCode.Unknown,
}

-- ============================================================
-- DEFAULT STACK BUTTON POSITIONS
-- ============================================================
local BTN_W = 64
local BTN_H = 54
local BTN_GAP = 5
local COLS = 2
local stackDefs = {
    { key = "autoLeft", label = "AUTO\nLEFT" },
    { key = "autoRight", label = "AUTO\nRIGHT" },
    { key = "aimbot", label = "AUTO\nBAT" },
    { key = "lagger", label = "LAGGER\nMODE" },
    { key = "drop", label = "DROP\nBR" },
    { key = "autoTPDown", label = "AUTO\nTP D" },
    { key = "carrySpeed", label = "CARRY\nSPEED" },
}
local GRID_W = COLS * (BTN_W + BTN_GAP) - BTN_GAP
local GRID_H = math.ceil(#stackDefs / COLS) * (BTN_H + BTN_GAP) - BTN_GAP

local function getDefaultStackPos(i)
    local col = (i - 1) % COLS
    local row = math.floor((i - 1) / COLS)
    return UDim2.new(1, -(GRID_W + 14) + col * (BTN_W + BTN_GAP), 0.5, -(GRID_H / 2) + row * (BTN_H + BTN_GAP))
end

-- Will store button positions for saving
local buttonPositions = {}  -- key -> UDim2

local function saveButtonPositions()
    for key, frame in pairs(stackWrappers) do
        if frame and frame.Position then
            local pos = frame.Position
            buttonPositions[key] = { XScale = pos.X.Scale, XOffset = pos.X.Offset, YScale = pos.Y.Scale, YOffset = pos.Y.Offset }
        end
    end
    pcall(saveConfig)  -- trigger full config save
end

local saveDebounce = nil
local function debouncedSaveButtonPositions()
    if saveDebounce then saveDebounce:Disconnect() end
    saveDebounce = task.delay(0.3, function()
        saveButtonPositions()
        saveDebounce = nil
    end)
end

-- ============================================================
-- AUTO STEAL SYSTEM (unchanged)
-- ============================================================
local AutoSteal = {
    Enabled = false,
    StealRadius = 59,
    StealDuration = 1.3,
    isStealing = false,
    StealData = {},
    screenGui = nil,
    barContainer = nil,
    progressBar = nil,
    statusLabel = nil,
    heartbeatConn = nil,
    progressConn = nil,
    scanTimer = 0,
    SCAN_INTERVAL = 0.15,
    cachedPrompts = {},
    lastFullScan = 0,
    FULL_SCAN_INTERVAL = 3,
}

local STEAL_COLORS = {
    Border = Color3.fromRGB(160, 160, 160),
    Progress = Color3.fromRGB(180, 180, 180),
}

local function createStealUI()
    if AutoSteal.screenGui then return end
    AutoSteal.screenGui = Instance.new("ScreenGui")
    AutoSteal.screenGui.Name = "ApoloStealProgress"
    AutoSteal.screenGui.ResetOnSpawn = false
    AutoSteal.screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    AutoSteal.screenGui.Parent = LP:WaitForChild("PlayerGui")

    AutoSteal.barContainer = Instance.new("Frame")
    AutoSteal.barContainer.Size = UDim2.new(0, 200, 0, 28)
    AutoSteal.barContainer.Position = UDim2.new(0.5, -100, 0.08, 0)
    AutoSteal.barContainer.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    AutoSteal.barContainer.BackgroundTransparency = 0.2
    AutoSteal.barContainer.BorderSizePixel = 0
    AutoSteal.barContainer.Parent = AutoSteal.screenGui
    AutoSteal.barContainer.Visible = false

    local containerCorner = Instance.new("UICorner")
    containerCorner.CornerRadius = UDim.new(0, 14)
    containerCorner.Parent = AutoSteal.barContainer
    local containerStroke = Instance.new("UIStroke")
    containerStroke.Color = STEAL_COLORS.Border
    containerStroke.Thickness = 1.5
    containerStroke.Transparency = 0.7
    containerStroke.Parent = AutoSteal.barContainer

    local barBackground = Instance.new("Frame")
    barBackground.Size = UDim2.new(1, -8, 1, -8)
    barBackground.Position = UDim2.new(0, 4, 0, 4)
    barBackground.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    barBackground.BackgroundTransparency = 0.3
    barBackground.BorderSizePixel = 0
    barBackground.Parent = AutoSteal.barContainer
    local barBgCorner = Instance.new("UICorner")
    barBgCorner.CornerRadius = UDim.new(0, 10)
    barBgCorner.Parent = barBackground

    AutoSteal.progressBar = Instance.new("Frame")
    AutoSteal.progressBar.Size = UDim2.new(0, 0, 1, 0)
    AutoSteal.progressBar.BackgroundColor3 = STEAL_COLORS.Progress
    AutoSteal.progressBar.BorderSizePixel = 0
    AutoSteal.progressBar.Parent = barBackground
    local barCorner = Instance.new("UICorner")
    barCorner.CornerRadius = UDim.new(0, 10)
    barCorner.Parent = AutoSteal.progressBar

    AutoSteal.statusLabel = Instance.new("TextLabel")
    AutoSteal.statusLabel.Size = UDim2.new(1, -16, 1, 0)
    AutoSteal.statusLabel.Position = UDim2.new(0, 8, 0, 0)
    AutoSteal.statusLabel.BackgroundTransparency = 1
    AutoSteal.statusLabel.Text = "READY"
    AutoSteal.statusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    AutoSteal.statusLabel.TextSize = 11
    AutoSteal.statusLabel.Font = Enum.Font.GothamBold
    AutoSteal.statusLabel.TextXAlignment = Enum.TextXAlignment.Left
    AutoSteal.statusLabel.Parent = AutoSteal.barContainer
end

local function getHRP()
    local c = LP.Character
    if c then
        return c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("Torso") or c:FindFirstChild("UpperTorso")
    end
    return nil
end

local function isMyPlotByName(pn)
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return false end
    local plot = plots:FindFirstChild(pn)
    if not plot then return false end
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local yb = sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") then
            return yb.Enabled == true
        end
    end
    return false
end

local function refreshPromptCache()
    local now = tick()
    if now - AutoSteal.lastFullScan < AutoSteal.FULL_SCAN_INTERVAL and #AutoSteal.cachedPrompts > 0 then
        return
    end
    AutoSteal.lastFullScan = now
    AutoSteal.cachedPrompts = {}

    local plots = workspace:FindFirstChild("Plots")
    if not plots then return end

    for _, plot in ipairs(plots:GetChildren()) do
        if isMyPlotByName(plot.Name) then continue end
        local pods = plot:FindFirstChild("AnimalPodiums")
        if not pods then continue end
        for _, pod in ipairs(pods:GetChildren()) do
            local base = pod:FindFirstChild("Base")
            if not base then continue end
            local spawn = base:FindFirstChild("Spawn")
            if not spawn then continue end
            local att = spawn:FindFirstChild("PromptAttachment")
            if att then
                for _, p in ipairs(att:GetChildren()) do
                    if p:IsA("ProximityPrompt") then
                        table.insert(AutoSteal.cachedPrompts, {
                            prompt = p,
                            spawnPos = spawn.Position
                        })
                        break
                    end
                end
            end
        end
    end
end

local function findNearestPrompt()
    local hrp = getHRP()
    if not hrp then return nil end
    refreshPromptCache()
    local nearest, nearestDist = nil, AutoSteal.StealRadius + 1
    for _, data in ipairs(AutoSteal.cachedPrompts) do
        local dist = (data.spawnPos - hrp.Position).Magnitude
        if dist <= AutoSteal.StealRadius and dist < nearestDist then
            nearestDist = dist
            nearest = data.prompt
        end
    end
    return nearest
end

local function updateProgress()
    if not AutoSteal.isStealing then return end
    local startTime = tick()
    local duration = AutoSteal.StealDuration
    if AutoSteal.statusLabel then AutoSteal.statusLabel.Text = "STEALING" end
    if AutoSteal.progressBar then AutoSteal.progressBar.Size = UDim2.new(0, 0, 1, 0) end
    if AutoSteal.progressConn then AutoSteal.progressConn:Disconnect() end
    AutoSteal.progressConn = RunService.Heartbeat:Connect(function()
        if not AutoSteal.isStealing then
            if AutoSteal.progressConn then AutoSteal.progressConn:Disconnect() end
            AutoSteal.progressConn = nil
            return
        end
        local elapsed = tick() - startTime
        local progress = math.min(elapsed / duration, 1)
        if AutoSteal.progressBar then AutoSteal.progressBar.Size = UDim2.new(progress, 0, 1, 0) end
        if progress >= 1 then
            if AutoSteal.progressConn then AutoSteal.progressConn:Disconnect() end
            AutoSteal.progressConn = nil
            if AutoSteal.statusLabel then AutoSteal.statusLabel.Text = "READY" end
        end
    end)
end

local function executeSteal(prompt)
    if AutoSteal.isStealing then return end
    if not AutoSteal.StealData[prompt] then
        AutoSteal.StealData[prompt] = { hold = {}, trigger = {}, ready = true }
        if getconnections then
            for _, c in ipairs(getconnections(prompt.PromptButtonHoldBegan)) do
                if c.Function then table.insert(AutoSteal.StealData[prompt].hold, c.Function) end
            end
            for _, c in ipairs(getconnections(prompt.Triggered)) do
                if c.Function then table.insert(AutoSteal.StealData[prompt].trigger, c.Function) end
            end
        else
            AutoSteal.StealData[prompt].useFallback = true
        end
    end
    local data = AutoSteal.StealData[prompt]
    if not data.ready then return end
    data.ready = false
    AutoSteal.isStealing = true
    if AutoSteal.barContainer then AutoSteal.barContainer.Visible = true end
    updateProgress()
    task.spawn(function()
        local ok = false
        if not data.useFallback then
            pcall(function()
                for _, f in ipairs(data.hold) do task.spawn(f) end
                task.wait(AutoSteal.StealDuration)
                for _, f in ipairs(data.trigger) do task.spawn(f) end
                ok = true
            end)
        end
        if not ok and fireproximityprompt then
            pcall(function() fireproximityprompt(prompt); ok = true end)
        end
        if not ok then
            pcall(function()
                prompt:InputHoldBegin()
                task.wait(AutoSteal.StealDuration)
                prompt:InputHoldEnd()
            end)
        end
        task.wait(0.05)
        if AutoSteal.barContainer then AutoSteal.barContainer.Visible = false end
        if AutoSteal.progressBar then AutoSteal.progressBar.Size = UDim2.new(0, 0, 1, 0) end
        if AutoSteal.statusLabel then AutoSteal.statusLabel.Text = "READY" end
        data.ready = true
        AutoSteal.isStealing = false
    end)
end

local function enableAutoSteal()
    if AutoSteal.Enabled then return end
    AutoSteal.Enabled = true
    if not AutoSteal.screenGui then createStealUI() end
    if AutoSteal.screenGui then AutoSteal.screenGui.Enabled = true end
    LP.CharacterAdded:Connect(function() AutoSteal.isStealing = false end)
    local lastScan = 0
    AutoSteal.heartbeatConn = RunService.Heartbeat:Connect(function()
        if not AutoSteal.Enabled then return end
        if AutoSteal.isStealing then return end
        local now = tick()
        if now - lastScan < AutoSteal.SCAN_INTERVAL then return end
        lastScan = now
        local success, prompt = pcall(findNearestPrompt)
        if success and prompt then pcall(executeSteal, prompt) end
    end)
end

local function disableAutoSteal()
    if not AutoSteal.Enabled then return end
    AutoSteal.Enabled = false
    if AutoSteal.screenGui then AutoSteal.screenGui.Enabled = false end
    if AutoSteal.heartbeatConn then AutoSteal.heartbeatConn:Disconnect(); AutoSteal.heartbeatConn = nil end
    if AutoSteal.progressConn then AutoSteal.progressConn:Disconnect(); AutoSteal.progressConn = nil end
    AutoSteal.isStealing = false
end

-- ============================================================
-- AUTO TP DOWN
-- ============================================================
local autoTPDownConnection = nil
local GROUND_Y = -8.8
local TP_THRESHOLD = 10

local function startAutoTPDown()
    if autoTPDownConnection then return end
    autoTPDownConnection = RunService.Heartbeat:Connect(function()
        if not State.autoTPDownEnabled then return end
        local char = LP.Character
        if not char then return end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        if hrp.Position.Y >= TP_THRESHOLD then
            hrp.CFrame = CFrame.new(hrp.Position.X, GROUND_Y, hrp.Position.Z)
            hrp.AssemblyLinearVelocity = Vector3.new(hrp.AssemblyLinearVelocity.X, 0, hrp.AssemblyLinearVelocity.Z)
        end
    end)
end

local function stopAutoTPDown()
    if autoTPDownConnection then
        autoTPDownConnection:Disconnect()
        autoTPDownConnection = nil
    end
end

-- ============================================================
-- AUTO LEFT / RIGHT (with button reset fix)
-- ============================================================
local POS_LEFT_1 = Vector3.new(-476.48, -6.28, 92.73)
local POS_LEFT_2 = Vector3.new(-483.12, -4.95, 94.80)
local POS_RIGHT_1 = Vector3.new(-476.16, -6.52, 25.62)
local POS_RIGHT_2 = Vector3.new(-483.04, -5.09, 23.14)

local Conns = { autoLeft = nil, autoRight = nil, antiRag = nil, aimbot = nil, anchor = {}, batCounter = nil, autoMedusa = nil }

-- Detect if player is holding brainrot (equipped in hand, not in backpack)
local function isHoldingBrainrot()
    local c = LP.Character
    if not c then return false end
    for _, v in ipairs(c:GetChildren()) do
        if v:IsA("Tool") then
            local n = v.Name:lower()
            if not n:find("bat") and not n:find("medusa") and not n:find("head") and not n:find("stone") then
                return true
            end
        end
    end
    return false
end

local function startAutoLeft()
    if Conns.autoLeft then Conns.autoLeft:Disconnect() end
    State.autoLeftPhase = 1
    Conns.autoLeft = RunService.Heartbeat:Connect(function()
        if not State.autoLeftEnabled then return end
        if isHoldingBrainrot() then return end
        local char = LP.Character
        if not char then return end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not hrp or not hum then return end
        local spd = State.normalSpeed
        if State.autoLeftPhase == 1 then
            local target = Vector3.new(POS_LEFT_1.X, hrp.Position.Y, POS_LEFT_1.Z)
            if (target - hrp.Position).Magnitude < 1 then
                State.autoLeftPhase = 2
                local dir = (POS_LEFT_2 - hrp.Position).Unit
                hum:Move(dir, false)
                hrp.AssemblyLinearVelocity = Vector3.new(dir.X * spd, hrp.AssemblyLinearVelocity.Y, dir.Z * spd)
                return
            end
            local dir = (POS_LEFT_1 - hrp.Position).Unit
            hum:Move(dir, false)
            hrp.AssemblyLinearVelocity = Vector3.new(dir.X * spd, hrp.AssemblyLinearVelocity.Y, dir.Z * spd)
        elseif State.autoLeftPhase == 2 then
            local target = Vector3.new(POS_LEFT_2.X, hrp.Position.Y, POS_LEFT_2.Z)
            if (target - hrp.Position).Magnitude < 1 then
                hum:Move(Vector3.zero, false)
                hrp.AssemblyLinearVelocity = Vector3.zero
                State.autoLeftEnabled = false
                if Conns.autoLeft then Conns.autoLeft:Disconnect(); Conns.autoLeft = nil end
                State.autoLeftPhase = 1
                if stackBtnRefs.autoLeft then stackBtnRefs.autoLeft.setOn(false) end
                return
            end
            local dir = (POS_LEFT_2 - hrp.Position).Unit
            hum:Move(dir, false)
            hrp.AssemblyLinearVelocity = Vector3.new(dir.X * spd, hrp.AssemblyLinearVelocity.Y, dir.Z * spd)
        end
    end)
end

local function stopAutoLeft()
    if Conns.autoLeft then Conns.autoLeft:Disconnect(); Conns.autoLeft = nil end
    State.autoLeftPhase = 1
    local char = LP.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then hum:Move(Vector3.zero, false) end
    end
    if stackBtnRefs.autoLeft then stackBtnRefs.autoLeft.setOn(false) end
end

local function startAutoRight()
    if Conns.autoRight then Conns.autoRight:Disconnect() end
    State.autoRightPhase = 1
    Conns.autoRight = RunService.Heartbeat:Connect(function()
        if not State.autoRightEnabled then return end
        if isHoldingBrainrot() then return end
        local char = LP.Character
        if not char then return end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not hrp or not hum then return end
        local spd = State.normalSpeed
        if State.autoRightPhase == 1 then
            local target = Vector3.new(POS_RIGHT_1.X, hrp.Position.Y, POS_RIGHT_1.Z)
            if (target - hrp.Position).Magnitude < 1 then
                State.autoRightPhase = 2
                local dir = (POS_RIGHT_2 - hrp.Position).Unit
                hum:Move(dir, false)
                hrp.AssemblyLinearVelocity = Vector3.new(dir.X * spd, hrp.AssemblyLinearVelocity.Y, dir.Z * spd)
                return
            end
            local dir = (POS_RIGHT_1 - hrp.Position).Unit
            hum:Move(dir, false)
            hrp.AssemblyLinearVelocity = Vector3.new(dir.X * spd, hrp.AssemblyLinearVelocity.Y, dir.Z * spd)
        elseif State.autoRightPhase == 2 then
            local ta
