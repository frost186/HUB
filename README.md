-- ========== IIFE PARA ISOLAR O ESCOPO PRINCIPAL ==========
(function()
    -- ========== SISTEMA DE WHITELIST ==========
    local WHITELIST = {
        [7020163192] = true,  -- xeatz999
    }
    local function isWhitelisted()
        return WHITELIST[game.Players.LocalPlayer.UserId] == true
    end
    if not isWhitelisted() then
        game.Players.LocalPlayer:Kick("Você não tem permissão para usar esse script")
        return
    end

    -- ========== SERVIÇOS ==========
    local Players = game:GetService("Players")
    local UserInputService = game:GetService("UserInputService")
    local TweenService = game:GetService("TweenService")
    local TextChatService = game:GetService("TextChatService")
    local VirtualUser = game:GetService("VirtualUser")
    local Workspace = game:GetService("Workspace")
    local ReplicatedStorage = game:GetService("ReplicatedStorage")
    local TeleportService = game:GetService("TeleportService")
    local RunService = game:GetService("RunService")

    local player = Players.LocalPlayer
    local playerGui = player:WaitForChild("PlayerGui")
    local mouse = player:GetMouse()
    local camera = workspace.CurrentCamera

    -- ========== TABELA CENTRAL DE CONEXÕES ==========
    local allConnections = {}

    local function connectAndTrack(signal, callback)
        local conn = signal:Connect(callback)
        table.insert(allConnections, conn)
        return conn
    end

    -- ========== LIMPEZA DO MAPA ==========
    task.spawn(function()
        pcall(function()
            local barrier = workspace:FindFirstChild("Map")
            if barrier then
                barrier = barrier:FindFirstChild("Barrier")
                if barrier then
                    barrier = barrier:FindFirstChild("Barrier")
                    if barrier then
                        for _, child in ipairs(barrier:GetChildren()) do
                            if child:IsA("BasePart") then
                                child.CanCollide = (child.Name == "Teto")
                            end
                        end
                    end
                end
            end
        end)
    end)

    -- ========== DESTRUIR OBJETO "Joker.Reverse" ==========
    pcall(function()
        local joker = ReplicatedStorage:FindFirstChild("Joker")
        if joker then
            local reverse = joker:FindFirstChild("Reverse")
            if reverse then reverse:Destroy() end
        end
    end)

    -- ========== FUNÇÃO AUXILIAR PARA CHAT ==========
    local textChannel = TextChatService.TextChannels.RBXGeneral
    local function sendChat(cmd)
        if textChannel then
            textChannel:SendAsync(cmd)
        end
    end

    -- ========== REMOTE DE COMANDOS ==========
    local commandRemote
    pcall(function()
        commandRemote = ReplicatedStorage:WaitForChild("Packages"):WaitForChild("Remotes"):WaitForChild("8ouhNReX6mIuAx+H2HwvOIeUN0s=")
    end)
    local function fireRemoteAction(action, userId)
        if not commandRemote then return end
        pcall(function()
            commandRemote:FireServer({action = action, targetUserId = userId})
        end)
    end

    -- ========== VARIÁVEIS GLOBAIS ==========
    local target = nil
    local multiTargets = {}
    local viewing = false
    local viewingTarget = nil
    local viewCharacterAddedConn = nil  -- conexão para renascimento do alvo
    local frozen = false
    local waterWalkEnabled = false
    local khEnabled = false
    local espEnabled = true
    local multiSelectMode = false
    local ctrlPressed = false
    local espObjects = {}
    local guiVisible = true
    local isAnimating = false

    local targetMuted = false
    local muteBtn = nil

    local antiAFKConnection = nil
    local autoReCharAddedConn = nil
    local toolCharAddedConn = nil
    local toolBackpackConn = nil

    local antiCarryEnabled = false
    local antiCarryConnections = {}

    local chestAutoEnabled = false
    local chestConnections = {}

    local playerCards = {}
    local lastSearchText = ""

    -- ========== ANTI AFK ==========
    local function enableAntiAFK()
        if antiAFKConnection then return end
        antiAFKConnection = connectAndTrack(player.Idled, function()
            pcall(function()
                VirtualUser:CaptureController()
                VirtualUser:ClickButton2(Vector2.new())
            end)
        end)
    end
    local function disableAntiAFK()
        if antiAFKConnection then
            antiAFKConnection:Disconnect()
            antiAFKConnection = nil
        end
    end
    enableAntiAFK()

    -- ========== ANTI‑FORCE ==========
    local carpetName = "TwoSeaterRainbowMagicCarpet"
    local seatName = "Seat"
    local function isOnCarpet()
        local char = player.Character
        if not char then return false end
        local folder = workspace:FindFirstChild(player.Name)
        if folder then
            local carpet = folder:FindFirstChild(carpetName)
            if carpet and carpet:FindFirstChild(seatName) then return true end
        end
        return false
    end
    local function neutralizarBodyVelocity(obj)
        obj.MaxForce = Vector3.zero
        obj.Velocity = Vector3.zero
    end
    local function destruirComAtraso(obj)
        task.spawn(function()
            task.wait()
            if obj and obj.Parent then obj:Destroy() end
        end)
    end
    local function limparBodyVelocity(char)
        if not char then return end
        if isOnCarpet() then return end
        for _, obj in ipairs(char:GetDescendants()) do
            if obj:IsA("BodyVelocity") then
                neutralizarBodyVelocity(obj)
                destruirComAtraso(obj)
            end
        end
    end
    local function conectarLimpeza(char)
        limparBodyVelocity(char)
        connectAndTrack(char.DescendantAdded, function(obj)
            if isOnCarpet() then return end
            if obj:IsA("BodyVelocity") then
                neutralizarBodyVelocity(obj)
                destruirComAtraso(obj)
            end
        end)
    end
    if player.Character then conectarLimpeza(player.Character) end
    connectAndTrack(player.CharacterAdded, conectarLimpeza)

    connectAndTrack(RunService.Heartbeat, function()
        local char = player.Character
        if not char then return end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        if not isOnCarpet() and hrp.AssemblyLinearVelocity.Magnitude > 150 then
            hrp.AssemblyLinearVelocity = Vector3.zero
        end
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            if humanoid.Sit and not humanoid.SeatPart then humanoid.Sit = false end
            if humanoid.PlatformStand then humanoid.PlatformStand = false end
        end
    end)

    -- ========== ANTI-RAGDOLL ORIGINAL ==========
    local characterRagdollData = {}
    local ragdollCharAddedConn = nil

    local function stopCharacterRagdollProtection(char)
        local data = characterRagdollData[char]
        if not data then return end
        for _, conn in ipairs(data.connections) do conn:Disconnect() end
        characterRagdollData[char] = nil
    end
    local function proteger(personagem)
        local data = {connections = {}}
        local humanoid = personagem:WaitForChild("Humanoid")
        local conn1 = connectAndTrack(humanoid.StateChanged, function(_, estado)
            if estado == Enum.HumanoidStateType.Ragdoll then
                humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
            end
        end)
        table.insert(data.connections, conn1)
        local conn2 = connectAndTrack(humanoid:GetPropertyChangedSignal("PlatformStand"), function()
            if humanoid.PlatformStand then humanoid.PlatformStand = false end
        end)
        table.insert(data.connections, conn2)
        local conn3 = connectAndTrack(personagem.DescendantAdded, function(objeto)
            if objeto:IsA("BallSocketConstraint") and objeto.Name:find("Ragdoll:") then
                objeto:Destroy()
            elseif objeto:IsA("Part") and objeto.Name:find("RagdollCollider_") then
                objeto:Destroy()
            end
        end)
        table.insert(data.connections, conn3)
        for _, motor in ipairs(personagem:GetDescendants()) do
            if motor:IsA("Motor6D") then
                local conn4 = connectAndTrack(motor:GetPropertyChangedSignal("Enabled"), function()
                    if not motor.Enabled then motor.Enabled = true end
                end)
                table.insert(data.connections, conn4)
            end
        end
        local function limparInicial()
            for _, obj in ipairs(personagem:GetDescendants()) do
                if obj:IsA("BallSocketConstraint") and obj.Name:find("Ragdoll:") then obj:Destroy()
                elseif obj:IsA("Part") and obj.Name:find("RagdollCollider_") then obj:Destroy() end
            end
            local pastaRagdoll = personagem:FindFirstChild("RagdollCollision")
            if pastaRagdoll then pastaRagdoll:Destroy() end
            for _, motor in ipairs(personagem:GetDescendants()) do
                if motor:IsA("Motor6D") and not motor.Enabled then motor.Enabled = true end
            end
        end
        limparInicial()
        characterRagdollData[personagem] = data
    end
    local function startAntiRagdoll()
        if ragdollCharAddedConn then return end
        if player.Character then proteger(player.Character) end
        ragdollCharAddedConn = connectAndTrack(player.CharacterAdded, proteger)
    end
    local function stopAntiRagdoll()
        if ragdollCharAddedConn then
            ragdollCharAddedConn:Disconnect()
            ragdollCharAddedConn = nil
        end
        for char, _ in pairs(characterRagdollData) do stopCharacterRagdollProtection(char) end
    end
    startAntiRagdoll()

    -- ========== ANTI-RAGDOLL NOVO (ATRIBUTO + STEPPED) ==========
    local VALOR_TRAVA = -2
    local function ConfigurarHumanoid(char)
        if not char then return end
        local hum = char:WaitForChild("Humanoid", 5)
        if hum then
            hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false)
            hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false)
        end
    end
    connectAndTrack(RunService.Stepped, function()
        local char = player.Character
        if not char then return end
        local valorAtual = char:GetAttribute("Ragdoll")
        if valorAtual ~= VALOR_TRAVA then char:SetAttribute("Ragdoll", VALOR_TRAVA) end
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            local estadoAtual = hum:GetState()
            if estadoAtual == Enum.HumanoidStateType.Physics or estadoAtual == Enum.HumanoidStateType.Ragdoll or estadoAtual == Enum.HumanoidStateType.FallingDown then
                hum:ChangeState(Enum.HumanoidStateType.GettingUp)
            end
        end
    end)
    connectAndTrack(player.CharacterAdded, ConfigurarHumanoid)
    if player.Character then ConfigurarHumanoid(player.Character) end

    -- ========== FLEX FPS SPOOF ==========
    local FlexFPS = {}
    do
        local CONFIG = {
            ToolName = "FlexFPS",
            MinFPS = 337,
            MaxFPS = 346,
            TickRate = 1,
            AnimationId = "rbxassetid://98589164894779"
        }
        local cachedAnimation = Instance.new("Animation")
        cachedAnimation.AnimationId = CONFIG.AnimationId

        local State = {
            Remote = nil,
            Accumulator = 0,
            IsRunning = true,
            CurrentSpoof = nil,
            WatcherConnections = {},
            CurrentAnimTrack = nil
        }

        function FlexFPS.StopFlexSpoof()
            State.IsRunning = false
            if State.CurrentAnimTrack then
                State.CurrentAnimTrack:Stop(0.2)
                State.CurrentAnimTrack:Destroy()
                State.CurrentAnimTrack = nil
            end
            for _, conns in pairs(State.WatcherConnections) do
                for _, conn in ipairs(conns) do conn:Disconnect() end
            end
            State.WatcherConnections = {}
            State.Remote = nil
        end

        local function SendFakeFPS()
            if not State.Remote then return end
            local fakeValue = math.random(CONFIG.MinFPS, CONFIG.MaxFPS)
            State.CurrentSpoof = fakeValue
            pcall(function() State.Remote:FireServer(fakeValue) end)
        end

        local function StopEquipAnimation()
            if State.CurrentAnimTrack then
                State.CurrentAnimTrack:Stop(0.2)
                State.CurrentAnimTrack:Destroy()
                State.CurrentAnimTrack = nil
            end
        end

        local function PlayEquipAnimation(tool)
            StopEquipAnimation()
            local char = player.Character
            if not char then return end
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            if not humanoid then return end
            local animator = humanoid:FindFirstChildOfClass("Animator") or humanoid:WaitForChild("Animator", 2)
            if not animator then return end
            local track = animator:LoadAnimation(cachedAnimation)
            track:Play(0.2)
            State.CurrentAnimTrack = track
        end

        local function OnToolAdded(tool)
            if tool.Name ~= CONFIG.ToolName then return end
            if not tool:GetAttribute("FlexScriptsDisabled") then
                for _, child in ipairs(tool:GetDescendants()) do
                    if child:IsA("LocalScript") then child.Disabled = true end
                end
                tool:SetAttribute("FlexScriptsDisabled", true)
            end
            if not tool:GetAttribute("FlexHooked") then
                connectAndTrack(tool.Equipped, function() PlayEquipAnimation(tool) end)
                connectAndTrack(tool.Unequipped, StopEquipAnimation)
                tool:SetAttribute("FlexHooked", true)
            end
            if tool.Parent == player.Character then PlayEquipAnimation(tool) end
            local remote = tool:FindFirstChild("FPSRemote") or tool:WaitForChild("FPSRemote", 3)
            if remote then State.Remote = remote end
        end

        local function OnToolRemoved(tool)
            if tool.Name ~= CONFIG.ToolName then return end
            StopEquipAnimation()
            State.Remote = nil
        end

        local function WatchContainer(container)
            if not container or State.WatcherConnections[container] then return end
            local connAdded = connectAndTrack(container.ChildAdded, function(child)
                if child:IsA("Tool") then OnToolAdded(child) end
            end)
            local connRemoved = connectAndTrack(container.ChildRemoved, function(child)
                if child:IsA("Tool") then OnToolRemoved(child) end
            end)
            State.WatcherConnections[container] = {connAdded, connRemoved}
            for _, child in ipairs(container:GetChildren()) do
                if child:IsA("Tool") and child.Name == CONFIG.ToolName then
                    OnToolAdded(child)
                    break
                end
            end
        end

        local function CleanWatchers()
            local toRemove = {}
            for container in pairs(State.WatcherConnections) do
                if not container.Parent then table.insert(toRemove, container) end
            end
            for _, container in ipairs(toRemove) do
                for _, conn in ipairs(State.WatcherConnections[container]) do conn:Disconnect() end
                State.WatcherConnections[container] = nil
            end
        end

        local function SetupAllWatchers()
            CleanWatchers()
            local char = player.Character
            if char then
                WatchContainer(char)
                local backpack = player.Backpack
                if backpack then WatchContainer(backpack) end
            end
        end

        local function MainLoop(dt)
            if not State.IsRunning then return end
            State.Accumulator = State.Accumulator + dt
            if State.Accumulator >= CONFIG.TickRate then
                State.Accumulator = State.Accumulator % CONFIG.TickRate
                if State.Remote then SendFakeFPS() end
            end
        end

        SetupAllWatchers()
        connectAndTrack(player.CharacterAdded, function()
            task.wait()
            SetupAllWatchers()
        end)
        connectAndTrack(RunService.Heartbeat, MainLoop)
    end

    -- ========== AUTO RE ==========
    local function executarComandoSimples() sendChat("/re") end
    local function executarComandoERetornar()
        local character = player.Character
        if not character then return end
        local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
        if not humanoidRootPart then return end
        local cframeOriginal = humanoidRootPart.CFrame
        executarComandoSimples()
        task.delay(1, function()
            local novoCharacter = player.Character
            if novoCharacter then
                local novoHumanoidRootPart = novoCharacter:FindFirstChild("HumanoidRootPart")
                if novoHumanoidRootPart then novoHumanoidRootPart.CFrame = cframeOriginal end
            end
        end)
    end
    local function monitorarVida(humanoid)
        connectAndTrack(humanoid.HealthChanged, function(health)
            if health > 0 and (health / humanoid.MaxHealth) <= 0.2 then executarComandoSimples() end
        end)
    end
    local function monitorarPersonagem(character)
        local humanoid = character:WaitForChild("Humanoid")
        connectAndTrack(humanoid.Died, executarComandoSimples)
        monitorarVida(humanoid)
    end
    local function iniciarAutoRe()
        if player.Character then monitorarPersonagem(player.Character) end
    end
    iniciarAutoRe()
    if autoReCharAddedConn then autoReCharAddedConn:Disconnect() end
    autoReCharAddedConn = connectAndTrack(player.CharacterAdded, monitorarPersonagem)

    -- ========== COMANDOS K/H ==========
    local comandosAtivos = true
    local function enviarComandos()
        if not comandosAtivos or not khEnabled then return end
        task.spawn(function() sendChat("/korblox") end)
        task.spawn(function()
            task.wait(0.2)
            sendChat("/headless")
        end)
    end
    connectAndTrack(player.CharacterAdded, function()
        if khEnabled then task.wait(0.2); enviarComandos() end
    end)

    -- ========== WATER WALK ==========
    local walkWaterPart = nil
    local waterWalkHeartbeatConn = nil
    local WATER_SURFACE_Y = -2

    local function startWaterWalkHeartbeat()
        if waterWalkHeartbeatConn then
            waterWalkHeartbeatConn:Disconnect()
            waterWalkHeartbeatConn = nil
        end
        waterWalkHeartbeatConn = RunService.Heartbeat:Connect(function()
            if not waterWalkEnabled or not walkWaterPart then return end
            local char = player.Character
            if not char then return end
            local root = char:FindFirstChild("HumanoidRootPart")
            if not root then return end
            walkWaterPart.CFrame = CFrame.new(root.Position.X, WATER_SURFACE_Y, root.Position.Z)
        end)
    end

    local function updateWalkWater()
        if waterWalkEnabled then
            if not walkWaterPart then
                walkWaterPart = Instance.new("Part")
                walkWaterPart.Name = "WalkWater"
                walkWaterPart.Size = Vector3.new(2048, 0.2, 2048)
                walkWaterPart.Transparency = 1
                walkWaterPart.Anchored = true
                walkWaterPart.Material = Enum.Material.SmoothPlastic
                walkWaterPart.Parent = workspace
            end
            walkWaterPart.CanCollide = true
            startWaterWalkHeartbeat()
        else
            if walkWaterPart then
                walkWaterPart.CanCollide = false
            end
            if waterWalkHeartbeatConn then
                waterWalkHeartbeatConn:Disconnect()
                waterWalkHeartbeatConn = nil
            end
        end
    end

    -- ========== ANTI CARRY ==========
    local function stopAntiCarry()
        for _, conn in ipairs(antiCarryConnections) do
            if conn and conn.Disconnect then pcall(conn.Disconnect, conn)
            elseif conn and conn.cancel then pcall(task.cancel, conn.cancel) end
        end
        table.clear(antiCarryConnections)
    end
    local function monitorAntiCarryCharacter(char)
        local humanoid = char:WaitForChild("Humanoid", 5)
        if not humanoid then return end
        local attrSignal = char:GetAttributeChangedSignal("IsCarried")
        if attrSignal then
            local signalConn = connectAndTrack(attrSignal, function()
                if char:GetAttribute("IsCarried") == true then sendChat("/re") end
            end)
            table.insert(antiCarryConnections, signalConn)
        else
            local pollingTask = task.spawn(function()
                while char.Parent and humanoid.Health > 0 and antiCarryEnabled do
                    if char:GetAttribute("IsCarried") == true then sendChat("/re") end
                    task.wait(0.3)
                end
            end)
            table.insert(antiCarryConnections, {cancel = pollingTask})
        end
    end
    local function startAntiCarry()
        stopAntiCarry()
        if not antiCarryEnabled then return end
        local conn = connectAndTrack(player.CharacterAdded, monitorAntiCarryCharacter)
        table.insert(antiCarryConnections, conn)
        if player.Character then monitorAntiCarryCharacter(player.Character) end
    end

    -- ========== AUTO CHEST ==========
    local function stopChestAuto()
        chestAutoEnabled = false
        for _, conn in ipairs(chestConnections) do
            conn:Disconnect()
        end
        table.clear(chestConnections)
    end
    local function startChestAuto()
        if chestAutoEnabled then return end
        chestAutoEnabled = true
        local success, err = pcall(function()
            local ChestSpawn = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("ChestSpawn", 5)
            if not ChestSpawn then return end
            local GetChestTarget = ChestSpawn:WaitForChild("GetChestTarget", 5)
            local ChestActiveFlag = ChestSpawn:WaitForChild("ChestActiveFlag", 5)
            if not (GetChestTarget and ChestActiveFlag) then return end
            local function teleportToChest()
                local success2, result = pcall(function() return GetChestTarget:InvokeServer() end)
                if success2 and type(result) == "table" and result.active and typeof(result.pos) == "Vector3" then
                    local character = player.Character
                    if character then
                        local rootPart = character:FindFirstChild("HumanoidRootPart")
                        if rootPart then
                            rootPart.CFrame = CFrame.new(result.pos + Vector3.new(0, 3, 0))
                        end
                    end
                end
            end
            local flagConn = connectAndTrack(ChestActiveFlag:GetPropertyChangedSignal("Value"), function()
                if ChestActiveFlag.Value == true then
                    teleportToChest()
                end
            end)
            table.insert(chestConnections, flagConn)
            if ChestActiveFlag.Value == true then
                teleportToChest()
            end
        end)
        if not success then
            warn("Chest auto failed:", err)
            stopChestAuto()
        end
    end

    -- ========== WARN COUNT ==========
    local warnCache = {}
    local function findWarnObject(plr)
        if warnCache[plr] and warnCache[plr].object then
            if warnCache[plr].object.Parent then return warnCache[plr].object else warnCache[plr] = nil end
        end
        local direct = plr:FindFirstChild("WarnCount")
        if direct and (direct:IsA("IntValue") or direct:IsA("NumberValue") or direct:IsA("StringValue")) then
            warnCache[plr] = {object = direct}
            return direct
        end
        local leaderstats = plr:FindFirstChild("leaderstats")
        if leaderstats then
            local lsWarn = leaderstats:FindFirstChild("WarnCount")
            if lsWarn and (lsWarn:IsA("IntValue") or lsWarn:IsA("NumberValue") or lsWarn:IsA("StringValue")) then
                warnCache[plr] = {object = lsWarn}
                return lsWarn
            end
        end
        local found = plr:FindFirstChild("WarnCount", true)
        if found and (found:IsA("IntValue") or found:IsA("NumberValue") or found:IsA("StringValue")) then
            warnCache[plr] = {object = found}
            return found
        end
        local attr = plr:GetAttribute("WarnCount")
        if attr ~= nil then return {type = "attribute", value = attr} end
        return nil
    end
    connectAndTrack(Players.PlayerRemoving, function(plr) warnCache[plr] = nil end)

    -- ========== ESP ==========
    local function createESP(plr)
        if plr == player then return end
        local function addESP(character)
            task.wait(0.5)
            local hrp = character:FindFirstChild("HumanoidRootPart")
            if not hrp then return end
            if character:FindFirstChild("Highlight") then character:FindFirstChild("Highlight"):Destroy() end
            if hrp:FindFirstChild("BillboardGui") then hrp:FindFirstChild("BillboardGui"):Destroy() end
            local highlight = Instance.new("Highlight")
            highlight.FillColor = Color3.new(1, 1, 1)
            highlight.OutlineColor = Color3.new(1, 1, 1)
            highlight.FillTransparency = 1
            highlight.OutlineTransparency = 0
            highlight.Enabled = espEnabled
            highlight.Parent = character
            table.insert(espObjects, highlight)
            local billboard = Instance.new("BillboardGui")
            billboard.Size = UDim2.new(0, 200, 0, 40)
            billboard.StudsOffset = Vector3.new(0, 5, 0)
            billboard.AlwaysOnTop = true
            billboard.Enabled = espEnabled
            billboard.Parent = hrp
            table.insert(espObjects, billboard)
            local label = Instance.new("TextLabel")
            label.Size = UDim2.new(1, 0, 1, 0)
            label.BackgroundTransparency = 1
            label.TextColor3 = Color3.new(1, 1, 1)
            label.TextStrokeTransparency = 0.5
            label.TextSize = 16
            label.Font = Enum.Font.SourceSansBold
            label.Text = plr.DisplayName .. "\n@" .. plr.Name
            label.Parent = billboard
        end
        if plr.Character then addESP(plr.Character) end
        connectAndTrack(plr.CharacterAdded, addESP)
    end
    for _, p in ipairs(Players:GetPlayers()) do task.spawn(createESP, p) end
    connectAndTrack(Players.PlayerAdded, createESP)

    -- ========== TOOL TARGET ==========
    local function criarToolTarget()
        if player.Backpack:FindFirstChild("TARGET") or (player.Character and player.Character:FindFirstChild("TARGET")) then return end
        local tool = Instance.new("Tool")
        tool.Name = "TARGET"
        tool.RequiresHandle = false
        tool.ToolTip = "Click on players to set as target"
        tool.Parent = player.Backpack
    end
    criarToolTarget()
    toolCharAddedConn = connectAndTrack(player.CharacterAdded, criarToolTarget)
    toolBackpackConn = connectAndTrack(player.Backpack.ChildRemoved, function(child)
        if child.Name == "TARGET" then task.wait(0.1); criarToolTarget() end
    end)

    -- =====================================================
    -- FUNÇÕES PARA VIEW DINÂMICO (renascimento)
    -- =====================================================
    local function updateViewCamera()
        if not viewing or not target then return end
        local char = target.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then
                camera.CameraSubject = hum
            end
        end
    end

    local function setupViewConnection()
        if viewCharacterAddedConn then
            viewCharacterAddedConn:Disconnect()
            viewCharacterAddedConn = nil
        end
        if not viewing or not target then return end
        -- Conecta ao renascimento do alvo
        viewCharacterAddedConn = connectAndTrack(target.CharacterAdded, function(char)
            -- Aguarda o Humanoid carregar
            task.wait(0.5)
            if viewing and target then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then
                    camera.CameraSubject = hum
                end
            end
        end)
    end

    -- ========== CONSTRUÇÃO DA GUI COMPLETA ==========
    local function createUI()
        local gui = Instance.new("ScreenGui")
        gui.Name = "AdminPanelCompact"
        gui.ResetOnSpawn = false
        gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        gui.Parent = playerGui

        local mainContainer = Instance.new("Frame")
        mainContainer.Name = "MainContainer"
        mainContainer.Size = UDim2.new(0, 500, 0, 400)
        mainContainer.Position = UDim2.new(0.5, -250, 0.5, -200)
        mainContainer.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
        mainContainer.BackgroundTransparency = 0.05
        mainContainer.BorderSizePixel = 0
        mainContainer.Active = true
        mainContainer.ClipsDescendants = true
        mainContainer.Parent = gui

        local mainCorner = Instance.new("UICorner")
        mainCorner.CornerRadius = UDim.new(0, 10)
        mainCorner.Parent = mainContainer

        local mainStroke = Instance.new("UIStroke")
        mainStroke.Color = Color3.fromRGB(60, 60, 60)
        mainStroke.Thickness = 1
        mainStroke.Transparency = 0.3
        mainStroke.Parent = mainContainer

        local header = Instance.new("Frame")
        header.Name = "Header"
        header.Size = UDim2.new(1, 0, 0, 44)
        header.Position = UDim2.new(0, 0, 0, 0)
        header.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
        header.BackgroundTransparency = 0.2
        header.BorderSizePixel = 0
        header.Parent = mainContainer

        local headerCorner = Instance.new("UICorner")
        headerCorner.CornerRadius = UDim.new(0, 10)
        headerCorner.Parent = header

        local headerBottom = Instance.new("Frame")
        headerBottom.Size = UDim2.new(1, 0, 0, 10)
        headerBottom.Position = UDim2.new(0, 0, 1, -10)
        headerBottom.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
        headerBottom.BackgroundTransparency = 0.2
        headerBottom.BorderSizePixel = 0
        headerBottom.Parent = header

        local headerStroke = Instance.new("UIStroke")
        headerStroke.Color = Color3.fromRGB(60, 60, 60)
        headerStroke.Thickness = 1
        headerStroke.Transparency = 0.3
        headerStroke.Parent = header

        local headerTitle = Instance.new("TextLabel")
        headerTitle.Name = "Title"
        headerTitle.Size = UDim2.new(0, 180, 0, 22)
        headerTitle.Position = UDim2.new(0, 12, 0, 11)
        headerTitle.BackgroundTransparency = 1
        headerTitle.Text = "Admin Panel"
        headerTitle.TextColor3 = Color3.fromRGB(220, 220, 220)
        headerTitle.TextSize = 14
        headerTitle.Font = Enum.Font.GothamBold
        headerTitle.TextXAlignment = Enum.TextXAlignment.Left
        headerTitle.Parent = header

        local closeBtn = Instance.new("TextButton")
        closeBtn.Name = "CloseBtn"
        closeBtn.Size = UDim2.new(0, 24, 0, 24)
        closeBtn.Position = UDim2.new(1, -32, 0, 10)
        closeBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
        closeBtn.BackgroundTransparency = 0.6
        closeBtn.BorderSizePixel = 0
        closeBtn.Text = "X"
        closeBtn.TextColor3 = Color3.fromRGB(220, 220, 220)
        closeBtn.TextSize = 14
        closeBtn.Font = Enum.Font.GothamBold
        closeBtn.AutoButtonColor = false
        closeBtn.Parent = header

        local closeCorner = Instance.new("UICorner")
        closeCorner.CornerRadius = UDim.new(0, 5)
        closeCorner.Parent = closeBtn

        connectAndTrack(closeBtn.MouseEnter, function()
            closeBtn.BackgroundColor3 = Color3.fromRGB(220, 40, 40)
            closeBtn.BackgroundTransparency = 0.2
            closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        end)
        connectAndTrack(closeBtn.MouseLeave, function()
            closeBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
            closeBtn.BackgroundTransparency = 0.6
            closeBtn.TextColor3 = Color3.fromRGB(220, 220, 220)
        end)

        local contentArea = Instance.new("Frame")
        contentArea.Name = "ContentArea"
        contentArea.Size = UDim2.new(1, -14, 1, -52)
        contentArea.Position = UDim2.new(0, 7, 0, 48)
        contentArea.BackgroundTransparency = 1
        contentArea.Parent = mainContainer

        local leftPanel = Instance.new("Frame")
        leftPanel.Name = "LeftPanel"
        leftPanel.Size = UDim2.new(0, 190, 1, 0)
        leftPanel.Position = UDim2.new(0, 0, 0, 0)
        leftPanel.BackgroundTransparency = 1
        leftPanel.Parent = contentArea

        -- TARGET CARD
        local targetCard = Instance.new("Frame")
        targetCard.Name = "TargetCard"
        targetCard.Size = UDim2.new(1, 0, 0, 240)
        targetCard.Position = UDim2.new(0, 0, 0, 0)
        targetCard.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
        targetCard.BackgroundTransparency = 0.3
        targetCard.BorderSizePixel = 0
        targetCard.Parent = leftPanel

        local targetCorner = Instance.new("UICorner")
        targetCorner.CornerRadius = UDim.new(0, 8)
        targetCorner.Parent = targetCard

        local targetStroke = Instance.new("UIStroke")
        targetStroke.Color = Color3.fromRGB(70, 70, 70)
        targetStroke.Thickness = 1
        targetStroke.Transparency = 0.3
        targetStroke.Parent = targetCard

        local targetHeader = Instance.new("Frame")
        targetHeader.Size = UDim2.new(1, -14, 0, 20)
        targetHeader.Position = UDim2.new(0, 7, 0, 7)
        targetHeader.BackgroundTransparency = 1
        targetHeader.Parent = targetCard

        local targetTitle = Instance.new("TextLabel")
        targetTitle.Size = UDim2.new(1, 0, 1, 0)
        targetTitle.BackgroundTransparency = 1
        targetTitle.Text = "TARGET"
        targetTitle.TextColor3 = Color3.fromRGB(220, 220, 220)
        targetTitle.TextSize = 12
        targetTitle.Font = Enum.Font.GothamBold
        targetTitle.TextXAlignment = Enum.TextXAlignment.Left
        targetTitle.Parent = targetHeader

        local targetAvatar = Instance.new("ImageLabel")
        targetAvatar.Name = "Avatar"
        targetAvatar.Size = UDim2.new(0, 44, 0, 44)
        targetAvatar.Position = UDim2.new(0, 10, 0, 34)
        targetAvatar.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        targetAvatar.BorderSizePixel = 0
        targetAvatar.Image = ""
        targetAvatar.Parent = targetCard

        local avatarCorner = Instance.new("UICorner")
        avatarCorner.CornerRadius = UDim.new(0, 7)
        avatarCorner.Parent = targetAvatar

        local avatarStroke = Instance.new("UIStroke")
        avatarStroke.Color = Color3.fromRGB(100, 100, 100)
        avatarStroke.Thickness = 1
        avatarStroke.Transparency = 0.5
        avatarStroke.Parent = targetAvatar

        local targetName = Instance.new("TextLabel")
        targetName.Name = "TargetName"
        targetName.Size = UDim2.new(0, 120, 0, 16)
        targetName.Position = UDim2.new(0, 62, 0, 38)
        targetName.BackgroundTransparency = 1
        targetName.Text = "No Target"
        targetName.TextColor3 = Color3.fromRGB(220, 220, 220)
        targetName.TextSize = 12
        targetName.Font = Enum.Font.GothamBold
        targetName.TextXAlignment = Enum.TextXAlignment.Left
        targetName.Parent = targetCard

        local targetUsername = Instance.new("TextLabel")
        targetUsername.Name = "TargetUsername"
        targetUsername.Size = UDim2.new(0, 120, 0, 14)
        targetUsername.Position = UDim2.new(0, 62, 0, 54)
        targetUsername.BackgroundTransparency = 1
        targetUsername.Text = "Select a player"
        targetUsername.TextColor3 = Color3.fromRGB(150, 150, 150)
        targetUsername.TextSize = 10
        targetUsername.Font = Enum.Font.Gotham
        targetUsername.TextXAlignment = Enum.TextXAlignment.Left
        targetUsername.Parent = targetCard

        local warnDisplay = Instance.new("Frame")
        warnDisplay.Name = "WarnDisplay"
        warnDisplay.Size = UDim2.new(1, -20, 0, 28)
        warnDisplay.Position = UDim2.new(0, 10, 0, 88)
        warnDisplay.BackgroundColor3 = Color3.fromRGB(239, 68, 68)
        warnDisplay.BackgroundTransparency = 0.9
        warnDisplay.BorderSizePixel = 0
        warnDisplay.Parent = targetCard

        local warnCorner = Instance.new("UICorner")
        warnCorner.CornerRadius = UDim.new(0, 5)
        warnCorner.Parent = warnDisplay

        local warnStroke = Instance.new("UIStroke")
        warnStroke.Color = Color3.fromRGB(239, 68, 68)
        warnStroke.Thickness = 1
        warnStroke.Transparency = 0.5
        warnStroke.Parent = warnDisplay

        local warnLabel = Instance.new("TextLabel")
        warnLabel.Size = UDim2.new(0, 80, 1, 0)
        warnLabel.Position = UDim2.new(0, 8, 0, 0)
        warnLabel.BackgroundTransparency = 1
        warnLabel.Text = "Warn Count"
        warnLabel.TextColor3 = Color3.fromRGB(248, 113, 113)
        warnLabel.TextSize = 11
        warnLabel.Font = Enum.Font.GothamBold
        warnLabel.TextXAlignment = Enum.TextXAlignment.Left
        warnLabel.Parent = warnDisplay

        local warnCount = Instance.new("TextLabel")
        warnCount.Name = "WarnCount"
        warnCount.Size = UDim2.new(0, 30, 1, 0)
        warnCount.Position = UDim2.new(1, -38, 0, 0)
        warnCount.BackgroundTransparency = 1
        warnCount.Text = "--"
        warnCount.TextColor3 = Color3.fromRGB(248, 113, 113)
        warnCount.TextSize = 14
        warnCount.Font = Enum.Font.GothamBold
        warnCount.TextXAlignment = Enum.TextXAlignment.Right
        warnCount.Parent = warnDisplay

        local actionsGrid = Instance.new("Frame")
        actionsGrid.Size = UDim2.new(1, -20, 0, 100)
        actionsGrid.Position = UDim2.new(0, 10, 0, 124)
        actionsGrid.BackgroundTransparency = 1
        actionsGrid.Parent = targetCard

        local function createActionButton(name, pos, sizeScale, parent, callback)
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(sizeScale, 0, 0, 28)
            btn.Position = pos
            btn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
            btn.BackgroundTransparency = 0.4
            btn.BorderSizePixel = 0
            btn.Text = name
            btn.TextColor3 = Color3.fromRGB(200, 200, 200)
            btn.TextSize = 11
            btn.Font = Enum.Font.GothamBold
            btn.AutoButtonColor = false
            btn.Parent = parent

            local corner = Instance.new("UICorner")
            corner.CornerRadius = UDim.new(0, 5)
            corner.Parent = btn

            local stroke = Instance.new("UIStroke")
            stroke.Color = Color3.fromRGB(80, 80, 80)
            stroke.Thickness = 1
            stroke.Transparency = 0.3
            stroke.Parent = btn

            connectAndTrack(btn.MouseEnter, function() btn.BackgroundColor3 = Color3.fromRGB(70, 70, 70) end)
            connectAndTrack(btn.MouseLeave, function() btn.BackgroundColor3 = Color3.fromRGB(50, 50, 50) end)
            if callback then connectAndTrack(btn.MouseButton1Click, callback) end
            return btn
        end

        createActionButton("TP", UDim2.new(0.335, 0, 0, 0), 0.33, actionsGrid, function()
            if not target or not target.Character then return end
            local char = player.Character
            if not char then return end
            local hrp = char:FindFirstChild("HumanoidRootPart")
            local targetHrp = target.Character:FindFirstChild("HumanoidRootPart")
            if hrp and targetHrp then hrp.CFrame = targetHrp.CFrame end
        end)

        createActionButton("BRING", UDim2.new(0.67, 0, 0, 0), 0.33, actionsGrid, function()
            local action = "Bring"
            if multiSelectMode then
                local playersToProcess = {}
                for plr in pairs(multiTargets) do if plr and plr.Parent then table.insert(playersToProcess, plr) end end
                for i, plr in ipairs(playersToProcess) do
                    task.spawn(function() fireRemoteAction(action, plr.UserId) end)
                    if i < #playersToProcess then task.wait() end
                end
            elseif target then
                fireRemoteAction(action, target.UserId)
            end
        end)

        createActionButton("RES", UDim2.new(0.5, 0, 0, 32), 0.4950, actionsGrid, function()
            local action = "Refresh"
            if multiSelectMode then
                local playersToProcess = {}
                for plr in pairs(multiTargets) do if plr and plr.Parent then table.insert(playersToProcess, plr) end end
                for i, plr in ipairs(playersToProcess) do
                    task.spawn(function() fireRemoteAction(action, plr.UserId) end)
                    if i < #playersToProcess then task.wait(0.6) end
                end
            elseif target then
                fireRemoteAction(action, target.UserId)
            end
        end)

        -- VIEW BUTTON (toggle)
        local viewBtn = Instance.new("TextButton")
        viewBtn.Name = "ViewBtn"
        viewBtn.Size = UDim2.new(0.33, 0, 0, 28)
        viewBtn.Position = UDim2.new(0, 0, 0, 0)
        viewBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        viewBtn.BackgroundTransparency = 0.4
        viewBtn.BorderSizePixel = 0
        viewBtn.Text = "VIEW"
        viewBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
        viewBtn.TextSize = 11
        viewBtn.Font = Enum.Font.GothamBold
        viewBtn.AutoButtonColor = false
        viewBtn.Parent = actionsGrid

        local viewCorner = Instance.new("UICorner")
        viewCorner.CornerRadius = UDim.new(0, 5)
        viewCorner.Parent = viewBtn

        local viewStroke = Instance.new("UIStroke")
        viewStroke.Color = Color3.fromRGB(80, 80, 80)
        viewStroke.Thickness = 1
        viewStroke.Transparency = 0.3
        viewStroke.Parent = viewBtn

        connectAndTrack(viewBtn.MouseButton1Click, function()
            if not target then return end
            viewing = not viewing
            if viewing then
                viewBtn.BackgroundColor3 = Color3.fromRGB(34, 197, 94)
                updateViewCamera()
                setupViewConnection()
                viewingTarget = target
            else
                camera.CameraSubject = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
                viewBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
                viewingTarget = nil
                if viewCharacterAddedConn then
                    viewCharacterAddedConn:Disconnect()
                    viewCharacterAddedConn = nil
                end
            end
        end)

        connectAndTrack(viewBtn.MouseEnter, function()
            if not viewing then viewBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 70) end
        end)
        connectAndTrack(viewBtn.MouseLeave, function()
            if not viewing then viewBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50) end
        end)

        -- FREEZE BUTTON (toggle)
        local freezeBtn = Instance.new("TextButton")
        freezeBtn.Name = "FreezeBtn"
        freezeBtn.Size = UDim2.new(0.4950, 0, 0, 28)
        freezeBtn.Position = UDim2.new(0, 0, 0, 32)
        freezeBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        freezeBtn.BackgroundTransparency = 0.4
        freezeBtn.BorderSizePixel = 0
        freezeBtn.Text = "FREEZE"
        freezeBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
        freezeBtn.TextSize = 11
        freezeBtn.Font = Enum.Font.GothamBold
        freezeBtn.AutoButtonColor = false
        freezeBtn.Parent = actionsGrid

        local freezeCorner = Instance.new("UICorner")
        freezeCorner.CornerRadius = UDim.new(0, 5)
        freezeCorner.Parent = freezeBtn

        local freezeStroke = Instance.new("UIStroke")
        freezeStroke.Color = Color3.fromRGB(80, 80, 80)
        freezeStroke.Thickness = 1
        freezeStroke.Transparency = 0.3
        freezeStroke.Parent = freezeBtn

        connectAndTrack(freezeBtn.MouseButton1Click, function()
            frozen = not frozen
            local action = frozen and "Freeze" or "Unfreeze"
            if multiSelectMode then
                local playersToProcess = {}
                for plr in pairs(multiTargets) do if plr and plr.Parent then table.insert(playersToProcess, plr) end end
                for i, plr in ipairs(playersToProcess) do
                    task.spawn(function() fireRemoteAction(action, plr.UserId) end)
                    if i < #playersToProcess then task.wait(0.6) end
                end
            elseif target then
                fireRemoteAction(action, target.UserId)
            end
            freezeBtn.BackgroundColor3 = frozen and Color3.fromRGB(34, 197, 94) or Color3.fromRGB(50, 50, 50)
        end)

        connectAndTrack(freezeBtn.MouseEnter, function()
            if not frozen then freezeBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 70) end
        end)
        connectAndTrack(freezeBtn.MouseLeave, function()
            if not frozen then freezeBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50) end
        end)

        -- ========== BOTÕES AUTO CHEST E MUTE ==========
        local chestBtn = Instance.new("TextButton")
        chestBtn.Name = "ChestBtn"
        chestBtn.Size = UDim2.new(0.4950, 0, 0, 28)
        chestBtn.Position = UDim2.new(0, 0, 0, 64)
        chestBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        chestBtn.BackgroundTransparency = 0.4
        chestBtn.BorderSizePixel = 0
        chestBtn.Text = "AUTO CHEST"
        chestBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
        chestBtn.TextSize = 11
        chestBtn.Font = Enum.Font.GothamBold
        chestBtn.AutoButtonColor = false
        chestBtn.Parent = actionsGrid

        local chestCorner = Instance.new("UICorner")
        chestCorner.CornerRadius = UDim.new(0, 5)
        chestCorner.Parent = chestBtn

        local chestStroke = Instance.new("UIStroke")
        chestStroke.Color = Color3.fromRGB(80, 80, 80)
        chestStroke.Thickness = 1
        chestStroke.Transparency = 0.3
        chestStroke.Parent = chestBtn

        connectAndTrack(chestBtn.MouseButton1Click, function()
            if chestAutoEnabled then
                stopChestAuto()
                chestBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
            else
                startChestAuto()
                chestBtn.BackgroundColor3 = Color3.fromRGB(34, 197, 94)
            end
        end)

        connectAndTrack(chestBtn.MouseEnter, function()
            if not chestAutoEnabled then chestBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 70) end
        end)
        connectAndTrack(chestBtn.MouseLeave, function()
            if not chestAutoEnabled then chestBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50) end
        end)

        local muteBtnLocal = Instance.new("TextButton")
        muteBtnLocal.Name = "MuteBtn"
        muteBtnLocal.Size = UDim2.new(0.4950, 0, 0, 28)
        muteBtnLocal.Position = UDim2.new(0.5, 0, 0, 64)
        muteBtnLocal.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        muteBtnLocal.BackgroundTransparency = 0.4
        muteBtnLocal.BorderSizePixel = 0
        muteBtnLocal.Text = "MUTE"
        muteBtnLocal.TextColor3 = Color3.fromRGB(200, 200, 200)
        muteBtnLocal.TextSize = 11
        muteBtnLocal.Font = Enum.Font.GothamBold
        muteBtnLocal.AutoButtonColor = false
        muteBtnLocal.Parent = actionsGrid

        local muteCorner = Instance.new("UICorner")
        muteCorner.CornerRadius = UDim.new(0, 5)
        muteCorner.Parent = muteBtnLocal

        local muteStroke = Instance.new("UIStroke")
        muteStroke.Color = Color3.fromRGB(80, 80, 80)
        muteStroke.Thickness = 1
        muteStroke.Transparency = 0.3
        muteStroke.Parent = muteBtnLocal

        muteBtn = muteBtnLocal

        connectAndTrack(muteBtnLocal.MouseButton1Click, function()
            if not target then return end
            targetMuted = not targetMuted
            local action = targetMuted and "Mute" or "Unmute"
            fireRemoteAction(action, target.UserId)
            muteBtnLocal.BackgroundColor3 = targetMuted and Color3.fromRGB(34, 197, 94) or Color3.fromRGB(50, 50, 50)
        end)

        connectAndTrack(muteBtnLocal.MouseEnter, function()
            if not targetMuted then muteBtnLocal.BackgroundColor3 = Color3.fromRGB(70, 70, 70) end
        end)
        connectAndTrack(muteBtnLocal.MouseLeave, function()
            if not targetMuted then muteBtnLocal.BackgroundColor3 = Color3.fromRGB(50, 50, 50) end
        end)

        -- SETTINGS CARD
        local settingsCard = Instance.new("Frame")
        settingsCard.Name = "SettingsCard"
        settingsCard.Size = UDim2.new(1, 0, 0, 122)
        settingsCard.Position = UDim2.new(0, 0, 0, 230)
        settingsCard.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
        settingsCard.BackgroundTransparency = 0.3
        settingsCard.BorderSizePixel = 0
        settingsCard.Parent = leftPanel

        local settingsCorner = Instance.new("UICorner")
        settingsCorner.CornerRadius = UDim.new(0, 8)
        settingsCorner.Parent = settingsCard

        local settingsStroke = Instance.new("UIStroke")
        settingsStroke.Color = Color3.fromRGB(70, 70, 70)
        settingsStroke.Thickness = 1
        settingsStroke.Transparency = 0.3
        settingsStroke.Parent = settingsCard

        local settingsHeader = Instance.new("Frame")
        settingsHeader.Size = UDim2.new(1, -14, 0, 20)
        settingsHeader.Position = UDim2.new(0, 7, 0, 2)
        settingsHeader.BackgroundTransparency = 1
        settingsHeader.Parent = settingsCard

        local settingsTitle = Instance.new("TextLabel")
        settingsTitle.Size = UDim2.new(1, 0, 1, 0)
        settingsTitle.BackgroundTransparency = 1
        settingsTitle.Text = " QUICK SETTINGS"
        settingsTitle.TextColor3 = Color3.fromRGB(220, 220, 220)
        settingsTitle.TextSize = 11
        settingsTitle.Font = Enum.Font.GothamBold
        settingsTitle.TextXAlignment = Enum.TextXAlignment.Left
        settingsTitle.Parent = settingsHeader

        local function createToggle(name, yPos, parent, callback)
            local toggleFrame = Instance.new("Frame")
            toggleFrame.Size = UDim2.new(1, -14, 0, 24)
            toggleFrame.Position = UDim2.new(0, 7, 0, yPos)
            toggleFrame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
            toggleFrame.BackgroundTransparency = 0.4
            toggleFrame.BorderSizePixel = 0
            toggleFrame.Parent = parent

            local corner = Instance.new("UICorner")
            corner.CornerRadius = UDim.new(0, 5)
            corner.Parent = toggleFrame

            local stroke = Instance.new("UIStroke")
            stroke.Color = Color3.fromRGB(80, 80, 80)
            stroke.Thickness = 1
            stroke.Transparency = 0.3
            stroke.Parent = toggleFrame

            local textLabel = Instance.new("TextLabel")
            textLabel.Size = UDim2.new(0, 100, 1, 0)
            textLabel.Position = UDim2.new(0, 8, 0, 0)
            textLabel.BackgroundTransparency = 1
            textLabel.Text = name
            textLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
            textLabel.TextSize = 11
            textLabel.Font = Enum.Font.Gotham
            textLabel.TextXAlignment = Enum.TextXAlignment.Left
            textLabel.Parent = toggleFrame

            local toggleButton = Instance.new("TextButton")
            toggleButton.Size = UDim2.new(0, 30, 0, 18)
            toggleButton.Position = UDim2.new(1, -36, 0.5, -9)
            toggleButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
            toggleButton.BorderSizePixel = 0
            toggleButton.Text = ""
            toggleButton.Parent = toggleFrame

            local toggleCorner = Instance.new("UICorner")
            toggleCorner.CornerRadius = UDim.new(1, 0)
            toggleCorner.Parent = toggleButton

            local toggleKnob = Instance.new("Frame")
            toggleKnob.Size = UDim2.new(0, 12, 0, 12)
            toggleKnob.Position = UDim2.new(0, 3, 0.5, -6)
            toggleKnob.BackgroundColor3 = Color3.fromRGB(220, 220, 220)
            toggleKnob.BorderSizePixel = 0
            toggleKnob.Parent = toggleButton

            local knobCorner = Instance.new("UICorner")
            knobCorner.CornerRadius = UDim.new(1, 0)
            knobCorner.Parent = toggleKnob

            local enabled = false
            connectAndTrack(toggleButton.MouseButton1Click, function()
                enabled = not enabled
                if callback then callback(enabled) end
                if enabled then
                    toggleButton.BackgroundColor3 = Color3.fromRGB(34, 197, 94)
                    toggleKnob:TweenPosition(UDim2.new(1, -15, 0.5, -6), Enum.EasingDirection.Out, Enum.EasingStyle.Quad, 0.2, true)
                else
                    toggleButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
                    toggleKnob:TweenPosition(UDim2.new(0, 3, 0.5, -6), Enum.EasingDirection.Out, Enum.EasingStyle.Quad, 0.2, true)
                end
            end)

            return toggleButton, toggleFrame
        end

        createToggle("Water Walk", 24, settingsCard, function(enabled)
            waterWalkEnabled = enabled
            updateWalkWater()
        end)

        createToggle("Korblox/Headless", 52, settingsCard, function(enabled)
            khEnabled = enabled
            if enabled and player.Character then enviarComandos() end
        end)

        local dualButtonFrame = Instance.new("Frame")
        dualButtonFrame.Size = UDim2.new(1, -14, 0, 28)
        dualButtonFrame.Position = UDim2.new(0, 7, 0, 82)
        dualButtonFrame.BackgroundTransparency = 1
        dualButtonFrame.Parent = settingsCard

        local rejoinBtn = Instance.new("TextButton")
        rejoinBtn.Name = "RejoinBtn"
        rejoinBtn.Size = UDim2.new(0.5, -3, 1, 0)
        rejoinBtn.Position = UDim2.new(0, 0, 0, 0)
        rejoinBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        rejoinBtn.BackgroundTransparency = 0.4
        rejoinBtn.BorderSizePixel = 0
        rejoinBtn.Text = "REJOIN"
        rejoinBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
        rejoinBtn.TextSize = 11
        rejoinBtn.Font = Enum.Font.GothamBold
        rejoinBtn.AutoButtonColor = false
        rejoinBtn.Parent = dualButtonFrame

        local rejoinCorner = Instance.new("UICorner")
        rejoinCorner.CornerRadius = UDim.new(0, 5)
        rejoinCorner.Parent = rejoinBtn

        local rejoinStroke = Instance.new("UIStroke")
        rejoinStroke.Color = Color3.fromRGB(80, 80, 80)
        rejoinStroke.Thickness = 1
        rejoinStroke.Transparency = 0.3
        rejoinStroke.Parent = rejoinBtn

        local antiCarryBtn = Instance.new("TextButton")
        antiCarryBtn.Name = "AntiCarryBtn"
        antiCarryBtn.Size = UDim2.new(0.5, -3, 1, 0)
        antiCarryBtn.Position = UDim2.new(0.5, 3, 0, 0)
        antiCarryBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        antiCarryBtn.BackgroundTransparency = 0.4
        antiCarryBtn.BorderSizePixel = 0
        antiCarryBtn.Text = "ANTI CARRY"
        antiCarryBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
        antiCarryBtn.TextSize = 10
        antiCarryBtn.Font = Enum.Font.GothamBold
        antiCarryBtn.AutoButtonColor = false
        antiCarryBtn.Parent = dualButtonFrame

        local antiCarryCorner = Instance.new("UICorner")
        antiCarryCorner.CornerRadius = UDim.new(0, 5)
        antiCarryCorner.Parent = antiCarryBtn

        local antiCarryStroke = Instance.new("UIStroke")
        antiCarryStroke.Color = Color3.fromRGB(80, 80, 80)
        antiCarryStroke.Thickness = 1
        antiCarryStroke.Transparency = 0.3
        antiCarryStroke.Parent = antiCarryBtn

        connectAndTrack(antiCarryBtn.MouseButton1Click, function()
            antiCarryEnabled = not antiCarryEnabled
            if antiCarryEnabled then
                antiCarryBtn.BackgroundColor3 = Color3.fromRGB(34, 197, 94)
                startAntiCarry()
            else
                antiCarryBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
                stopAntiCarry()
            end
        end)
        connectAndTrack(antiCarryBtn.MouseEnter, function()
            if not antiCarryEnabled then antiCarryBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 70) end
        end)
        connectAndTrack(antiCarryBtn.MouseLeave, function()
            if not antiCarryEnabled then antiCarryBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50) end
        end)

        connectAndTrack(rejoinBtn.MouseEnter, function() rejoinBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 70) end)
        connectAndTrack(rejoinBtn.MouseLeave, function() rejoinBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50) end)

        -- Confirm dialog (rejoin)
        local confirmGui = Instance.new("ScreenGui")
        confirmGui.Name = "RejoinConfirmGui"
        confirmGui.ResetOnSpawn = false
        confirmGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        confirmGui.Parent = playerGui

        local confirmFrame = Instance.new("Frame")
        confirmFrame.Name = "ConfirmFrame"
        confirmFrame.Size = UDim2.new(0, 260, 0, 110)
        confirmFrame.Position = UDim2.new(1, -280, 1, -130)
        confirmFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
        confirmFrame.BackgroundTransparency = 0.15
        confirmFrame.BorderSizePixel = 0
        confirmFrame.Active = true
        confirmFrame.Visible = false
        confirmFrame.Parent = confirmGui

        local confirmCorner = Instance.new("UICorner")
        confirmCorner.CornerRadius = UDim.new(0, 12)
        confirmCorner.Parent = confirmFrame

        local confirmStroke = Instance.new("UIStroke")
        confirmStroke.Color = Color3.fromRGB(80, 80, 80)
        confirmStroke.Thickness = 1.5
        confirmStroke.Transparency = 0.4
        confirmStroke.Parent = confirmFrame

        local outsideCloseBtn = Instance.new("TextButton")
        outsideCloseBtn.Size = UDim2.new(1, 0, 1, 0)
        outsideCloseBtn.BackgroundTransparency = 1
        outsideCloseBtn.Text = ""
        outsideCloseBtn.Parent = confirmFrame

        local confirmText = Instance.new("TextLabel")
        confirmText.Size = UDim2.new(1, -50, 0, 20)
        confirmText.Position = UDim2.new(0, 20, 0, 18)
        confirmText.BackgroundTransparency = 1
        confirmText.Text = "Gostaria de relogar?"
        confirmText.TextColor3 = Color3.fromRGB(255, 255, 255)
        confirmText.TextSize = 14
        confirmText.Font = Enum.Font.GothamBold
        confirmText.TextXAlignment = Enum.TextXAlignment.Left
        confirmText.Parent = confirmFrame

        local yesBtn = Instance.new("TextButton")
        yesBtn.Size = UDim2.new(0, 100, 0, 30)
        yesBtn.Position = UDim2.new(0, 20, 0, 60)
        yesBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        yesBtn.BackgroundTransparency = 0.3
        yesBtn.BorderSizePixel = 0
        yesBtn.Text = "Sim"
        yesBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        yesBtn.TextSize = 13
        yesBtn.Font = Enum.Font.GothamBold
        yesBtn.AutoButtonColor = false
        yesBtn.Parent = confirmFrame

        local yesCorner = Instance.new("UICorner")
        yesCorner.CornerRadius = UDim.new(0, 6)
        yesCorner.Parent = yesBtn

        local noBtn = Instance.new("TextButton")
        noBtn.Size = UDim2.new(0, 100, 0, 30)
        noBtn.Position = UDim2.new(0, 140, 0, 60)
        noBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        noBtn.BackgroundTransparency = 0.3
        noBtn.BorderSizePixel = 0
        noBtn.Text = "Não"
        noBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        noBtn.TextSize = 13
        noBtn.Font = Enum.Font.GothamBold
        noBtn.AutoButtonColor = false
        noBtn.Parent = confirmFrame

        local noCorner = Instance.new("UICorner")
        noCorner.CornerRadius = UDim.new(0, 6)
        noCorner.Parent = noBtn

        local confirmTweenInfo = TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
        local confirmFadeIn = TweenService:Create(confirmFrame, confirmTweenInfo, {BackgroundTransparency = 0.15})
        local confirmFadeOut = TweenService:Create(confirmFrame, confirmTweenInfo, {BackgroundTransparency = 1.0})

        local function hideConfirm()
            confirmFadeOut:Play()
            task.delay(0.5, function() confirmFrame.Visible = false end)
        end
        local function showRejoinConfirm()
            if confirmFrame.Visible then return end
            confirmFrame.Visible = true
            confirmFrame.BackgroundTransparency = 1.0
            confirmFadeIn:Play()
        end
        connectAndTrack(rejoinBtn.MouseButton1Click, showRejoinConfirm)
        connectAndTrack(yesBtn.MouseButton1Click, function()
            hideConfirm()
            pcall(function() TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, player) end)
        end)
        connectAndTrack(noBtn.MouseButton1Click, hideConfirm)
        connectAndTrack(outsideCloseBtn.MouseButton1Click, hideConfirm)

        connectAndTrack(yesBtn.MouseEnter, function()
            yesBtn.BackgroundColor3 = Color3.fromRGB(5, 5, 5)
            yesBtn.BackgroundTransparency = 0.1
        end)
        connectAndTrack(yesBtn.MouseLeave, function()
            yesBtn.BackgroundColor3 = Color3.fromRGB(5, 5, 5)
            yesBtn.BackgroundTransparency = 0.3
        end)
        connectAndTrack(noBtn.MouseEnter, function()
            noBtn.BackgroundColor3 = Color3.fromRGB(5, 5, 5)
            noBtn.BackgroundTransparency = 0.1
        end)
        connectAndTrack(noBtn.MouseLeave, function()
            noBtn.BackgroundColor3 = Color3.fromRGB(5, 5, 5)
            noBtn.BackgroundTransparency = 0.3
        end)

        -- Right panel (lista de jogadores)
        local rightPanel = Instance.new("Frame")
        rightPanel.Name = "RightPanel"
        rightPanel.Size = UDim2.new(1, -198, 1, 0)
        rightPanel.Position = UDim2.new(0, 196, 0, 0)
        rightPanel.BackgroundTransparency = 1
        rightPanel.Parent = contentArea

        local playerListCard = Instance.new("Frame")
        playerListCard.Name = "PlayerListCard"
        playerListCard.Size = UDim2.new(1, 0, 1, 0)
        playerListCard.Position = UDim2.new(0, 0, 0, 0)
        playerListCard.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
        playerListCard.BackgroundTransparency = 0.3
        playerListCard.BorderSizePixel = 0
        playerListCard.Parent = rightPanel

        local playerListCorner = Instance.new("UICorner")
        playerListCorner.CornerRadius = UDim.new(0, 8)
        playerListCorner.Parent = playerListCard

        local playerListStroke = Instance.new("UIStroke")
        playerListStroke.Color = Color3.fromRGB(70, 70, 70)
        playerListStroke.Thickness = 1
        playerListStroke.Transparency = 0.3
        playerListStroke.Parent = playerListCard

        local playerListHeader = Instance.new("Frame")
        playerListHeader.Size = UDim2.new(1, 0, 0, 68)
        playerListHeader.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
        playerListHeader.BackgroundTransparency = 0.3
        playerListHeader.BorderSizePixel = 0
        playerListHeader.Parent = playerListCard

        local playerListHeaderCorner = Instance.new("UICorner")
        playerListHeaderCorner.CornerRadius = UDim.new(0, 8)
        playerListHeaderCorner.Parent = playerListHeader

        local playerListHeaderBottom = Instance.new("Frame")
        playerListHeaderBottom.Size = UDim2.new(1, 0, 0, 8)
        playerListHeaderBottom.Position = UDim2.new(0, 0, 1, -8)
        playerListHeaderBottom.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
        playerListHeaderBottom.BackgroundTransparency = 0.3
        playerListHeaderBottom.BorderSizePixel = 0
        playerListHeaderBottom.Parent = playerListHeader

        local playerListHeaderStroke = Instance.new("UIStroke")
        playerListHeaderStroke.Color = Color3.fromRGB(70, 70, 70)
        playerListHeaderStroke.Thickness = 1
        playerListHeaderStroke.Transparency = 0.3
        playerListHeaderStroke.Parent = playerListHeader

        local headerTitleArea = Instance.new("Frame")
        headerTitleArea.Size = UDim2.new(1, -14, 0, 20)
        headerTitleArea.Position = UDim2.new(0, 7, 0, 7)
        headerTitleArea.BackgroundTransparency = 1
        headerTitleArea.Parent = playerListHeader

        local playersTitle = Instance.new("TextLabel")
        playersTitle.Size = UDim2.new(0, 78, 1, 0)
        playersTitle.Position = UDim2.new(0, 0, 0, 0)
        playersTitle.BackgroundTransparency = 1
        playersTitle.Text = "Players Online"
        playersTitle.TextColor3 = Color3.fromRGB(220, 220, 220)
        playersTitle.TextSize = 11
        playersTitle.Font = Enum.Font.GothamBold
        playersTitle.TextXAlignment = Enum.TextXAlignment.Left
        playersTitle.Parent = headerTitleArea

        local playerCountBadge = Instance.new("Frame")
        playerCountBadge.Size = UDim2.new(0, 22, 0, 16)
        playerCountBadge.Position = UDim2.new(0, 80, 0, 2)
        playerCountBadge.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
        playerCountBadge.BackgroundTransparency = 0.6
        playerCountBadge.BorderSizePixel = 0
        playerCountBadge.Parent = headerTitleArea

        local badgeCorner = Instance.new("UICorner")
        badgeCorner.CornerRadius = UDim.new(1, 0)
        badgeCorner.Parent = playerCountBadge

        local playerCountLabel = Instance.new("TextLabel")
        playerCountLabel.Name = "PlayerCount"
        playerCountLabel.Size = UDim2.new(1, 0, 1, 0)
        playerCountLabel.BackgroundTransparency = 1
        playerCountLabel.Text = tostring(#Players:GetPlayers())
        playerCountLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
        playerCountLabel.TextSize = 9
        playerCountLabel.Font = Enum.Font.GothamBold
        playerCountLabel.Parent = playerCountBadge

        local selectedCountLabel = Instance.new("TextLabel")
        selectedCountLabel.Name = "SelectedCount"
        selectedCountLabel.Size = UDim2.new(0, 32, 0, 16)
        selectedCountLabel.Position = UDim2.new(0, 106, 0, 2)
        selectedCountLabel.BackgroundTransparency = 1
        selectedCountLabel.Text = ""
        selectedCountLabel.TextColor3 = Color3.fromRGB(168, 85, 247)
        selectedCountLabel.TextSize = 9
        selectedCountLabel.Font = Enum.Font.GothamBold
        selectedCountLabel.Visible = false
        selectedCountLabel.Parent = headerTitleArea

        local clearMultiBtn = Instance.new("TextButton")
        clearMultiBtn.Name = "ClearMultiBtn"
        clearMultiBtn.Size = UDim2.new(0, 18, 0, 18)
        clearMultiBtn.Position = UDim2.new(1, -20, 0, 1)
        clearMultiBtn.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
        clearMultiBtn.BackgroundTransparency = 0.3
        clearMultiBtn.BorderSizePixel = 0
        clearMultiBtn.Text = "🗑️"
        clearMultiBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        clearMultiBtn.TextSize = 10
        clearMultiBtn.Font = Enum.Font.GothamBold
        clearMultiBtn.AutoButtonColor = false
        clearMultiBtn.Visible = false
        clearMultiBtn.ZIndex = 5
        clearMultiBtn.Parent = headerTitleArea

        local clearCorner = Instance.new("UICorner")
        clearCorner.CornerRadius = UDim.new(0, 3)
        clearCorner.Parent = clearMultiBtn

        local multiSelectBtn = Instance.new("TextButton")
        multiSelectBtn.Name = "MultiSelectBtn"
        multiSelectBtn.Size = UDim2.new(0, 76, 0, 20)
        multiSelectBtn.Position = UDim2.new(1, -102, 0, 0)
        multiSelectBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        multiSelectBtn.BackgroundTransparency = 0.4
        multiSelectBtn.BorderSizePixel = 0
        multiSelectBtn.Text = "Multi Select"
        multiSelectBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
        multiSelectBtn.TextSize = 9
        multiSelectBtn.Font = Enum.Font.GothamBold
        multiSelectBtn.Parent = headerTitleArea

        local multiSelectCorner = Instance.new("UICorner")
        multiSelectCorner.CornerRadius = UDim.new(0, 5)
        multiSelectCorner.Parent = multiSelectBtn

        local searchBox = Instance.new("TextBox")
        searchBox.Name = "SearchBox"
        searchBox.Size = UDim2.new(1, -14, 0, 26)
        searchBox.Position = UDim2.new(0, 7, 0, 34)
        searchBox.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
        searchBox.BackgroundTransparency = 0.3
        searchBox.BorderSizePixel = 0
        searchBox.PlaceholderText = "Search players..."
        searchBox.PlaceholderColor3 = Color3.fromRGB(120, 120, 120)
        searchBox.Text = ""
        searchBox.TextColor3 = Color3.fromRGB(220, 220, 220)
        searchBox.TextSize = 10
        searchBox.Font = Enum.Font.Gotham
        searchBox.ClearTextOnFocus = false
        searchBox.TextXAlignment = Enum.TextXAlignment.Left
        searchBox.Parent = playerListHeader

        local searchCorner = Instance.new("UICorner")
        searchCorner.CornerRadius = UDim.new(0, 5)
        searchCorner.Parent = searchBox

        local searchStroke = Instance.new("UIStroke")
        searchStroke.Color = Color3.fromRGB(70, 70, 70)
        searchStroke.Thickness = 1
        searchStroke.Transparency = 0.3
        searchStroke.Parent = searchBox

        local searchPadding = Instance.new("UIPadding")
        searchPadding.PaddingLeft = UDim.new(0, 7)
        searchPadding.PaddingRight = UDim.new(0, 7)
        searchPadding.Parent = searchBox

        local playerListScroll = Instance.new("ScrollingFrame")
        playerListScroll.Name = "PlayerListScroll"
        playerListScroll.Size = UDim2.new(1, -7, 1, -74)
        playerListScroll.Position = UDim2.new(0, 3, 0, 72)
        playerListScroll.BackgroundTransparency = 1
        playerListScroll.BorderSizePixel = 0
        playerListScroll.ScrollBarThickness = 4
        playerListScroll.ScrollBarImageColor3 = Color3.fromRGB(100, 100, 100)
        playerListScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
        playerListScroll.Parent = playerListCard

        local playerListLayout = Instance.new("UIListLayout")
        playerListLayout.Padding = UDim.new(0, 5)
        playerListLayout.SortOrder = Enum.SortOrder.Name
        playerListLayout.Parent = playerListScroll

        local playerListPadding = Instance.new("UIPadding")
        playerListPadding.PaddingLeft = UDim.new(0, 3)
        playerListPadding.PaddingRight = UDim.new(0, 3)
        playerListPadding.PaddingTop = UDim.new(0, 3)
        playerListPadding.PaddingBottom = UDim.new(0, 3)
        playerListPadding.Parent = playerListScroll

        local function updateCanvasSize()
            playerListScroll.CanvasSize = UDim2.new(0, 0, 0, playerListLayout.AbsoluteContentSize.Y + 8)
        end
        connectAndTrack(playerListLayout:GetPropertyChangedSignal("AbsoluteContentSize"), updateCanvasSize)

        local function calculateCreationDate(accountAgeDays)
            local currentTime = os.time()
            local estimatedCreationTime = currentTime - (accountAgeDays * 86400)
            local dateTable = os.date("*t", estimatedCreationTime)
            return string.format("%02d/%02d/%02d", dateTable.day, dateTable.month, dateTable.year % 100)
        end

        local function createPlayerCard(targetPlayer)
            local isSelf = (targetPlayer == player)
            local card = Instance.new("TextButton")
            card.Name = "PlayerCard_" .. targetPlayer.UserId
            card.Size = UDim2.new(1, -7, 0, 44)
            card.BackgroundColor3 = isSelf and Color3.fromRGB(60, 60, 80) or Color3.fromRGB(50, 50, 50)
            card.BackgroundTransparency = 0.4
            card.BorderSizePixel = 0
            card.Text = ""
            card.AutoButtonColor = false
            card.Parent = playerListScroll

            local cardCorner = Instance.new("UICorner")
            cardCorner.CornerRadius = UDim.new(0, 7)
            cardCorner.Parent = card

            local cardStroke = Instance.new("UIStroke")
            cardStroke.Name = "CardStroke"
            cardStroke.Color = isSelf and Color3.fromRGB(100, 100, 150) or Color3.fromRGB(80, 80, 80)
            cardStroke.Thickness = 1
            cardStroke.Transparency = 0.3
            cardStroke.Parent = card

            local avatar = Instance.new("ImageLabel")
            avatar.Size = UDim2.new(0, 32, 0, 32)
            avatar.Position = UDim2.new(0, 6, 0.5, -16)
            avatar.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
            avatar.BorderSizePixel = 0
            avatar.Image = "https://www.roblox.com/headshot-thumbnail/image?userId=" .. targetPlayer.UserId .. "&width=48&height=48&format=png"
            avatar.Parent = card

            local avatarCorner = Instance.new("UICorner")
            avatarCorner.CornerRadius = UDim.new(0, 5)
            avatarCorner.Parent = avatar

            local avatarStroke = Instance.new("UIStroke")
            avatarStroke.Color = Color3.fromRGB(80, 80, 80)
            avatarStroke.Thickness = 1
            avatarStroke.Transparency = 0.3
            avatarStroke.Parent = avatar

            local displayName = Instance.new("TextLabel")
            displayName.Size = UDim2.new(0, 150, 0, 13)
            displayName.Position = UDim2.new(0, 44, 0, 8)
            displayName.BackgroundTransparency = 1
            displayName.Text = targetPlayer.DisplayName
            displayName.TextColor3 = isSelf and Color3.fromRGB(180, 180, 255) or Color3.fromRGB(220, 220, 220)
            displayName.TextSize = 10
            displayName.Font = Enum.Font.GothamBold
            displayName.TextXAlignment = Enum.TextXAlignment.Left
            displayName.TextTruncate = Enum.TextTruncate.AtEnd
            displayName.Parent = card

            local username = Instance.new("TextLabel")
            username.Size = UDim2.new(0, 150, 0, 12)
            username.Position = UDim2.new(0, 44, 0, 23)
            username.BackgroundTransparency = 1
            username.Text = "@" .. targetPlayer.Name .. (isSelf and " (You)" or "")
            username.TextColor3 = isSelf and Color3.fromRGB(150, 150, 220) or Color3.fromRGB(150, 150, 150)
            username.TextSize = 11
            username.Font = Enum.Font.Gotham
            username.TextXAlignment = Enum.TextXAlignment.Left
            username.TextTruncate = Enum.TextTruncate.AtEnd
            username.Parent = card

            local creationDate = calculateCreationDate(targetPlayer.AccountAge)
            local dateLabel = Instance.new("TextLabel")
            dateLabel.Name = "CreationDate"
            dateLabel.Size = UDim2.new(0, 45, 0, 12)
            dateLabel.Position = UDim2.new(1, -110, 0, 16)
            dateLabel.BackgroundTransparency = 1
            dateLabel.Text = creationDate
            dateLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
            dateLabel.TextSize = 10
            dateLabel.Font = Enum.Font.Gotham
            dateLabel.TextXAlignment = Enum.TextXAlignment.Right
            dateLabel.Parent = card

            local warnBadge = Instance.new("Frame")
            warnBadge.Size = UDim2.new(0, 44, 0, 16)
            warnBadge.Position = UDim2.new(1, -54, 0.5, -8)
            warnBadge.BackgroundColor3 = Color3.fromRGB(34, 197, 94)
            warnBadge.BackgroundTransparency = 0.85
            warnBadge.BorderSizePixel = 0
            warnBadge.Parent = card

            local warnBadgeCorner = Instance.new("UICorner")
            warnBadgeCorner.CornerRadius = UDim.new(0, 5)
            warnBadgeCorner.Parent = warnBadge

            local warnBadgeStroke = Instance.new("UIStroke")
            warnBadgeStroke.Color = Color3.fromRGB(34, 197, 94)
            warnBadgeStroke.Thickness = 1
            warnBadgeStroke.Transparency = 0.5
            warnBadgeStroke.Parent = warnBadge

            local warnText = Instance.new("TextLabel")
            warnText.Size = UDim2.new(1, 0, 1, 0)
            warnText.BackgroundTransparency = 1
            warnText.Text = "0 warns"
            warnText.TextColor3 = Color3.fromRGB(74, 222, 128)
            warnText.TextSize = 8
            warnText.Font = Enum.Font.GothamBold
            warnText.Parent = warnBadge

            local checkmark = Instance.new("Frame")
            checkmark.Name = "Checkmark"
            checkmark.Size = UDim2.new(0, 12, 0, 12)
            checkmark.Position = UDim2.new(0, -3, 0, -3)
            checkmark.BackgroundColor3 = Color3.fromRGB(168, 85, 247)
            checkmark.BorderSizePixel = 0
            checkmark.Visible = false
            checkmark.ZIndex = 10
            checkmark.Parent = avatar

            local checkmarkCorner = Instance.new("UICorner")
            checkmarkCorner.CornerRadius = UDim.new(1, 0)
            checkmarkCorner.Parent = checkmark

            local checkmarkIcon = Instance.new("TextLabel")
            checkmarkIcon.Size = UDim2.new(1, 0, 1, 0)
            checkmarkIcon.BackgroundTransparency = 1
            checkmarkIcon.Text = "✓"
            checkmarkIcon.TextColor3 = Color3.fromRGB(255, 255, 255)
            checkmarkIcon.TextSize = 9
            checkmarkIcon.Font = Enum.Font.GothamBold
            checkmarkIcon.Parent = checkmark

            local function updateWarnCount()
                local warnObj = findWarnObject(targetPlayer)
                local count = 0
                if warnObj then
                    if warnObj.type == "attribute" then count = warnObj.value
                    elseif warnObj:IsA("IntValue") or warnObj:IsA("NumberValue") then count = warnObj.Value
                    elseif warnObj:IsA("StringValue") then count = tonumber(warnObj.Value) or 0 end
                end
                warnText.Text = count .. (count == 1 and " warn" or " warns")
                if count == 0 then
                    warnBadge.BackgroundColor3 = Color3.fromRGB(34, 197, 94)
                    warnBadgeStroke.Color = Color3.fromRGB(34, 197, 94)
                    warnText.TextColor3 = Color3.fromRGB(74, 222, 128)
                elseif count <= 3 then
                    warnBadge.BackgroundColor3 = Color3.fromRGB(249, 115, 22)
                    warnBadgeStroke.Color = Color3.fromRGB(249, 115, 22)
                    warnText.TextColor3 = Color3.fromRGB(253, 186, 116)
                else
                    warnBadge.BackgroundColor3 = Color3.fromRGB(168, 85, 247)
                    warnBadgeStroke.Color = Color3.fromRGB(168, 85, 247)
                    warnText.TextColor3 = Color3.fromRGB(216, 180, 254)
                end
            end
            updateWarnCount()

            connectAndTrack(card.MouseButton1Click, function()
                if multiSelectMode then
                    if multiTargets[targetPlayer] then
                        multiTargets[targetPlayer] = nil
                        checkmark.Visible = false
                        card.BackgroundColor3 = isSelf and Color3.fromRGB(60, 60, 80) or Color3.fromRGB(50, 50, 50)
                        cardStroke.Color = isSelf and Color3.fromRGB(100, 100, 150) or Color3.fromRGB(80, 80, 80)
                    else
                        multiTargets[targetPlayer] = true
                        checkmark.Visible = true
                        card.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
                        cardStroke.Color = Color3.fromRGB(168, 85, 247)
                    end
                    local count = 0
                    for _ in pairs(multiTargets) do count = count + 1 end
                    selectedCountLabel.Text = count > 0 and (count .. " sel.") or ""
                else
                    target = targetPlayer
                    targetMuted = false
                    if muteBtn then
                        muteBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
                    end

                    -- Atualiza o view se estiver ativo
                    if viewing then
                        updateViewCamera()
                        setupViewConnection()
                    end

                    for _, otherCard in ipairs(playerListScroll:GetChildren()) do
                        if otherCard:IsA("TextButton") and otherCard.Name:match("^PlayerCard_") then
                            local otherIsSelf = (otherCard.Name == "PlayerCard_" .. player.UserId)
                            otherCard.BackgroundColor3 = otherIsSelf and Color3.fromRGB(60, 60, 80) or Color3.fromRGB(50, 50, 50)
                            local otherStroke = otherCard:FindFirstChild("CardStroke")
                            if otherStroke then otherStroke.Color = otherIsSelf and Color3.fromRGB(100, 100, 150) or Color3.fromRGB(80, 80, 80) end
                        end
                    end
                    card.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
                    cardStroke.Color = Color3.fromRGB(168, 85, 247)
                    targetName.Text = targetPlayer.DisplayName
                    targetUsername.Text = "@" .. targetPlayer.Name
                    targetAvatar.Image = "https://www.roblox.com/headshot-thumbnail/image?userId=" .. targetPlayer.UserId .. "&width=64&height=64&format=png"
                    local warnObj = findWarnObject(targetPlayer)
                    local count = 0
                    if warnObj then
                        if warnObj.type == "attribute" then count = warnObj.value
                        elseif warnObj:IsA("IntValue") or warnObj:IsA("NumberValue") then count = warnObj.Value
                        elseif warnObj:IsA("StringValue") then count = tonumber(warnObj.Value) or 0 end
                    end
                    warnCount.Text = tostring(count)
                    warnDisplay.BackgroundColor3 = Color3.fromRGB(239, 68, 68)
                    warnStroke.Color = Color3.fromRGB(239, 68, 68)
                    warnLabel.TextColor3 = Color3.fromRGB(248, 113, 113)
                    warnCount.TextColor3 = Color3.fromRGB(248, 113, 113)
                end
            end)

            connectAndTrack(card.MouseEnter, function()
                if target ~= targetPlayer and not multiTargets[targetPlayer] then
                    card.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
                end
            end)
            connectAndTrack(card.MouseLeave, function()
                if target ~= targetPlayer and not multiTargets[targetPlayer] then
                    card.BackgroundColor3 = isSelf and Color3.fromRGB(60, 60, 80) or Color3.fromRGB(50, 50, 50)
                end
            end)

            playerCards[targetPlayer.UserId] = card
            return card
        end

        local refreshDebounce = false
        local function refreshPlayerList()
            if refreshDebounce then return end
            refreshDebounce = true
            task.defer(function()
                local searchText = searchBox.Text:lower()
                if searchText ~= lastSearchText then
                    lastSearchText = searchText
                    for _, child in ipairs(playerListScroll:GetChildren()) do
                        if child:IsA("TextButton") and child.Name:match("^PlayerCard_") then child:Destroy() end
                    end
                    table.clear(playerCards)
                    for _, p in ipairs(Players:GetPlayers()) do
                        local matches = searchText == "" or p.Name:lower():find(searchText, 1, true) or p.DisplayName:lower():find(searchText, 1, true)
                        if matches then createPlayerCard(p) end
                    end
                else
                    for _, p in ipairs(Players:GetPlayers()) do
                        local matches = searchText == "" or p.Name:lower():find(searchText, 1, true) or p.DisplayName:lower():find(searchText, 1, true)
                        local existingCard = playerCards[p.UserId]
                        if matches and not existingCard then createPlayerCard(p)
                        elseif not matches and existingCard then existingCard:Destroy(); playerCards[p.UserId] = nil end
                    end
                end
                updateCanvasSize()
                task.wait(0.1)
                refreshDebounce = false
            end)
        end

        local function updateTotalPlayerCount()
            playerCountLabel.Text = tostring(#Players:GetPlayers())
        end

        refreshPlayerList()
        updateTotalPlayerCount()

        connectAndTrack(Players.PlayerAdded, function()
            task.wait(0.1)
            refreshPlayerList()
            updateTotalPlayerCount()
        end)
        connectAndTrack(Players.PlayerRemoving, function(plr)
            if playerCards[plr.UserId] then
                playerCards[plr.UserId]:Destroy()
                playerCards[plr.UserId] = nil
            end
            task.wait(0.1)
            updateTotalPlayerCount()
        end)

        local searchDebounce = false
        connectAndTrack(searchBox:GetPropertyChangedSignal("Text"), function()
            if not searchDebounce then
                searchDebounce = true
                task.delay(0.3, function()
                    refreshPlayerList()
                    searchDebounce = false
                end)
            end
        end)

        connectAndTrack(clearMultiBtn.MouseButton1Click, function()
            for plr in pairs(multiTargets) do multiTargets[plr] = nil end
            for _, card in ipairs(playerListScroll:GetChildren()) do
                if card:IsA("TextButton") and card.Name:match("^PlayerCard_") then
                    local avatarImg = card:FindFirstChild("ImageLabel")
                    if avatarImg then
                        local checkmark = avatarImg:FindFirstChild("Checkmark")
                        if checkmark then checkmark.Visible = false end
                    end
                    local isSelf = (card.Name == "PlayerCard_" .. player.UserId)
                    card.BackgroundColor3 = isSelf and Color3.fromRGB(60, 60, 80) or Color3.fromRGB(50, 50, 50)
                    local stroke = card:FindFirstChild("CardStroke")
                    if stroke then stroke.Color = isSelf and Color3.fromRGB(100, 100, 150) or Color3.fromRGB(80, 80, 80) end
                end
            end
            selectedCountLabel.Text = ""
        end)

        connectAndTrack(multiSelectBtn.MouseButton1Click, function()
            multiSelectMode = not multiSelectMode
            if multiSelectMode then
                multiSelectBtn.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
                multiSelectBtn.Text = "Single Select"
                clearMultiBtn.Visible = true
                selectedCountLabel.Visible = true
                local count = 0
                for _ in pairs(multiTargets) do count = count + 1 end
                selectedCountLabel.Text = count > 0 and (count .. " sel.") or ""
            else
                multiSelectBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
                multiSelectBtn.Text = "Multi Select"
                clearMultiBtn.Visible = false
                selectedCountLabel.Visible = false
                selectedCountLabel.Text = ""
                table.clear(multiTargets)
                for _, card in ipairs(playerListScroll:GetChildren()) do
                    if card:IsA("TextButton") and card.Name:match("^PlayerCard_") then
                        local avatarImg = card:FindFirstChild("ImageLabel")
                        if avatarImg then
                            local checkmark = avatarImg:FindFirstChild("Checkmark")
                            if checkmark then checkmark.Visible = false end
                        end
                    end
                end
            end
        end)

        -- Referências do card de alvo (usadas pelo TARGET tool)
        local targetCardRefs = {
            targetName = targetName,
            targetUsername = targetUsername,
            targetAvatar = targetAvatar,
            warnCount = warnCount,
            warnDisplay = warnDisplay,
            warnStroke = warnStroke,
            warnLabel = warnLabel,
        }

        return {
            gui = gui,
            closeBtn = closeBtn,
            mainContainer = mainContainer,
            header = header,
            confirmGui = confirmGui,
            targetCardRefs = targetCardRefs,
        }
    end

    local ui = createUI()

    -- ========== COLETOR AUTOMÁTICO DE QUESTS (ATUALIZADO) ==========
    local QuestSystemRemotes = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("QuestSystem"):WaitForChild("Remotes")
    local QuestRF = QuestSystemRemotes:WaitForChild("QuestRF")
    local QuestUpdateRE = QuestSystemRemotes:WaitForChild("QuestUpdateRE")

    local function collectAvailableQuests()
        pcall(function()
            local state = QuestRF:InvokeServer("GetState")
            if type(state) == "table" then
                for _, quest in ipairs(state.quests) do
                    if quest.canCollect then
                        pcall(function()
                            QuestRF:InvokeServer("Collect", {questId = quest.id})
                        end)
                        task.wait(0.1)
                    end
                end
            end
        end)
    end

    connectAndTrack(QuestUpdateRE.OnClientEvent, function(eventType, data)
        if eventType == "State" and type(data) == "table" then
            for _, quest in ipairs(data.quests) do
                if quest.canCollect then
                    pcall(function()
                        QuestRF:InvokeServer("Collect", {questId = quest.id})
                    end)
                    task.wait(0.1)
                end
            end
        end
    end)

    local autoQuestThread = task.spawn(function()
        while true do
            collectAvailableQuests()
            task.wait(5)
        end
    end)

    -- ========== DESTRUIÇÃO COMPLETA ==========
    local function destroyAll()
        disableAntiAFK()
        stopAntiCarry()
        stopChestAuto()
        stopAntiRagdoll()
        if FlexFPS.StopFlexSpoof then FlexFPS.StopFlexSpoof() end

        -- Desconecta view
        if viewCharacterAddedConn then
            viewCharacterAddedConn:Disconnect()
            viewCharacterAddedConn = nil
        end

        for _, conn in ipairs(allConnections) do
            if conn.Connected then conn:Disconnect() end
        end
        table.clear(allConnections)

        if autoReCharAddedConn then autoReCharAddedConn:Disconnect(); autoReCharAddedConn = nil end
        if toolCharAddedConn then toolCharAddedConn:Disconnect(); toolCharAddedConn = nil end
        if toolBackpackConn then toolBackpackConn:Disconnect(); toolBackpackConn = nil end
        comandosAtivos = false

        if waterWalkHeartbeatConn then waterWalkHeartbeatConn:Disconnect(); waterWalkHeartbeatConn = nil end
        if walkWaterPart then walkWaterPart:Destroy(); walkWaterPart = nil end

        for _, obj in ipairs(espObjects) do if obj then obj:Destroy() end end
        table.clear(espObjects)
        if viewing then camera.CameraSubject = player.Character and player.Character:FindFirstChildOfClass("Humanoid") end
        table.clear(playerCards)
        table.clear(warnCache)
        if autoQuestThread then task.cancel(autoQuestThread) end
        ui.confirmGui:Destroy()
        ui.gui:Destroy()
    end

    connectAndTrack(ui.closeBtn.MouseButton1Click, destroyAll)

    -- ========== DRAG DA GUI ==========
    local dragging = false
    local dragStartPos, frameStartPos
    connectAndTrack(ui.header.InputBegan, function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStartPos = UserInputService:GetMouseLocation()
            frameStartPos = ui.mainContainer.Position
        end
    end)
    connectAndTrack(ui.header.InputEnded, function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
    end)
    connectAndTrack(UserInputService.InputChanged, function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = UserInputService:GetMouseLocation() - dragStartPos
            ui.mainContainer.Position = UDim2.new(
                frameStartPos.X.Scale, frameStartPos.X.Offset + delta.X,
                frameStartPos.Y.Scale, frameStartPos.Y.Offset + delta.Y
            )
        end
    end)

    -- ========== ANIMAÇÃO PREMIUM (B) ==========
    local tweenInfo = TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    local function hideGui()
        if isAnimating or not guiVisible then return end
        isAnimating = true
        guiVisible = false
        local heightTween = TweenService:Create(ui.mainContainer, tweenInfo, {Size = UDim2.new(0, 500, 0, 44)})
        heightTween:Play()
        heightTween.Completed:Wait()
        local widthTween = TweenService:Create(ui.mainContainer, tweenInfo, {Size = UDim2.new(0, 0, 0, 44)})
        widthTween:Play()
        widthTween.Completed:Wait()
        ui.mainContainer.Visible = false
        isAnimating = false
    end
    local function showGui()
        if isAnimating or guiVisible then return end
        isAnimating = true
        ui.mainContainer.Visible = true
        ui.mainContainer.Size = UDim2.new(0, 0, 0, 44)
        local widthTween = TweenService:Create(ui.mainContainer, tweenInfo, {Size = UDim2.new(0, 500, 0, 44)})
        widthTween:Play()
        widthTween.Completed:Wait()
        local heightTween = TweenService:Create(ui.mainContainer, tweenInfo, {Size = UDim2.new(0, 500, 0, 400)})
        heightTween:Play()
        heightTween.Completed:Wait()
        guiVisible = true
        isAnimating = false
    end

    -- ========== TECLADO ==========
    connectAndTrack(UserInputService.InputBegan, function(input, gameProcessed)
        if gameProcessed then return end
        if input.KeyCode == Enum.KeyCode.LeftControl then
            ctrlPressed = true
        elseif input.KeyCode == Enum.KeyCode.B then
            if guiVisible then hideGui() else showGui() end
        elseif input.KeyCode == Enum.KeyCode.K then
            espEnabled = not espEnabled
            for _, obj in ipairs(espObjects) do obj.Enabled = espEnabled end
        elseif input.KeyCode == Enum.KeyCode.R then
            executarComandoERetornar()
        end
    end)
    connectAndTrack(UserInputService.InputEnded, function(input)
        if input.KeyCode == Enum.KeyCode.LeftControl then ctrlPressed = false end
    end)

    -- ========== ATALHO E (OPEN DOOR) ==========
    pcall(function()
        local remotes = require(ReplicatedStorage:WaitForChild("Packages"):WaitForChild("Remotes"))
        local openDoorRemote = remotes.Event("OpenDoor")
        connectAndTrack(UserInputService.InputBegan, function(input, gameProcessed)
            if input.KeyCode == Enum.KeyCode.E and not gameProcessed then openDoorRemote:FireServer() end
        end)
    end)

    -- ========== CTRL+TP & CLIPBOARD (ATUALIZAÇÃO DO TARGET TOOL) ==========
    connectAndTrack(mouse.Button1Down, function()
        local toolEquipped = player.Character and player.Character:FindFirstChild("TARGET")
        if ctrlPressed then
            local char = player.Character
            if char then
                local hrp = char:FindFirstChild("HumanoidRootPart")
                if hrp then
                    local targetPos = mouse.Hit.Position + Vector3.new(0, 3, 0)
                    hrp.CFrame = CFrame.new(targetPos) * (hrp.CFrame - hrp.CFrame.Position)
                end
            end
        end

        local ray = camera:ScreenPointToRay(mouse.X, mouse.Y)
        local result = Workspace:Raycast(ray.Origin, ray.Direction * 1000)
        if not result then return end
        local character = result.Instance.Parent
        if not character:FindFirstChild("Highlight") then character = character.Parent end
        if not character or not character:FindFirstChild("Highlight") then return end
        local p = Players:GetPlayerFromCharacter(character)
        if p and p ~= player then
            pcall(function() setclipboard("@" .. p.Name .. " | ID: " .. p.UserId) end)

            if toolEquipped then
                target = p
                targetMuted = false
                if muteBtn then
                    muteBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
                end

                -- Atualiza o view se estiver ativo
                if viewing then
                    updateViewCamera()
                    setupViewConnection()
                end

                local refs = ui.targetCardRefs
                if refs then
                    refs.targetName.Text = p.DisplayName
                    refs.targetUsername.Text = "@" .. p.Name
                    refs.targetAvatar.Image = "https://www.roblox.com/headshot-thumbnail/image?userId=" .. p.UserId .. "&width=64&height=64&format=png"

                    local warnObj = findWarnObject(p)
                    local count = 0
                    if warnObj then
                        if warnObj.type == "attribute" then
                            count = warnObj.value
                        elseif warnObj:IsA("IntValue") or warnObj:IsA("NumberValue") then
                            count = warnObj.Value
                        elseif warnObj:IsA("StringValue") then
                            count = tonumber(warnObj.Value) or 0
                        end
                    end
                    refs.warnCount.Text = tostring(count)
                    refs.warnDisplay.BackgroundColor3 = Color3.fromRGB(239, 68, 68)
                    refs.warnStroke.Color = Color3.fromRGB(239, 68, 68)
                    refs.warnLabel.TextColor3 = Color3.fromRGB(248, 113, 113)
                    refs.warnCount.TextColor3 = Color3.fromRGB(248, 113, 113)
                end
            end
        end
    end)
end)()
