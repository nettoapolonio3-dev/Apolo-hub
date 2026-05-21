-- Garante o carregamento completo do jogo
repeat task.wait() until game:IsLoaded()

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local gui = player:WaitForChild("PlayerGui")

-- Variáveis de Controle das Funções
local autoGrabEnabled = false
local autoLeftEnabled = false
local autoRightEnabled = false
local batAimbotToggled = false
local optimizerEnabled = false
local espEnabled = false
local STEAL_RADIUS = 20

-- Tabelas para gerenciamento do ESP e Otimização
local espConnections = {}
local espObjects = {}
local originalTransparency = {}

-- Posições da movimentação
local POSITION_L1 = Vector3.new(-476.48, -6.28, 92.73)
local POSITION_L2 = Vector3.new(-483.12, -4.95, 94.80)
local POSITION_R1 = Vector3.new(-476.16, -6.52, 25.62)
local POSITION_R2 = Vector3.new(-483.04, -5.09, 23.14)

local BAT_MOVE_SPEED = 56.5
local BAT_ENGAGE_RANGE = 20
local BAT_LOOP_TIME = 0.3
local BAT_LOOK_DISTANCE = 50
local lastEquipTick_bat, lastUseTick_bat = 0, 0
local lookConnection_bat, attachment_bat, alignOrientation_bat

local autoLeftPhase, autoRightPhase = 1, 1
local char, hum, hrp

local function updateCharacterCache()
    char = player.Character or player.CharacterAdded:Wait()
    hum = char:WaitForChild("Humanoid")
    hrp = char:WaitForChild("HumanoidRootPart")
end
updateCharacterCache()
player.CharacterAdded:Connect(updateCharacterCache)

-- ==========================================
-- SISTEMA 1: ESP PLAYER
-- ==========================================
local function createESP(plr)
    if plr == player then return end
    if not plr.Character then return end
    if plr.Character:FindFirstChild("ApoloESP") then return end
    
    local c = plr.Character
    local charHrp = c:WaitForChild("HumanoidRootPart", 5)
    if not charHrp then return end
    
    local humanoid = c:FindFirstChildOfClass("Humanoid")
    if humanoid then humanoid.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None end
    
    local hitbox = Instance.new("BoxHandleAdornment")
    hitbox.Name = "ApoloESP"
    hitbox.Adornee = charHrp
    hitbox.Size = Vector3.new(4, 6, 2)
    hitbox.Color3 = Color3.fromRGB(160, 0, 255) -- Cor roxa idêntica ao antigo
    hitbox.Transparency = 0.5
    hitbox.ZIndex = 10
    hitbox.AlwaysOnTop = true
    hitbox.Parent = c
    espObjects[plr] = {box = hitbox, character = c}
end

local function removeESP(plr)
    pcall(function()
        if plr.Character then
            local hitbox = plr.Character:FindFirstChild("ApoloESP")
            if hitbox then hitbox:Destroy() end
            local humanoid = plr.Character:FindFirstChildOfClass("Humanoid")
            if humanoid then humanoid.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.Automatic end
        end
        if espObjects[plr] then espObjects[plr] = nil end
    end)
end

local function enableESP()
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= player then
            if plr.Character then pcall(function() createESP(plr) end) end
            local conn = plr.CharacterAdded:Connect(function()
                task.wait(0.1)
                if espEnabled then pcall(function() createESP(plr) end) end
            end)
            table.insert(espConnections, conn)
        end
    end
    local pAdded = Players.PlayerAdded:Connect(function(plr)
        if plr == player then return end
        local cAdded = plr.CharacterAdded:Connect(function()
            task.wait(0.1)
            if espEnabled then pcall(function() createESP(plr) end) end
        end)
        table.insert(espConnections, cAdded)
    end)
    table.insert(espConnections, pAdded)
end

local function disableESP()
    for _, plr in ipairs(Players:GetPlayers()) do pcall(function() removeESP(plr) end) end
    for _, conn in ipairs(espConnections) do
        if conn and conn.Connected then conn:Disconnect() end
    end
    espConnections = {}
    espObjects = {}
end

