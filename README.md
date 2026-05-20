local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local player = Players.LocalPlayer
local gui = player:WaitForChild("PlayerGui")

local function tpDown()
    local char = player.Character or player.CharacterAdded:Wait()
    local hrp = char:WaitForChild("HumanoidRootPart")
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {char}
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    local resultado = workspace:Raycast(hrp.Position, Vector3.new(0, -50, 0), raycastParams)
    if resultado then hrp.CFrame = CFrame.new(resultado.Position + Vector3.new(0, 3, 0)) else hrp.CFrame = hrp.CFrame + Vector3.new(0, -20, 0) end
    hrp.AssemblyLinearVelocity = Vector3.zero
end

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
menuButton.AutoButtonColor = false

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
mainFrame.Size = UDim2.new(0, 280, 0, 200)
mainFrame.Position = UDim2.new(0, 20, 0, 310)
mainFrame.BackgroundColor3 = Color3.fromRGB(10, 15, 25)
mainFrame.BorderSizePixel = 0
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
glowLine.BorderSizePixel = 0

local title = Instance.new("TextLabel")
title.Name = "ApoloTitle"
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

local container = Instance.new("Frame")
container.Parent = mainFrame
container.Size = UDim2.new(1, -40, 1, -80)
container.Position = UDim2.new(0, 20, 0, 60)
container.BackgroundTransparency = 1

local tpBtn = Instance.new("TextButton")
tpBtn.Name = "TPDownBtn"
tpBtn.Parent = container
tpBtn.Size = UDim2.new(1, 0, 0, 50)
tpBtn.BackgroundColor3 = Color3.fromRGB(20, 30, 50)
tpBtn.Text = "TP DOWN"
tpBtn.TextColor3 = Color3.fromRGB(0, 255, 255)
tpBtn.Font = Enum.Font.GothamBold
tpBtn.TextSize = 16
tpBtn.AutoButtonColor = false

local btnCorner = Instance.new("UICorner")
btnCorner.CornerRadius = UDim.new(0, 6)
btnCorner.Parent = tpBtn

local btnStroke = Instance.new("UIStroke")
btnStroke.Parent = tpBtn
btnStroke.Color = Color3.fromRGB(0, 255, 255)
btnStroke.Thickness = 1.2
btnStroke.Transparency = 0.5

local status = Instance.new("TextLabel")
status.Parent = mainFrame
status.Size = UDim2.new(1, 0, 0, 20)
status.Position = UDim2.new(0, 0, 1, -25)
status.BackgroundTransparency = 1
status.Text = "Status: Online | Tecla [G]"
status.TextColor3 = Color3.fromRGB(100, 100, 100)
status.Font = Enum.Font.Gotham
status.TextSize = 11

menuButton.MouseEnter:Connect(function() TweenService:Create(menuButton, TweenInfo.new(0.25), {BackgroundColor3 = Color3.fromRGB(0, 255, 255), TextColor3 = Color3.fromRGB(10, 15, 25)}):Play() end)
menuButton.MouseLeave:Connect(function() TweenService:Create(menuButton, TweenInfo.new(0.25), {BackgroundColor3 = Color3.fromRGB(10, 15, 25), TextColor3 = Color3.fromRGB(0, 255, 255)}):Play() end)
tpBtn.MouseEnter:Connect(function() TweenService:Create(tpBtn, TweenInfo.new(0.25), {BackgroundColor3 = Color3.fromRGB(0, 255, 255), TextColor3 = Color3.fromRGB(10, 15, 25)}):Play() end)
tpBtn.MouseLeave:Connect(function() TweenService:Create(tpBtn, TweenInfo.new(0.25), {BackgroundColor3 = Color3.fromRGB(20, 30, 50), TextColor3 = Color3.fromRGB(0, 255, 255)}):Play() end)

menuButton.MouseButton1Click:Connect(function() menuButton.Visible = false mainFrame.Visible = true end)
close.MouseButton1Click:Connect(function() mainFrame.Visible = false menuButton.Visible = true end)
tpBtn.MouseButton1Click:Connect(function() tpDown() end)

UIS.InputBegan:Connect(function(input, gp) if not gp and input.KeyCode == Enum.KeyCode.G then tpDown() end end)
print("Apolo Hub carregado com sucesso!")
