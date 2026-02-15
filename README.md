--[[
    TEAPOT ADMIN SUPREME V16 - LAG FIX & OPTIMIZED
    - Removido o loop pesado (GetDescendants) para evitar travamentos.
    - Sistema de detecção circular leve.
    - Gravidade média (0.75) e alcance médio (40).
]]

local player = game.Players.LocalPlayer
local mouse = player:GetMouse()
local RunService = game:GetService("RunService")

local function notify(text)
    local sg = Instance.new("ScreenGui", player.PlayerGui)
    local msg = Instance.new("TextLabel", sg)
    msg.Size = UDim2.new(0, 200, 0, 40)
    msg.Position = UDim2.new(0.5, -100, 0.2, 0)
    msg.BackgroundColor3 = Color3.fromRGB(40, 0, 90)
    msg.TextColor3 = Color3.new(1, 1, 1)
    msg.Text = text
    msg.TextScaled = true
    Instance.new("UICorner", msg)
    game.Debris:AddItem(sg, 1.2)
end

local function SetupAdmin()
    local character = player.Character or player.CharacterAdded:Wait()
    local humanoid = character:WaitForChild("Humanoid")
    local root = character:WaitForChild("HumanoidRootPart")

    -- --- ITEM 1: FLY ---
    local flyTool = Instance.new("Tool")
    flyTool.Name = "FLY"
    flyTool.Parent = player.Backpack
    
    local flyHandle = Instance.new("Part", flyTool)
    flyHandle.Name = "Handle"; flyHandle.Size = Vector3.new(0.1, 0.1, 0.1); flyHandle.Transparency = 1
    Instance.new("SpecialMesh", flyHandle).MeshId = "rbxassetid://1131102431"

    local flightGui = Instance.new("ScreenGui", player.PlayerGui)
    flightGui.Enabled = false
    local frame = Instance.new("Frame", flightGui)
    frame.Size = UDim2.new(0, 110, 0, 200); frame.Position = UDim2.new(0.85, 0, 0.35, 0)
    frame.BackgroundColor3 = Color3.new(0,0,0); frame.BackgroundTransparency = 0.5

    local function createBtn(text, pos)
        local btn = Instance.new("TextButton", frame)
        btn.Text = text; btn.Size = UDim2.new(1, 0, 0.25, 0); btn.Position = UDim2.new(0, 0, pos, 0)
        btn.BackgroundColor3 = Color3.fromRGB(50, 0, 110); btn.TextColor3 = Color3.new(1, 1, 1); btn.TextScaled = true
        Instance.new("UICorner", btn); return btn
    end

    local btnUp = createBtn("▲", 0)
    local btnDown = createBtn("▼", 0.25)
    local btnPlus = createBtn("+ Speed", 0.5)
    local btnMinus = createBtn("- Speed", 0.75)

    local flying = false
    local flySpeed = 50
    local bv = Instance.new("BodyVelocity")
    local bg = Instance.new("BodyGyro")
    bv.MaxForce = Vector3.new(9e9, 9e9, 9e9); bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)

    flyTool.Equipped:Connect(function()
        flying = true; flightGui.Enabled = true; bv.Parent = root; bg.Parent = root
        local w = Instance.new("Weld", flyHandle)
        w.Part0 = character.Head; w.Part1 = flyHandle; w.C0 = CFrame.new(0, 0, 0)
    end)

    flyTool.Unequipped:Connect(function()
        flying = false; flightGui.Enabled = false; bv.Parent = nil; bg.Parent = nil
        humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
    end)

    btnUp.MouseButton1Click:Connect(function() root.CFrame *= CFrame.new(0, 10, 0) end)
    btnDown.MouseButton1Click:Connect(function() root.CFrame *= CFrame.new(0, -10, 0) end)
    btnPlus.MouseButton1Click:Connect(function() flySpeed += 25; notify("Voo: "..flySpeed) end)
    btnMinus.MouseButton1Click:Connect(function() flySpeed -= 25; notify("Voo: "..flySpeed) end)

    -- --- ITEM 2: TEAPOT ADMIN ---
    local adminTool = Instance.new("Tool")
    adminTool.Name = "TEAPOT TURRET 🤪"
    adminTool.Parent = player.Backpack

    local handle = Instance.new("Part", adminTool)
    handle.Name = "Handle"; handle.Size = Vector3.new(0.1, 0.1, 0.1)
    local mesh = Instance.new("SpecialMesh", handle)
    mesh.MeshId = "rbxassetid://1029523"; mesh.Scale = Vector3.new(0.075, 0.075, 0.075)

    adminTool.Equipped:Connect(function()
        local w = Instance.new("Weld", handle)
        w.Part0 = character.Head; w.Part1 = handle; w.C0 = CFrame.new(0, 0.6, 0) 
        humanoid.WalkSpeed = 45
    end)
    adminTool.Unequipped:Connect(function() humanoid.WalkSpeed = 16 end)

    local lastClick = 0
    adminTool.Activated:Connect(function()
        local now = tick()
        if now - lastClick < 0.3 then root.CFrame = CFrame.new(mouse.Hit.p + Vector3.new(0, 3, 0)) end
        lastClick = now
    end)

    -- LOOP OTIMIZADO (Heartbeat a cada 0.1s para evitar lag)
    local lastGravityTick = 0
    RunService.Heartbeat:Connect(function()
        -- VOO
        if flying and flyTool.Parent == character then
            bv.Velocity = humanoid.MoveDirection * flySpeed
            bg.CFrame = workspace.CurrentCamera.CFrame
            humanoid:ChangeState(Enum.HumanoidStateType.Physics)
        end

        -- GRAVIDADE OTIMIZADA (Checa apenas a cada 0.1 segundos)
        local now = tick()
        if adminTool.Parent == character and now - lastGravityTick > 0.1 then
            lastGravityTick = now
            
            -- Detecta partes próximas de forma leve
            local region = OverlapParams.new()
            region.FilterType = Enum.RaycastFilterType.Exclude
            region.FilterDescendantsInstances = {character}
            
            local parts = workspace:GetPartBoundsInRadius(root.Position, 40, region)
            
            -- Remove forças antigas de partes que se afastaram
            for _, p in pairs(parts) do
                if p:IsA("BasePart") and not p.Anchored then
                    local bf = p:FindFirstChild("TPAntiGrav") or Instance.new("BodyForce", p)
                    bf.Name = "TPAntiGrav"
                    bf.Force = Vector3.new(0, p:GetMass() * workspace.Gravity * 0.75, 0)
                end
            end
        end

        -- ANTI-RAGDOLL
        if humanoid:GetState() == Enum.HumanoidStateType.FallingDown or humanoid:GetState() == Enum.HumanoidStateType.Ragdoll then
            humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
        end
    end)
end

SetupAdmin()
player.CharacterAdded:Connect(SetupAdmin)