-- ==========================================
-- SISTEMA 2: OTIMIZAÇÃO (FPS BOOST / XRAY)
-- ==========================================
local function enableOptimizer()
    pcall(function()
        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        Lighting.GlobalShadows = false
        Lighting.Brightness = 2
        Lighting.FogEnd = 9e9
        Lighting.FogStart = 9e9
        for _, fx in ipairs(Lighting:GetChildren()) do
            if fx:IsA("PostEffect") then fx.Enabled = false end
        end
    end)

    pcall(function()
        for _, obj in ipairs(workspace:GetDescendants()) do
            if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam")
                or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then
                obj.Enabled = false
            elseif obj:IsA("SelectionBox") then
                obj:Destroy()
            elseif obj:IsA("BasePart") then
                obj.CastShadow = false
                obj.Material = Enum.Material.Plastic
                
                -- Aplica a transparência de X-Ray em estruturas estáticas/âncoras
                if obj.Anchored and (obj.Name:lower():find("base") or (obj.Parent and obj.Parent.Name:lower():find("base"))) then
                    originalTransparency[obj] = obj.LocalTransparencyModifier
                    obj.LocalTransparencyModifier = 0.88
                end
                
                for _, child in ipairs(obj:GetChildren()) do
                    if child:IsA("Decal") or child:IsA("Texture") or child:IsA("SurfaceAppearance") then
                        child:Destroy()
                    end
                end
            end
        end
    end)
    if hum then hum.WalkSpeed = 24 end -- Incremento de velocidade padrão do otimizador antigo
end

local function disableOptimizer()
    for part, value in pairs(originalTransparency) do
        if part then part.LocalTransparencyModifier = value end
    end
    originalTransparency = {}
    if hum then hum.WalkSpeed = 16 end
end

-- ==========================================
-- LÓGICAS OPERACIONAIS ANTIGAS
-- ==========================================
local function checkAutoGrab()
    if not autoGrabEnabled or not hrp then return end
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return end
    for _, plot in ipairs(plots:GetChildren()) do
        local yourBase = plot:FindFirstChild("PlotSign") and plot.PlotSign:FindFirstChild("YourBase")
        if yourBase and not yourBase.Enabled then 
            local podiums = plot:FindFirstChild("AnimalPodiums")
            if podiums then
                for _, podium in ipairs(podiums:GetChildren()) do
                    local spawnPart = podium:FindFirstChild("Base") and podium.Base:FindFirstChild("Spawn")
                    if spawnPart then
                        if (hrp.Position - spawnPart.Position).Magnitude <= STEAL_RADIUS then
                            local attachment = spawnPart:FindFirstChild("PromptAttachment")
                            local prompt = attachment and attachment:FindFirstChildOfClass("ProximityPrompt")
                            if prompt then fireproximityprompt(prompt) end
                        end
                    end
                end
            end
        end
    end
end

local function nearestPlayer_target()
    if not hrp then return nil, math.huge end
    local closest, minDist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= player and plr.Character then
            local tHRP = plr.Character:FindFirstChild("HumanoidRootPart")
            local tHum = plr.Character:FindFirstChildOfClass("Humanoid")
            if tHRP and tHum and tHum.Health > 0 then
                local distance = (tHRP.Position - hrp.Position).Magnitude
                if distance < minDist then minDist = distance; closest = tHRP end
            end
        end
    end
    return closest, minDist
end

local function closestLookTarget()
    if not hrp then return nil end
    local nearest, shortest = nil, BAT_LOOK_DISTANCE
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= player and plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") then
            local dist = (hrp.Position - plr.Character.HumanoidRootPart.Position).Magnitude
            if dist < shortest then shortest = dist; nearest = plr.Character.HumanoidRootPart end
        end
    end
    return nearest
end

local function stopLookAt()
    if lookConnection_bat then lookConnection_bat:Disconnect() lookConnection_bat = nil end
    if alignOrientation_bat then alignOrientation_bat:Destroy() alignOrientation_bat = nil end
    if attachment_bat then attachment_bat:Destroy() attachment_bat = nil end
    if hum then hum.AutoRotate = true end
end

local function startLookAt()
    if not hrp or not hum then return end
    hum.AutoRotate = false
    attachment_bat = Instance.new("Attachment", hrp)
    alignOrientation_bat = Instance.new("AlignOrientation")
    alignOrientation_bat.Attachment0 = attachment_bat
    alignOrientation_bat.Mode = Enum.OrientationAlignmentMode.OneAttachment
    alignOrientation_bat.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
    alignOrientation_bat.Responsiveness = 1000
    alignOrientation_bat.RigidityEnabled = true
    alignOrientation_bat.Parent = hrp
    lookConnection_bat = RunService.RenderStepped:Connect(function()
        if not hrp or not alignOrientation_bat then return end
        local target = closestLookTarget()
        if not target then return end
        alignOrientation_bat.CFrame = CFrame.lookAt(hrp.Position, Vector3.new(target.Position.X, hrp.Position.Y, target.Position.Z))
    end)
end

local function processAutoBat()
    if not batAimbotToggled or not hrp or not hum then return end
    hrp.CanCollide = false
    local target, distance = nearestPlayer_target()
    if not target then return end
    hrp.AssemblyLinearVelocity = (Vector3.new(target.Position.X, target.Position.Y, target.Position.Z) - hrp.Position).Unit * BAT_MOVE_SPEED
    if distance <= BAT_ENGAGE_RANGE then
        if tick() - lastEquipTick_bat >= BAT_LOOP_TIME then
            local tool = player.Backpack:FindFirstChild("Bat") or (char and char:FindFirstChild("Bat"))
            if tool then hum:EquipTool(tool) end
            lastEquipTick_bat = tick()
        end
        if tick() - lastUseTick_bat >= BAT_LOOP_TIME then
            local tool = char and char:FindFirstChild("Bat")
            if tool then tool:Activate() end
            lastUseTick_bat = tick()
        end
    end
end

-- Passagem livre de noclip ativada continuamente igual ao script antigo
RunService.Stepped:Connect(function()
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= player and plr.Character then
            for _, part in ipairs(plr.Character:GetDescendants()) do
                if part:IsA("BasePart") then part.CanCollide = false end
            end
        end
    end
end)

RunService.Heartbeat:Connect(processAutoBat)
task.spawn(function()
    while true do
        checkAutoGrab()
        if autoLeftEnabled and hrp then
            hrp.CFrame = CFrame.new(autoLeftPhase == 1 and POSITION_L1 or POSITION_L2)
            autoLeftPhase = autoLeftPhase == 1 and 2 or 1
        end
        if autoRightEnabled and hrp then
            hrp.CFrame = CFrame.new(autoRightPhase == 1 and POSITION_R1 or POSITION_R2)
            autoRightPhase = autoRightPhase == 1 and 2 or 1
        end
        task.wait(0.1)
    end
end)

-- ==========================================
-- INTERFACE DESIGN (APOLO HUB V3)
-- ==========================================
if gui:FindFirstChild("ApoloHub") then gui.ApoloHub:Destroy() end

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "ApoloHub"
screenGui.Parent = gui
screenGui.ResetOnSpawn = false

local menuButton = Instance.new("TextButton")
menuButton.Name = "MenuTrigger"
menuButton.Parent = screenGui
menuButton.Size = UDim2.new(0, 100, 0, 40)
menuButton.Position = UDim2.new(0, 20, 0, 310)
menuButton.BackgroundColor3 = Color3.fromRGB(10, 15, 25)
menuButton.Text = "MENU"
menuButton.TextColor3 = Color3.fromRGB(0, 255, 255)
menuButton.Font = Enum.Font.GothamBold
menuButton.TextSize = 16

local menuCorner = Instance.new("UICorner")
menuCorner.CornerRadius = UDim.new(0, 6)
menuCorner.Parent = menuButton

local menuStroke = Instance.new("UIStroke")
menuStroke.Parent = menuButton
menuStroke.Color = Color3.fromRGB(0, 255, 255)
menuStroke.Thickness = 1.5

local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Parent = screenGui
mainFrame.Size = UDim2.new(0, 300, 0, 470) -- Painel expandido para caber as novas funções
mainFrame.Position = UDim2.new(0, 20, 0, 200)
mainFrame.BackgroundColor3 = Color3.fromRGB(10, 15, 25)
mainFrame.Visible = false
mainFrame.Active = true
mainFrame.Draggable = true 

local frameCorner = Instance.new("UICorner")
frameCorner.CornerRadius = UDim.new(0, 10)
frameCorner.Parent = mainFrame

local glowLine = Instance.new("Frame")
glowLine.Parent = mainFrame
glowLine.Size = UDim2.new(1, 0, 0, 2)
glowLine.BackgroundColor3 = Color3.fromRGB(0, 255, 255)

local title = Instance.new("TextLabel")
title.Parent = mainFrame
title.Size = UDim2.new(1, 0, 0, 45)
title.Position = UDim2.new(0, 0, 0, 5)
title.BackgroundTransparency = 1
title.Text = "APOLO <font color='#ffffff'>HUB</font>"
title.RichText = true
title.TextColor3 = Color3.fromRGB(0, 255, 255)
title.Font = Enum.Font.GothamBlack
title.TextSize = 22

local close = Instance.new("TextButton")
close.Parent = mainFrame
close.Size = UDim2.new(0, 30, 0, 30)
close.Position = UDim2.new(1, -35, 0, 8)
close.BackgroundTransparency = 1
close.Text = "×"
close.TextColor3 = Color3.fromRGB(255, 255, 255)
close.TextSize = 25
close.Font = Enum.Font.Gotham

-- CONSTRUTOR DE BOTÕES
local function createToggle(name, pos, defaultState, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -40, 0, 40)
    btn.Position = UDim2.new(0, 20, 0, pos)
    btn.BackgroundColor3 = defaultState and Color3.fromRGB(0, 255, 255) or Color3.fromRGB(20, 30, 50)
    btn.Text = name .. " : " .. (defaultState and "ON" or "OFF")
    btn.TextColor3 = defaultState and Color3.fromRGB(10, 15, 25) or Color3.fromRGB(0, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 14
    btn.Parent = mainFrame

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = btn

    local state = defaultState
    btn.MouseButton1Click:Connect(function()
        state = not state
        btn.BackgroundColor3 = state and Color3.fromRGB(0, 255, 255) or Color3.fromRGB(20, 30, 50)
        btn.Text = name .. " : " .. (state and "ON" or "OFF")
        btn.TextColor3 = state and Color3.fromRGB(10, 15, 25) or Color3.fromRGB(0, 255, 255)
        callback(state)
    end)
end

-- Lista Completa de Toggles no Menu Principal
createToggle("AUTO GRAB", 60, false, function(v) autoGrabEnabled = v end)
createToggle("AUTO LEFT", 110, false, function(v) autoLeftEnabled = v; if v then autoRightEnabled = false end end)
createToggle("AUTO RIGHT", 160, false, function(v) autoRightEnabled = v; if v then autoLeftEnabled = false end end)
createToggle("AUTO BAT", 210, false, function(v) if v then startLookAt() else stopLookAt() end; batAimbotToggled = v end)
createToggle("OTIMIZADOR (FPS)", 260, false, function(v) optimizerEnabled = v; if v then enableOptimizer() else disableOptimizer() end end)
createToggle("PLAYER ESP", 310, false, function(v) espEnabled = v; if v then enableESP() else disableESP() end end)

-- SEÇÃO DA BARRA DE RADIUS
local radiusLabel = Instance.new("TextLabel")
radiusLabel.Size = UDim2.new(1, -40, 0, 20)
radiusLabel.Position = UDim2.new(0, 20, 0, 365)
radiusLabel.BackgroundTransparency = 1
radiusLabel.Text = "AUTO GRAB RADIUS"
radiusLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
radiusLabel.Font = Enum.Font.GothamBold
radiusLabel.TextSize = 12
radiusLabel.TextXAlignment = Enum.TextXAlignment.Left
radiusLabel.Parent = mainFrame

local radiusInput = Instance.new("TextBox")
radiusInput.Size = UDim2.new(1, -40, 0, 35)
radiusInput.Position = UDim2.new(0, 20, 0, 390)
radiusInput.BackgroundColor3 = Color3.fromRGB(15, 20, 35)
radiusInput.Text = tostring(STEAL_RADIUS)
radiusInput.TextColor3 = Color3.fromRGB(0, 255, 255)
radiusInput.Font = Enum.Font.GothamBold
radiusInput.TextSize = 15
radiusInput.ClearTextOnFocus = false
radiusInput.Parent = mainFrame

local inputCorner = Instance.new("UICorner")
inputCorner.CornerRadius = UDim.new(0, 6)
inputCorner.Parent = radiusInput

local inputStroke = Instance.new("UIStroke")
inputStroke.Parent = radiusInput
inputStroke.Color = Color3.fromRGB(0, 255, 255)
inputStroke.Thickness = 1

radiusInput.FocusLost:Connect(function()
    local val = tonumber(radiusInput.Text)
    if val then STEAL_RADIUS = val else radiusInput.Text = tostring(STEAL_RADIUS) end
end)

local status = Instance.new("TextLabel")
status.Parent = mainFrame
status.Size = UDim2.new(1, 0, 0, 20)
status.Position = UDim2.new(0, 0, 1, -25)
status.BackgroundTransparency = 1
status.Text = "Apolo Hub v3 | Sistema Completo"
status.TextColor3 = Color3.fromRGB(100, 100, 100)
status.Font = Enum.Font.Gotham
status.TextSize = 11

-- Conexões de visibilidade do Menu Principal
menuButton.MouseButton1Click:Connect(function() menuButton.Visible = false; mainFrame.Visible = true end)
close.MouseButton1Click:Connect(function() mainFrame.Visible = false; menuButton.Visible = true end)

print("Apolo Hub v3 Carregado com Sucesso!")
