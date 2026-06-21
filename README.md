--[[
    SUPER GIRL MOD v2
]]

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")
local SoundService = game:GetService("SoundService")
local VirtualUser = game:GetService("VirtualUser")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local hrp = character:FindFirstChild("HumanoidRootPart")
local mouse = player:GetMouse()
local camera = Workspace.CurrentCamera

local allConnections = {}
local persistentConnections = {}
local originalGravity = (Workspace.Gravity > 0) and Workspace.Gravity or 196.2
local antiGravityForce = nil
local lockedPlayer = nil
local isR6 = (humanoid.RigType == Enum.HumanoidRigType.R6)

local R6_FLIGHT_ANIMS = {
    Idle = 97172005,
    Idle2 = 21633130,
    Forward = 46196309,
    Fast = 282574440,
    Back = 48957148, -- Se congelará en 0.7
    Up = 90872539,
    Down = 233322916
}

-- FLIGHT VARIABLES
local isFlying = false
local r6FlightIdleMode = 1
local flightSpeedValue = 16
local flightIdleAnim = "124339843934871"
local flightForwardAnim = "121652468298377"
local flightFastAnim = "120303626369803"
local flightSlowAnim = "138992096476836"
local flightBackwardAnim = "78684337927052"
local currentFlightAnimTrack = nil
local currentAnimTrack = nil
local currentEmoteSound = nil
local flyGyro = nil
local flyVelocity = nil
local flyPos = nil
local flightMoveState = {forward = 0, backward = 0, left = 0, right = 0, up = 0, down = 0}
local flightConns = {}
local r6FlightTracks = {}
local r6ForwardHold = 0
local currentFlightCF = nil
local flightLerpCoef = 0.25
local flightCurrentRoll = 0
local flightEmoteActive = false
local lastStationaryPos = nil
local flightStaticRotation = true
local currentImpulseVelocity = Vector3.new(0, 0, 0)
local walkPos = nil
local lastWalkStationaryPos = nil

local FLIGHT_EMOTES = isR6 and {
    66703954,
    54513258 -- Antiguo Idle 1 ahora como Emote
} or {
    102620389167016, 104879320142507, 118287320739937, 120718177198595, 
    124369689700678, 89173946476926, 133245709757511, 72132583099218, 
    136280235646918, 114527286143657, 77099947447557, 127117414275525
}

local currentFlightEmoteIndex = 1

-- ANIMACIONES
local ANIMATIONS = {
    Idle = 136517279340153,
    Walk = 16738340646,
    Run = 70636286183373,
    Jump = 104325245285198,
    Fall = 16738333171,
    Climb = 10921257536,
    Swim = 10921243048,
    SwimIdle = 10921265698,
    Sit = 2506281703
}

local idleAnimId = 136517279340153
local hurtAnimId = 140704739422954

local managedAnimationIds = {} -- Almacenamos solo los números de ID para comparaciones precisas
local function registerManagedId(id)
    if not id then return end
    local numId = typeof(id) == "number" and string.format("%.0f", id) or tostring(id):match("%d+")
    if numId then managedAnimationIds[numId] = true end
end

local LOOP_ANIMATION_IDS = {}
for _, id in pairs(ANIMATIONS) do
    LOOP_ANIMATION_IDS["rbxassetid://" .. id] = true
    registerManagedId(id)
end

local extraIds = {flightIdleAnim, "93032051646965", flightForwardAnim, flightFastAnim, flightSlowAnim, flightBackwardAnim, "124339843934871", "138992096476836", "84043660421785", "111690963588496", "128406664848479", "140704739422954", "83860986564910", "103145593690285", "128578785610052", "78510387198062", "72301599441680", "87088218490918", "104687069461693", "142495255", "248336294", "54513258", "95424077", "97170520", "97172005", "21633130", "48957148", "56146409", "66703954", "73033633"}
for _, id in ipairs(extraIds) do
    LOOP_ANIMATION_IDS["rbxassetid://" .. id] = true
    registerManagedId(id)
end

registerManagedId(idleAnimId)

local selectableAnimations = isR6 and {
    73033633, -- Emote tierra buscado
    128777973, 128853357, 129423131
} or {
    82682811348660, 92689671662713, 102274955976143, 104687069461693,
    104998729896195, 106516971471692, 76167683097846,
    95494576365544, 138316142522795, 129115715729473, 106394178681320,
    136687532783709, 112192314086885, 133979485105808,
    80222624644964, 70456775231265, 132990883627615,
    120071905410216, 117722192552703, 124200992648318,
    113375965758912, 83766124558950, 133394554631338, 122599479076921, 130196652446769,
    91398182744709, 102439655497210, 93655929598076,
    140409138293585, 72101120437031, 119822440796584, 127786890738997,
    129269946654799,
    130264584381746,
    99873676774268,
    101409107532852,
    82238508652742,
    140536225775470,
    76176643257712,
    137880837576956,
    117913449580238,
    72326999255699,
    84668569083234, 97751249599208, 126644738448952,
    75117059305996, 110608120999378, 108703544549132,
    131673340109237, 77949063578180
}
for _, id in ipairs(selectableAnimations) do registerManagedId(id) end
for _, id in ipairs(FLIGHT_EMOTES) do registerManagedId(id) end
-- Registrar IDs adicionales del sistema de vuelo R6
local r6ExtraIds = {90872539, 136801964, 142495255, 97169019, 46196309, 282574440, 106772613, 42070810, 214744412, 233322916, 97171309, 248336294, 54513258, 95424077, 97170520, 97172005, 21633130}
for _, id in ipairs(r6ExtraIds) do registerManagedId(id) end

local currentAnimationIndex = 1
local muteEmotes = false
local noSitActive = false
local MOVEMENT_AUDIO_VOLUME = 0.7
local originalSoundProperties = {}

-- OTRAS HABILIDADES
local isWalkSpeedActive = false
local walkSpeed = 21
local minWalkSpeed = 11
local maxWalkSpeed = 200
local preFlightWalkSpeed = nil
local preFlightIsSpeedActive = nil
local originalWalkSpeed = 11
local originalJumpPower = 50

local walkAnimTrack = nil
local hurtOverlayTrack = nil
local sitAnimTrack = nil
local swimAnimTrack = nil
local swimIdleAnimTrack = nil
local climbAnimTrack = nil
local jumpAnimTrack = nil
local fallAnimTrack = nil
local isMirageSpeedActive = false
local mirageSpeedEmoteTrack = nil

local isSuperJumpActive = false
local superJumpConnection = nil
local superJumpPower = 70
local minSuperJumpPower = 10
local maxSuperJumpPower = 500
local superJumpStep = 10

local isSuperSaltoCharged = false
local superSaltoChargeTime = 0
local superSaltoTrack = nil
local isGravJumpActive = false

local SuperStrength = {
    Active = false,
    GrabbedData = nil,
    Distance = 500,
    Connection = nil,
    ThrowForce = 800,
    AnimTrack = nil,
    LastActionTime = 0,
    ActivationTime = 0
}

local SuperHearing = {
    Active = false,
    Heartbeat = nil,
    PlayerHighlights = {},
    NPCHighlights = {},
    Sound = nil,
    LastUpdate = 0,
    PlayerCache = {},
    NPCCache = {},
    ActiveNPCs = {},
    Timer = nil,
    Blur = nil,
    ColorCorrection = nil,
    CleanupTimer = nil,
    CachedNPCList = {},
    ActiveNPCs = {},
    LastNPCScanTime = 0,
    Echo = nil,
    Reverb = nil
}

local InstantReaction = {
    Active = false,
    LastDodge = 0,
    Cooldown = 0.3, -- Cooldown optimizado para combate rápido
    DetectionRange = 20,
    Connection = nil,
    DodgeActive = false
}
local superHearingUpdateInterval = 0.1
local SUPER_HEARING_MAX_VOLUME = 5
local SUPER_HEARING_DURATION = 15

local ThermalVision = {
    Active = false,
    OriginalLighting = {},
    ColorCorrection = nil,
    Connection = nil,
    Heartbeat = nil,
    Highlights = {},
    LastNPCScanTime = 0,
    CachedNPCList = {},
    CachedItemList = {},
    LastItemScanTime = 0,
    LastPositions = {},
    ChildAddedConnection = nil
}

local isInvisibilityActive = false
local invisibilitySeat = nil
local noclipConnection = nil
local preInvisibilityCF = nil

local isAntiTPActive = false
local antiTPConnections = {}

local isAntiFallActive = false
local antiFallConnections = {}
local antiFallTeleporting = false

local isAntiFlingActive = false
local antiFlingConnection = nil

local isSprintActive = false
local sprintAnimTrack = nil

local isCrouching = false
local customAnimationsLoaded = false
local animationTracks = {}
local idleSequenceConnection = nil

local toggleAbility = nil
local cleanupEverything = nil
local adjustSpeed = nil
local startMainAnimationLoop = nil
local playAnimation = nil
local loadAnimations = nil
local cancelCustomAnims = false

local function isExternalAnimPlaying()
    if not humanoid or not character then return false end
    
    -- Detectar si estamos soldados a otro jugador (Carry/Grab de Brookhaven)
    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("Weld") or part:IsA("WeldConstraint") or part:IsA("ManualWeld") then
            local p0 = part.Part0
            local p1 = part.Part1
            if p0 and p1 then
                local other = p0:IsDescendantOf(character) and p1 or p0
                if not other:IsDescendantOf(character) and other:FindFirstAncestorOfClass("Model") and Players:GetPlayerFromCharacter(other:FindFirstAncestorOfClass("Model")) then
                    return true
                end
            end
        end
    end

    for _, track in ipairs(humanoid:GetPlayingAnimationTracks()) do
        local numId = track.Animation.AnimationId:match("%d+")
        local name = track.Name:lower()
        local isBasic = name:find("walk") or name:find("run") or name:find("idle") or name:find("jump") or name:find("fall") or name:find("strafe") or name:find("swimming") or name:find("movement") or name:find("dash") or name:find("sprint") or track.Looped
        
        if track.IsPlaying and numId and not managedAnimationIds[numId] then
            if not isBasic and (name:find("cutscene") or name:find("scene") or name:find("event") or name:find("elevator") or name:find("intro") or name:find("cinematic") or name:find("search") or name:find("open") or track.Priority.Value >= Enum.AnimationPriority.Action3.Value) then
                 return true
            end
        end
    end
    return false
end

local function getTargetFromCamera()
    local center = camera.ViewportSize / 2
    local target = nil
    local minDistance = 250 -- Sensibilidad para detectar al jugador en el centro
    
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local hum = p.Character:FindFirstChild("Humanoid")
            if hum and hum.Health > 0 then
                local pos, onScreen = camera:WorldToViewportPoint(p.Character.HumanoidRootPart.Position)
                if onScreen then
                    local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
                    if dist < minDistance then
                        minDistance = dist
                        target = p
                    end
                end
            end
        end
    end
    return target
end

-- ============ SUPER STRENGTH HELPERS (EXTRACTION) ============
local function getVehicleRootPart(targetPart)
    local model = targetPart:FindFirstAncestorOfClass("Model")
    if not model then return targetPart end
    local vehicleParts = {"MainPart", "Body", "Chassis", "RootPart", "PrimaryPart", "Handle", "VehicleSeat", "Seat"}
    for _, partName in ipairs(vehicleParts) do
        local foundPart = model:FindFirstChild(partName, true)
        if foundPart and foundPart:IsA("BasePart") then return foundPart end
    end
    return model.PrimaryPart or targetPart
end

local function disconnectVehicleOccupants(model)
    local savedStates = {}
    for _, seat in ipairs(model:GetDescendants()) do
        if seat:IsA("VehicleSeat") or seat:IsA("Seat") then
            savedStates[seat] = {
                Disabled = seat.Disabled,
                Occupant = seat.Occupant,
                CanCollide = seat.CanCollide
            }
            seat.Disabled = true
            if seat.Occupant then
                local hum = seat.Occupant.Parent:FindFirstChild("Humanoid")
                if hum then hum.Sit = false end
                seat.Occupant = nil
            end
        end
    end
    return savedStates
end

local function canGrabObject(part)
    if not part or part:IsDescendantOf(character) or part:IsA("Terrain") then return false end

    -- SEGURIDAD: No agarrar el suelo que estamos pisando
    local rayParams = RaycastParams.new()
    rayParams.FilterDescendantsInstances = {character}
    local groundCheck = Workspace:Raycast(hrp.Position, Vector3.new(0, -10, 0), rayParams)
    if groundCheck and (part == groundCheck.Instance or part:IsAncestorOf(groundCheck.Instance)) then
        return false
    end

    local model = part:FindFirstAncestorOfClass("Model")
    local isVehicle = false
    if model and (model:FindFirstChildWhichIsA("VehicleSeat") or model:FindFirstChildWhichIsA("Seat") or 
       model.Name:lower():find("car") or model.Name:lower():find("vehic") or model.Name:lower():find("chassis")) then
        isVehicle = true
    end
    
    -- Si es un vehículo, permitimos el agarre aunque esté anclado o sea grande
    if isVehicle then
        local mainPart = getVehicleRootPart(part)
        return true, mainPart or part, model, isVehicle
    end

    if part.Anchored and part.Size.Magnitude > 50 then return false end
    local mainPart = getVehicleRootPart(part)
    if mainPart and mainPart.Anchored then return false end
    
    return true, mainPart or part, model, isVehicle
end

local function getVehicleMass(model)
    local totalMass = 0
    for _, part in ipairs(model:GetDescendants()) do
        if part:IsA("BasePart") then totalMass = totalMass + part:GetMass() end
    end
    return math.max(totalMass, 1)
end

local function playGrabAnim()
    if not humanoid or SuperStrength.AnimTrack then return end
    local grabId = isR6 and 56146409 or 87088218490918
    local anim = Instance.new("Animation")
    anim.AnimationId = "rbxassetid://" .. tostring(grabId)
    SuperStrength.AnimTrack = humanoid:LoadAnimation(anim)
    SuperStrength.AnimTrack.Priority = Enum.AnimationPriority.Action4
    SuperStrength.AnimTrack.Looped = true
    SuperStrength.AnimTrack:Play()
    SuperStrength.AnimTrack:AdjustSpeed(0) -- Congela la pose
end

local function stopGrabAnim()
    if SuperStrength.AnimTrack then
        SuperStrength.AnimTrack:Stop(0.1)
        SuperStrength.AnimTrack = nil
    end
end

local function releaseGrabbedObject(shouldThrow)
    if not SuperStrength.GrabbedData then return end
    local data = SuperStrength.GrabbedData
    SuperStrength.GrabbedData = nil
    SuperStrength.LastActionTime = tick()
    
    -- Restaurar propiedades físicas originales (Masa y Colisión)
    if data.savedPhysics then
        for part, props in pairs(data.savedPhysics) do
            if part and part.Parent then 
                part.Massless = props.Massless
                part.CanCollide = props.CanCollide
                for _, child in ipairs(part:GetChildren()) do
                    if child:IsA("NoCollisionConstraint") and child.Name == "SuperGrabNC" then
                        child:Destroy()
                    end
                end
            end
        end
    end

    local root = data.root
    if data.alignPos then data.alignPos:Destroy() end
    if data.alignOrient then data.alignOrient:Destroy() end
    if data.att0 then data.att0:Destroy() end
    if data.att1 then data.att1:Destroy() end
    stopGrabAnim()
    
    if shouldThrow then
        local camLook = camera.CFrame.LookVector
        local mass = (data.isVehicle and data.model) and getVehicleMass(data.model) or (root.AssemblyMass or 1)
        pcall(function() root:ApplyImpulse(camLook * SuperStrength.ThrowForce * mass * 2) end)
    end

    if data.model and data.savedStates then
        for seat, state in pairs(data.savedStates) do
            if seat and seat.Parent then seat.Disabled = state.Disabled end
        end
    end

    -- Terminación limpia del estado para evitar re-agarre accidental
    SuperStrength.Active = false
    if SuperStrength.Connection then 
        SuperStrength.Connection:Disconnect() 
        SuperStrength.Connection = nil 
    end
    updateMainUI()
end
-- ============================================================

local abilities = {
    {name = "Velocidad", key = "walkspeed"},
    {name = "Espejismo", key = "miragespeed"},
    {name = "Super Audición", key = "superhearing"},
    {name = "S. Gravitatorio", key = "gravjump"},
    {name = "Super Salto", key = "supersalto"},
    {name = "Visión Térmica", key = "thermalvision"},
    {name = "V.Molecular", key = "molecularvibration"},
    {name = "Vuelo", key = "flight"},
    {name = "Super Agarre", key = "superstrength"},
    {name = "Re.Instantanea", key = "instantreaction"}
}

local function isAbilityAvailable(key)
    if isR6 and (key == "miragespeed" or key == "supersalto") then
        return false
    end
    if not isR6 and key == "instantreaction" then
        return false
    end
    return true
end

-- ============ MENÚ SELECTOR ============
local MODES = {
    ABILITIES = 1,
    EMOTES = 2,
    SETTINGS = 3
}
local currentMode = MODES.ABILITIES
local modeNames = {[1] = "HABILIDADES", [2] = "EMOTES", [3] = "AJUSTES"}
local settingOptions = {"ESTATICO: ON", "SILENCIAR: OFF", "NOSIT: OFF", "R6 FLY: 1", "DESTRUIR"}
local currentSettingIndex = 1

local screenGui = nil
local mainPanel = nil
local logoImage = nil
local modeText = nil
local abilityText = nil
local settingsPanel = nil
local speedIndicatorLabel = nil
local plusBtn = nil
local minusBtn = nil

local scriptActive = true
local menuOpen = false
local currentAbilityIndex = 7
local lastAbilityTapTime = 0

local lastModeTap = 0
local modeTapThread = nil
local lastAbilityTap = 0
local abilityTapThread = nil

-- ============ FLIGHT FUNCTIONS ============

local function disableDefaultAnimate()
    local function process(parent)
        if not parent then return end
        for _, scr in ipairs(parent:GetChildren()) do
            if scr:IsA("LocalScript") and (scr.Name:find("Animate") or scr.Name:find("Anim") or scr.Name:find("Handler") or scr.Name:find("Movement") or scr.Name:find("Sprint")) then
                scr.Disabled = true
            end
        end
    end
    process(character)
    process(player:FindFirstChild("PlayerGui"))
end

local function enableDefaultAnimate()
    local function process(parent)
        if not parent then return end
        for _, scr in ipairs(parent:GetChildren()) do
            if scr:IsA("LocalScript") and (scr.Name:find("Animate") or scr.Name:find("Anim") or scr.Name:find("Handler") or scr.Name:find("Movement") or scr.Name:find("Sprint")) then
                scr.Disabled = false
            end
        end
    end
    process(character)
    process(player:FindFirstChild("PlayerGui"))
end

local function playFlightAnimation(animId, startTime, speed)
    if not character or not humanoid then return end
    
    if currentFlightAnimTrack then
        currentFlightAnimTrack:Stop(0.1)
        currentFlightAnimTrack = nil
    end
    
    disableDefaultAnimate()
    
    for _, track in ipairs(humanoid:GetPlayingAnimationTracks()) do
        if not SuperStrength.AnimTrack or track ~= SuperStrength.AnimTrack then
            track:Stop()
        end
    end
    
    local anim = Instance.new("Animation")
    anim.AnimationId = "rbxassetid://" .. tostring(animId)
    currentFlightAnimTrack = humanoid:LoadAnimation(anim)
    currentFlightAnimTrack.Priority = Enum.AnimationPriority.Action4
    currentFlightAnimTrack.Looped = true
    currentFlightAnimTrack:Play()
    if startTime then currentFlightAnimTrack.TimePosition = startTime end
    if speed then currentFlightAnimTrack:AdjustSpeed(speed) end
end

local function stopFlightAnimation()
    if currentFlightAnimTrack then
        currentFlightAnimTrack:Stop(0.1)
        currentFlightAnimTrack = nil
    end
    if jumpAnimTrack then jumpAnimTrack:Stop(0.1) end
    if fallAnimTrack then fallAnimTrack:Stop(0.1) end
    enableDefaultAnimate()
end

local function setupFlightMovementAudio()
    if flightAudio then flightAudio:Destroy() end
    flightAudio = Instance.new("Sound")
    flightAudio.SoundId = "rbxassetid://596046130"
    flightAudio.Volume = 0
    flightAudio.Looped = true
    flightAudio.Name = "FlightMovementSound"
    flightAudio.Parent = hrp or character:FindFirstChild("HumanoidRootPart")
    flightAudio:Play()
end

local function updateFlightAudios(fwd)
    if not flightAudio or not isFlying then return end
    
    if fwd > 0.1 or fwd < -0.1 then
        if flightAudio.Volume < MOVEMENT_AUDIO_VOLUME then
            flightAudio.Volume = math.min(flightAudio.Volume + 0.05, MOVEMENT_AUDIO_VOLUME)
        end
        -- Ajustar el pitch según la velocidad (usa walkSpeed si está activa)
        local currentSpeed = isWalkSpeedActive and walkSpeed or 21
        local speedFactor = currentSpeed / 200
        flightAudio.Pitch = 0.8 + (speedFactor * 0.4)
    else
        if flightAudio.Volume > 0 then
            flightAudio.Volume = math.max(flightAudio.Volume - 0.05, 0)
        end
    end
end

local function stopFlightAudio()
    if flightAudio then
        local target = flightAudio
        flightAudio = nil
        task.spawn(function()
            local startVol = target.Volume
            for i = 1, 10 do
                target.Volume = startVol * (1 - i/10)
                task.wait(0.05)
            end
            target:Stop()
            target:Destroy()
        end)
    end
end

local function stopR6FlightAnims()
    for _, track in pairs(r6FlightTracks) do
        track:Stop(0.1)
    end
end

local function setupR6FlightAnims()
    if not isR6 or not humanoid then return end
    stopR6FlightAnims()
    r6FlightTracks = {}
    
    local idleId = (r6FlightIdleMode == 1) and R6_FLIGHT_ANIMS.Idle or R6_FLIGHT_ANIMS.Idle2

    local anims = {
        up = 90872539, 
        down = 233322916, 
        l1 = 48957148, l2 = 142495255,
        r1 = 48957148, r2 = 142495255,
        b1 = 48957148, b2 = 106772613, b3 = 42070810, b4 = 214744412,
        fLow1 = 46196309, fLow2 = 282574440, 
        fFast = 282574440,
        idle = idleId
    }
    
    for k, id in pairs(anims) do
        local a = Instance.new("Animation")
        a.AnimationId = "rbxassetid://" .. id
        local track = humanoid:LoadAnimation(a)
        track.Priority = Enum.AnimationPriority.Action4
        r6FlightTracks[k] = track
    end
end

local function startFlight()
    if isFlying then return end
    
    -- Guardar estado previo y establecer velocidad de vuelo inicial (16)
    preFlightWalkSpeed = walkSpeed
    preFlightIsSpeedActive = isWalkSpeedActive
    
    walkSpeed = flightSpeedValue
    isWalkSpeedActive = true

    isFlying = true
    
    -- No tocamos la gravedad del workspace (evita detección)
    -- Usamos un BodyForce para contrarrestar la gravedad localmente
    antiGravityForce = Instance.new("BodyForce")
    antiGravityForce.Name = "FlightBuoyancy"
    antiGravityForce.Force = Vector3.new(0, hrp:GetMass() * Workspace.Gravity, 0)
    antiGravityForce.Parent = hrp

    humanoid.AutoRotate = false
    humanoid.PlatformStand = true
    
    -- Reset momentum to avoid rubberbanding
    hrp.AssemblyLinearVelocity = Vector3.new(0,0,0)
    hrp.AssemblyAngularVelocity = Vector3.new(0,0,0)
    
    -- Aseguramos que el ancla de caminar esté apagada
    if walkPos then walkPos.MaxForce = Vector3.new(0,0,0) end

    setupFlightMovementAudio()
    
    if not SuperStrength.GrabbedData then
        if isR6 then 
            setupR6FlightAnims()
            disableDefaultAnimate()
        else 
            playFlightAnimation(flightIdleAnim, 0, 1) 
        end
    end
    
    flyGyro = Instance.new("BodyGyro")
    flyGyro.Name = "FlyGyro"
    flyGyro.P = 30000 -- Aumentado para un giro más responsivo y agresivo
    flyGyro.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
    flyGyro.CFrame = hrp.CFrame
    flyGyro.Parent = hrp
    
    flyPos = Instance.new("BodyPosition")
    flyPos.Name = "FlyPos"
    flyPos.MaxForce = Vector3.new(0, 0, 0) -- Inicia desactivado
    flyPos.D = 1500 -- Más amortiguación para evitar rebotes
    flyPos.P = 50000 -- Mucha más fuerza para que sea "rígido"
    flyPos.Parent = hrp
    
    flyVelocity = Instance.new("BodyVelocity")
    flyVelocity.Name = "FlyVelocity"
    flyVelocity.MaxForce = Vector3.new(9e9, 9e9, 9e9)
    flyVelocity.Velocity = Vector3.new(0, 0, 0)
    flyVelocity.Parent = hrp
    lastStationaryPos = nil
    currentFlightCF = hrp.CFrame
    
    local flightUpdate = RunService.RenderStepped:Connect(function(deltaTime)
        if not isFlying or not character or not hrp then return end

        -- Bypass: Forzamos el estado a 'Running' para que el servidor crea que estamos en el suelo
        -- Esto evita que el servidor detecte el estado 'Falling' prolongado
        if math.random(1, 5) == 1 then
            humanoid:ChangeState(Enum.HumanoidStateType.Running)
        end

        local isExternal = isExternalAnimPlaying()
        local levitationOffset = (isR6 and not isExternal) and (math.sin(tick() * 3.5) * 0.55) or 0

        if isExternal then
            if currentFlightAnimTrack then currentFlightAnimTrack:Stop(0.2) currentFlightAnimTrack = nil end
            if isR6 then stopR6FlightAnims() end
            -- No retornamos porque el movimiento (Velocity) debe seguir funcionando
        end
        
        local cam = Workspace.CurrentCamera
        
        -- Soporte para Joystick de Android y Teclado usando MoveDirection
        local fwd = 0
        local side = 0
        if humanoid and humanoid.MoveDirection.Magnitude > 0 then
            local relativeMove = cam.CFrame:VectorToObjectSpace(humanoid.MoveDirection)
            fwd = -relativeMove.Z
            side = relativeMove.X
        end
        
        local direction = cam.CFrame.LookVector

        local upDown = (flightMoveState.up or 0) - (flightMoveState.down or 0)
        -- Movimiento basado en la dirección de la cámara + controles de altura (Espacio/Alt)
        local inputVec = (direction * fwd) + (cam.CFrame.RightVector * side) + (Vector3.new(0, 1, 0) * upDown)
        
        -- Lógica de inclinación (Roll) para Joystick y Teclado
        if math.abs(side) > 0.1 then
            flightCurrentRoll = (side > 0) and 15 or -15
        else
            flightCurrentRoll = 0
        end

        local desiredVelocity = Vector3.new(0, 0, 0)
        local isMoving = inputVec.Magnitude > 0.01

        if isMoving then
            lastStationaryPos = nil
            -- Apagamos el ancla de posición completamente mientras nos movemos
            if flyPos then flyPos.MaxForce = Vector3.new(0, 0, 0) end
            
            local currentSpeed = isWalkSpeedActive and walkSpeed or 16
            local directionPart = (direction * fwd) + (cam.CFrame.RightVector * side)
            local finalWASD = Vector3.new(0, 0, 0)
            if directionPart.Magnitude > 0.001 then
                finalWASD = directionPart.Unit * currentSpeed
            end
            
            desiredVelocity = finalWASD + Vector3.new(0, upDown, 0)
            
            if flyVelocity then
                flyVelocity.MaxForce = Vector3.new(9e9, 9e9, 9e9)
                flyVelocity.Velocity = desiredVelocity
            end
        else
            -- SISTEMA DE ANCLAJE RÍGIDO: Bloqueo total para evitar drifting (especial para Doors)
            -- Si hay una animación externa (cinemática), actualizamos la posición para no "luchar" contra el juego
            if not lastStationaryPos or isExternal then lastStationaryPos = hrp.Position end
            
            if flyVelocity then
                flyVelocity.MaxForce = Vector3.new(9e9, 9e9, 9e9) -- Mantenemos fuerza para frenado activo
                flyVelocity.Velocity = Vector3.zero
            end
            if flyPos then
                flyPos.MaxForce = Vector3.new(9e9, 9e9, 9e9) -- Bloqueo total X,Y,Z
                flyPos.Position = lastStationaryPos + Vector3.new(0, levitationOffset, 0)
            end
            
            -- Forzamos la posición de la CFrame para evitar el deslizamiento por inercia o red
            hrp.CFrame = CFrame.new(lastStationaryPos + Vector3.new(0, levitationOffset, 0)) * hrp.CFrame.Rotation
            hrp.AssemblyLinearVelocity = Vector3.zero -- Mata el movimiento residual en cada frame
            hrp.AssemblyAngularVelocity = Vector3.zero -- Evita rotaciones extrañas al frenar
        end
        
        -- Manejo de animaciones basado en el movimiento real (compatible con Joystick)
        if isR6 then
            -- Lógica de Animaciones R6 (del script de referencia)
            if not SuperStrength.GrabbedData and not isExternal and not flightEmoteActive then
                -- Prioridad absoluta: Detener animaciones de movimiento del juego (TSB, etc) para evitar conflicto
                for _, track in ipairs(humanoid:GetPlayingAnimationTracks()) do
                    local tId = track.Animation.AnimationId:match("%d+")
                    if tId and not managedAnimationIds[tId] then
                        -- TÉCNICA SUPREMA PARA R6: Si estamos volando y NO es un ataque (Action3+), 
                        -- forzamos la detención de cualquier animación loopeada (como el sprint de TSB)
                        -- o cualquier animación de prioridad baja que se dispare al movernos hacia adelante.
                        if track.Looped or track.Priority.Value <= Enum.AnimationPriority.Action2.Value then
                            track:Stop(0)
                            track:AdjustWeight(0, 0) -- Doble seguridad para ocultarla
                        end
                    end
                end

                if math.abs(upDown) > 0.1 then
                    if not r6FlightTracks.idle.IsPlaying then 
                        stopR6FlightAnims(); 
                        r6FlightTracks.idle:Play(); 
                        if r6FlightIdleMode == 1 then r6FlightTracks.idle.TimePosition = 2.0 end
                        r6FlightTracks.idle:AdjustSpeed(0) 
                    end
                elseif side < -0.1 then
                    if not r6FlightTracks.l1.IsPlaying then
                        stopR6FlightAnims()
                        r6FlightTracks.l1:Play(); r6FlightTracks.l1.TimePosition = 2.0; r6FlightTracks.l1:AdjustSpeed(0)
                        r6FlightTracks.l2:Play(); r6FlightTracks.l2.TimePosition = 0.5; r6FlightTracks.l2:AdjustSpeed(0)
                    end
                elseif side > 0.1 then
                    if not r6FlightTracks.r1.IsPlaying then
                        stopR6FlightAnims()
                        r6FlightTracks.r1:Play(); r6FlightTracks.r1.TimePosition = 1.1; r6FlightTracks.r1:AdjustSpeed(0)
                        r6FlightTracks.r2:Play(); r6FlightTracks.r2.TimePosition = 0.5; r6FlightTracks.r2:AdjustSpeed(0)
                    end
                elseif fwd < -0.1 then
                    -- NUEVA LÓGICA RETROCESO R6 (ID 48957148 congelada en 0.7)
                    if not r6FlightTracks.b1.IsPlaying then
                        stopR6FlightAnims()
                        r6FlightTracks.b1:Play(); 
                        r6FlightTracks.b1.TimePosition = 0.7; 
                        r6FlightTracks.b1:AdjustSpeed(0)
                    end
                elseif fwd > 0.1 then
                    r6ForwardHold = r6ForwardHold + deltaTime
                    local currentSpeed = isWalkSpeedActive and walkSpeed or 16
                    
                    -- Lógica de velocidad ajustada: Vuelo lento hasta 19, Rápido desde 20
                    if currentSpeed < 20 then
                        -- VUELO LENTO: Animación activa (ID 46196309) sin mezcla de brazos
                        if not r6FlightTracks.fLow1.IsPlaying or r6FlightTracks.fLow1.Speed == 0 then
                            stopR6FlightAnims()
                            r6FlightTracks.fLow1:Play(0.1, 1) -- Peso completo para evitar que se vea rara
                            r6FlightTracks.fLow1:AdjustSpeed(0.8)
                        end
                    else
                        -- VUELO RÁPIDO: Se activa a partir de 20
                        if not r6FlightTracks.fFast.IsPlaying then
                            stopR6FlightAnims()
                            r6FlightTracks.fFast:Play(); r6FlightTracks.fFast:AdjustSpeed(0.05)
                        end
                    end
                else
                    r6ForwardHold = 0
                    if not r6FlightTracks.idle.IsPlaying then
                        stopR6FlightAnims();
                        r6FlightTracks.idle:Play();
                        if r6FlightIdleMode == 1 then 
                            r6FlightTracks.idle.TimePosition = 2.0 
                        end
                        r6FlightTracks.idle:AdjustSpeed(0)
                    end
                end
            end
        else
            -- Lógica R15 Original
            local targetAnim = nil
            if SuperStrength.GrabbedData then
                if currentFlightAnimTrack then currentFlightAnimTrack:Stop(0.2) currentFlightAnimTrack = nil end
            elseif not isExternal and not flightEmoteActive then
                if fwd > 0.1 then
                    local currentSpeed = isWalkSpeedActive and walkSpeed or 16
                    targetAnim = (currentSpeed >= 20 and flightFastAnim or flightForwardAnim)
                elseif fwd < -0.1 then
                    targetAnim = flightBackwardAnim
                else
                    targetAnim = flightIdleAnim
                end

                if targetAnim then
                    if not currentFlightAnimTrack or currentFlightAnimTrack.Animation.AnimationId ~= "rbxassetid://" .. tostring(targetAnim) then
                        -- Se elimina el valor 0 para que la animación de vuelo no se congele a velocidades bajas
                        playFlightAnimation(targetAnim, 0, 1)
                    end
                end
            end
        end

        -- Rotación inteligente:
        -- Si "ESTATICO" está OFF, gira siempre. 
        -- Si "ESTATICO" está ON, solo gira si te estás moviendo.
        if isMoving or not flightStaticRotation then
            local camRotation = CFrame.lookAt(Vector3.zero, cam.CFrame.LookVector)
            local targetRotation = camRotation * CFrame.Angles(0, 0, math.rad(flightCurrentRoll))

            if currentFlightCF then
                currentFlightCF = currentFlightCF:Lerp(targetRotation, flightLerpCoef)
            else
                currentFlightCF = targetRotation
            end
        end

        if flyGyro then
            flyGyro.CFrame = currentFlightCF
        end

        -- Actualizar audio dinámico basado en el movimiento hacia adelante/atrás
        updateFlightAudios(fwd)
    end)
    table.insert(flightConns, flightUpdate)
    
    local function onFlightInputBegan(input, gameProc)
        if gameProc then return end
        if input.UserInputType == Enum.UserInputType.Keyboard then
            local key = input.KeyCode
            if key == Enum.KeyCode.W then
                flightMoveState.forward = 1
            elseif key == Enum.KeyCode.S then
                flightMoveState.backward = 1
            elseif key == Enum.KeyCode.A then
                flightMoveState.left = 1
            elseif key == Enum.KeyCode.D then
                flightMoveState.right = 1
            elseif key == Enum.KeyCode.Space then -- Ascender con Espacio
                flightMoveState.up = 3.5
            elseif key == Enum.KeyCode.RightAlt or key == Enum.KeyCode.LeftAlt then -- Descender con Alt
                flightMoveState.down = 3.5
            end
        end
    end
    
    local function onFlightInputEnded(input, gameProc)
        if gameProc then return end
        if input.UserInputType == Enum.UserInputType.Keyboard then
            local key = input.KeyCode
            if key == Enum.KeyCode.W then
                flightMoveState.forward = 0
            elseif key == Enum.KeyCode.S then
                flightMoveState.backward = 0
            elseif key == Enum.KeyCode.A then
                flightMoveState.left = 0
            elseif key == Enum.KeyCode.D then
                flightMoveState.right = 0
            elseif key == Enum.KeyCode.Space then -- Soltar Espacio
                flightMoveState.up = 0
            elseif key == Enum.KeyCode.RightAlt or key == Enum.KeyCode.LeftAlt then -- Soltar Alt
                flightMoveState.down = 0
            end
        end
    end
    
    local flightBegan = UserInputService.InputBegan:Connect(onFlightInputBegan)
    local flightEnded = UserInputService.InputEnded:Connect(onFlightInputEnded)
    table.insert(flightConns, flightBegan)
    table.insert(flightConns, flightEnded)
end

local function stopFlight()
    if not isFlying then return end
    
    isFlying = false

    flightSpeedValue = walkSpeed

    if antiGravityForce then
        antiGravityForce:Destroy()
        antiGravityForce = nil
    end

    -- Restaurar estado previo al vuelo
    if preFlightWalkSpeed ~= nil then
        walkSpeed = preFlightWalkSpeed
        isWalkSpeedActive = preFlightIsSpeedActive
    end
    preFlightWalkSpeed = nil
    preFlightIsSpeedActive = nil

    humanoid.AutoRotate = true
    humanoid.PlatformStand = false
    flightEmoteActive = false
    
    -- Forzar estado de caída solo si no estamos sentados (evita romper vehículos)
    if not humanoid.Sit and not humanoid.SeatPart then
        humanoid:ChangeState(Enum.HumanoidStateType.Freefall)
    else
        humanoid:ChangeState(Enum.HumanoidStateType.Seated)
    end
    if hrp and not humanoid.SeatPart then
        hrp.AssemblyLinearVelocity = Vector3.new(0, -1, 0)
    end
    
    stopFlightAudio()
    stopFlightAnimation()
    stopR6FlightAnims()
    r6ForwardHold = 0
    
    if flyGyro then flyGyro:Destroy() flyGyro = nil end
    if flyVelocity then flyVelocity:Destroy() flyVelocity = nil end
    if flyPos then flyPos:Destroy() flyPos = nil end
    
    for _, conn in ipairs(flightConns) do
        if conn and conn.Connected then conn:Disconnect() end
    end
    flightConns = {}
    flightMoveState = {forward = 0, backward = 0, left = 0, right = 0, up = 0, down = 0}
    currentFlightCF = nil
    flightCurrentRoll = 0
end

local function toggleFlight()
    if isFlying then
        stopFlight()
    else
        startFlight()
    end
    updateMainUI()
end

-- ============ REACCIÓN INSTANTÁNEA (ULTRA INSTINCT) ============

local function performInstantDodge(attackerObject, forceBehind)
    if tick() - InstantReaction.LastDodge < InstantReaction.Cooldown or not hrp or not attackerObject then return end

    -- Mejora Inteligente: No esquivar si el atacante no está mirando hacia nosotros
    if attackerObject:IsA("Model") and attackerObject:FindFirstChild("HumanoidRootPart") then
        local attackerHrp = attackerObject.HumanoidRootPart
        local toMe = (hrp.Position - attackerHrp.Position).Unit
        if attackerHrp.CFrame.LookVector:Dot(toMe) < 0.5 then
            return -- El atacante está mirando hacia otro lado, no es una amenaza directa
        end
    end

    InstantReaction.LastDodge = tick()
    InstantReaction.DodgeActive = true
    
    local enemyHrp = attackerObject:IsA("Model") and attackerObject:FindFirstChild("HumanoidRootPart") or (attackerObject:IsA("BasePart") and attackerObject)
    local enemyPos = enemyHrp and enemyHrp.Position or (hrp.Position + hrp.CFrame.LookVector * 10)
    
    -- IA de Posicionamiento: Si es una habilidad pesada, buscamos la espalda del enemigo
    local targetPos
    if forceBehind and enemyHrp then
        targetPos = enemyHrp.Position - (enemyHrp.CFrame.LookVector * 8) + Vector3.new(0, 2, 0)
    else
        local directions = {hrp.CFrame.RightVector, -hrp.CFrame.RightVector, -hrp.CFrame.LookVector}
        local chosenDir = directions[math.random(1, #directions)]
        targetPos = hrp.Position + (chosenDir * 22)
    end

    -- Detener ataques propios para priorizar la supervivencia
    for _, track in ipairs(humanoid:GetPlayingAnimationTracks()) do
        if track.Priority.Value >= Enum.AnimationPriority.Action.Value and track.Priority.Value < Enum.AnimationPriority.Action4.Value then
            track:Stop(0.05)
        end
    end

    -- Reproducir Animación de Esquiva Universal (R6/R15)
    local dodgeAnimId = isR6 and 248336294 or 104687069461693
    local anim = Instance.new("Animation")
    anim.AnimationId = "rbxassetid://" .. dodgeAnimId
    local track = (humanoid:FindFirstChildOfClass("Animator") or humanoid):LoadAnimation(anim)
    track.Priority = Enum.AnimationPriority.Action4
    track:Play()
    track:AdjustSpeed(2.2) 
    task.delay(0.4, function() track:Stop(0.1) anim:Destroy() InstantReaction.DodgeActive = false end)

    -- Efecto de Sonido (UI Dodge)
    local sound = Instance.new("Sound", hrp)
    sound.SoundId = "rbxassetid://6831034832" 
    sound.Volume = 2
    sound:Play()
    game:GetService("Debris"):AddItem(sound, 1)

    -- Desactivar colisiones temporalmente para atravesar hitboxes sin daño
    local parts = {}
    for _, p in ipairs(character:GetDescendants()) do
        if p:IsA("BasePart") then parts[p] = p.CanCollide; p.CanCollide = false end
    end

    -- Salto de CFrame Instantáneo (Bypass de Latencia)
    -- En lugar de un Tween, movemos el HRP de inmediato para que el servidor deje de detectar el hitbox
    hrp.CFrame = CFrame.lookAt(targetPos, enemyPos)
    hrp.AssemblyLinearVelocity = Vector3.new(0, 0, 0) -- Mata la inercia para evitar "rubberbanding"

    task.delay(0.1, function()
        for p, originalCollide in pairs(parts) do if p and p.Parent then p.CanCollide = originalCollide end end
    end)
end

local function updateInstantReaction()
    if not InstantReaction.Active or not hrp then return end

    local myPos = hrp.Position
    local myVel = hrp.AssemblyLinearVelocity

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character then
            local char = otherPlayer.Character
            local eHum = char:FindFirstChildOfClass("Humanoid")
            local eHrp = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso")

            if eHrp and eHum then
                local ePos = eHrp.Position
                local eVel = eHrp.AssemblyLinearVelocity
                local toMe = (myPos - ePos)
                local dist = toMe.Magnitude

                if dist < 60 then 
                    -- Análisis Cinético de Proyección
                    local relativeVel = eVel
                    local velocityDot = relativeVel.Unit:Dot(toMe.Unit)
                    
                    -- Detección de Dash/Aceleración Agresiva
                    local isDashingTowardsMe = (eVel.Magnitude > 55) and (velocityDot > 0.8)
                    
                    -- Predicción de Impacto (TTC) - Umbral de 0.35s para mayor margen de seguridad
                    local isComingFast = (velocityDot > 0.7) and (dist / math.max(relativeVel.Magnitude, 1) < 0.4)
                    
                    -- Escaneo de estado de combate (Detección de inicio de animación)
                    local isExecutingAction = false
                    local priorityLevel = 0
                    for _, track in ipairs(eHum:GetPlayingAnimationTracks()) do
                        if track.Priority.Value >= Enum.AnimationPriority.Action.Value and track.WeightTarget > 0.1 then
                            local animId = track.Animation.AnimationId:match("%d+")
                            if not managedAnimationIds[animId] then
                                isExecutingAction = true
                                priorityLevel = track.Priority.Value
                                break
                            end
                        end
                    end

                    -- Disparador Inteligente
                    if isDashingTowardsMe or (isExecutingAction and (isComingFast or dist < 14)) then
                        -- Si es una habilidad pesada o un dash, forzamos esquiva por detrás
                        local forceBehind = (priorityLevel >= Enum.AnimationPriority.Action3.Value) or isDashingTowardsMe
                        performInstantDodge(char, forceBehind)
                        return
                    end
                end
            end
        end
    end

    -- 2. ESCANEO DE PROYECTILES (Universal)
    for _, obj in ipairs(Workspace:GetChildren()) do
        if obj:IsA("BasePart") and obj.AssemblyLinearVelocity.Magnitude > 40 then
            local dist = (hrp.Position - obj.Position).Magnitude
            if dist < 30 then
                local toMe = (hrp.Position - obj.Position).Unit
                if obj.AssemblyLinearVelocity.Unit:Dot(toMe) > 0.8 then
                    performInstantDodge(obj, false)
                    return
                end
            end
        end
    end
end

-- ============ FUNCIONES BASE ============

local function connect(event, func)
    local conn = event:Connect(func)
    table.insert(allConnections, conn)
    return conn
end

local function startReplicationLoop()
    task.spawn(function()
        while scriptActive do
            pcall(function()
                settings().Physics.AllowSleep = false
                settings().Physics.PhysicsEnvironmentalThrottle = Enum.EnviromentalPhysicsThrottle.Disabled
                if sethiddenproperty then
                    sethiddenproperty(Players.LocalPlayer, "SimulationRadius", math.huge)
                    sethiddenproperty(Players.LocalPlayer, "MaxSimulationRadius", math.huge)
                elseif setsimulationradius then
                    setsimulationradius(math.huge, math.huge)
                end
            end)
            task.wait(0.1)
        end
    end)
end

local function fadeAudio(sound, targetVolume, duration)
    if not scriptActive or not sound then return end
    local startVolume = sound.Volume
    local startTime = tick()
    local fadeConnection
    fadeConnection = RunService.Heartbeat:Connect(function()
        if not scriptActive then
            fadeConnection:Disconnect()
            return
        end
        local elapsed = tick() - startTime
        local progress = math.min(elapsed / duration, 1)
        sound.Volume = startVolume + (targetVolume - startVolume) * progress
        if progress >= 1 then
            fadeConnection:Disconnect()
        end
    end)
end

-- ============ ANIMACIONES ============

playAnimation = function(animId, startTime, speed, loop)
    if not scriptActive then return end
    if not character or not humanoid then return end
    
    local animIdStr = "rbxassetid://" .. tostring(animId)
    
    if currentAnimTrack and currentAnimTrack.Animation.AnimationId == animIdStr then
        if speed then currentAnimTrack:AdjustSpeed(speed) end
        if not currentAnimTrack.IsPlaying then currentAnimTrack:Play(0.1) end
        return
    end

    if currentAnimTrack then
        currentAnimTrack:Stop(0.2)
    end

    local track = animationTracks[animId]
    if not track then
        local anim = Instance.new("Animation")
        anim.AnimationId = animIdStr
        local animator = humanoid:FindFirstChildOfClass("Animator")
        if animator then
            track = animator:LoadAnimation(anim)
        else
            track = humanoid:LoadAnimation(anim)
        end
        track.Priority = Enum.AnimationPriority.Action
        animationTracks[animId] = track
    end

    currentAnimTrack = track
    currentAnimTrack.Looped = (loop == nil and true or loop)
    currentAnimTrack.Priority = Enum.AnimationPriority.Action
    
    for _, t in ipairs(humanoid:GetPlayingAnimationTracks()) do
        local id = t.Animation.AnimationId:match("%d+")
        if t ~= currentAnimTrack and t ~= SuperStrength.AnimTrack and t ~= mirageSpeedEmoteTrack and id and managedAnimationIds[id] then
            t:Stop(0.1)
        end
    end
    currentAnimTrack:Play(0.1, 1, speed or 1)
    if startTime and startTime > 0 then currentAnimTrack.TimePosition = startTime end

    if hrp then
        hrp.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
        lastWalkStationaryPos = hrp.Position
    end

    -- Lógica de Audio Sincronizado
    if currentEmoteSound then
        currentEmoteSound:Stop()
        currentEmoteSound:Destroy()
        currentEmoteSound = nil
    end

    local animIdNum = tostring(animId)
    local soundId = nil
    local duration = nil

    if animIdNum == "124200992648318" then -- Emote 24
        soundId = "105607698428277"
        duration = 6.6
    elseif animIdNum == "113375965758912" then -- Emote 25
        soundId = "93810693712746"
    elseif animIdNum == "83766124558950" then -- Emote 26
        soundId = nil -- Se quita el sonido para el emote 26
    elseif animIdNum == "133394554631338" then -- Emote 27 (nuevo audio)
        soundId = "132792243719690"
    elseif animIdNum == "122599479076921" then -- Emote 28 (nuevo audio)
        soundId = "140733597017773"
    elseif animIdNum == "130196652446769" then -- Emote 29 (nuevo audio)
        soundId = "140504265985079"
    elseif animIdNum == "117722192552703" then -- Emote 23 (In Love)
        soundId = "131562947723376"
    elseif animIdNum == "119822440796584" then
        duration = 3
    elseif animIdNum == "130264584381746" then -- Nuevo emote solicitado
        soundId = "110847301842528"
    elseif animIdNum == "99873676774268" then -- Emote 35
        soundId = "139780189764790"
    elseif animIdNum == "101409107532852" then -- Emote 36
        soundId = "107781059281474"
    end

    local sound = nil
    if soundId then
        if not muteEmotes or animIdNum == "124200992648318" then
            sound = Instance.new("Sound")
            sound.Name = "SpecialEmoteSound"
            sound.SoundId = "rbxassetid://" .. soundId
            sound.Volume = 1
            sound.Looped = true
            sound.Parent = hrp or character
            sound:Play()
            currentEmoteSound = sound
            sound.Ended:Once(function() if currentEmoteSound == sound then sound:Destroy() currentEmoteSound = nil end end)
        end
    end

    if duration then
        task.delay(duration, function()
            if currentAnimTrack and currentAnimTrack.Animation.AnimationId == animIdStr then
                currentAnimTrack:Stop(0.5)
                currentAnimTrack = nil
            end
            if sound and currentEmoteSound == sound then sound:Destroy() currentEmoteSound = nil end
        end)
    end
end

local function stopAnimation()
    if not humanoid then return end
    if currentAnimTrack then
        currentAnimTrack:Stop(0.1)
        currentAnimTrack = nil
    end
    if currentEmoteSound then
        currentEmoteSound:Stop()
        currentEmoteSound:Destroy()
        currentEmoteSound = nil
    end
end

-- ============ OTRAS FUNCIONES ============

local function stopNoclip()
    if noclipConnection then
        noclipConnection:Disconnect()
        noclipConnection = nil
    end
end

local function startNoclip()
    stopNoclip() 
    noclipConnection = RunService.Stepped:Connect(function()
        if not scriptActive or not isInvisibilityActive or not character or not character.Parent then
            stopNoclip()
            return
        end
        for _, part in ipairs(character:GetDescendants()) do
            if part:IsA("BasePart") and part.CanCollide then
                part.CanCollide = false
            end
        end
    end)
    table.insert(allConnections, noclipConnection)
end

local function stopAntiTP()
    isAntiTPActive = false
    for _, conn in ipairs(antiTPConnections) do
        if conn then conn:Disconnect() end
    end
    antiTPConnections = {}
end

local function stopAntiFall()
    isAntiFallActive = false
    for _, conn in ipairs(antiFallConnections) do
        if conn then conn:Disconnect() end
    end
    antiFallConnections = {}
end

local function stopAntiFling()
    isAntiFlingActive = false
    if antiFlingConnection then
        antiFlingConnection:Disconnect()
        antiFlingConnection = nil
    end
end

local function disableFootstepSounds()
    if not character then return end
    for _, descendant in ipairs(character:GetDescendants()) do
        if descendant:IsA("Sound") then
            local soundName = descendant.Name:lower()
            if soundName:find("foot") or soundName:find("step") or soundName:find("walk") or soundName:find("run") then
                if not originalSoundProperties[descendant] then
                    originalSoundProperties[descendant] = {
                        Volume = descendant.Volume,
                        Playing = descendant.Playing,
                        Looped = descendant.Looped
                    }
                end
                descendant.Volume = 0
                if descendant.Playing then
                    descendant:Stop()
                end
            end
        end
    end
    if humanoid then
        local animateScript = character:FindFirstChild("Animate")
        if animateScript then
            for _, child in ipairs(animateScript:GetDescendants()) do
                if child:IsA("Sound") then
                    if not originalSoundProperties[child] then
                        originalSoundProperties[child] = {
                            Volume = child.Volume,
                            Playing = child.Playing,
                            Looped = child.Looped
                        }
                    end
                    child.Volume = 0
                    if child.Playing then
                        child:Stop()
                    end
                end
            end
        end
    end
end

local function hideIdentity()
    if not scriptActive then return end
    task.spawn(function()
        pcall(function()
            if character then
                character.Name = "NPC"
                if humanoid then
                    humanoid.DisplayName = " "
                end
                for _, v in ipairs(character:GetDescendants()) do
                    if v:IsA("BillboardGui") or v:IsA("SurfaceGui") then
                        v:Destroy()
                    end
                end
            end
        end)
    end)
end

-- ============ HABILIDADES SECUNDARIAS ============

local function cleanupMirageSpeed()
    if mirageSpeedEmoteTrack then
        mirageSpeedEmoteTrack:Stop(0.1)
        mirageSpeedEmoteTrack = nil
    end
    isMirageSpeedActive = false
end

local function playMirageSpeedEmote()
    if not scriptActive then return end
    if not humanoid then return end
    
    if mirageSpeedEmoteTrack then
        mirageSpeedEmoteTrack:Stop(0.1)
        mirageSpeedEmoteTrack = nil
    end
    
    if currentAnimTrack then
        currentAnimTrack:Stop(0.1)
        currentAnimTrack = nil
    end
    
    local MIRAGE_SPEED_EMOTE_ID = 84043660421785
    local anim = Instance.new("Animation")
    anim.AnimationId = "rbxassetid://" .. tostring(MIRAGE_SPEED_EMOTE_ID)
    
    mirageSpeedEmoteTrack = humanoid:LoadAnimation(anim)
    mirageSpeedEmoteTrack.Priority = Enum.AnimationPriority.Action4
    mirageSpeedEmoteTrack.Looped = true
    mirageSpeedEmoteTrack:Play()
    animationTracks["mirage_speed"] = mirageSpeedEmoteTrack
end

local function stopMirageSpeedEmote()
    if mirageSpeedEmoteTrack then
        mirageSpeedEmoteTrack:Stop(0.1)
        mirageSpeedEmoteTrack = nil
        animationTracks["mirage_speed"] = nil
    end
end

local function toggleMirageSpeed()
    if not scriptActive then return end
    if isMirageSpeedActive then
        stopMirageSpeedEmote()
        isMirageSpeedActive = false
    else
        playMirageSpeedEmote()
        isMirageSpeedActive = true
    end
end

local function startSuperJump()
    if not scriptActive or not humanoid then return end
    isSuperJumpActive = true
    humanoid.UseJumpPower = true
    humanoid.JumpPower = superJumpPower
end

local function startSuperSaltoCharge()
    if not humanoid or isSuperSaltoCharged then return end
    isSuperSaltoCharged = true
    superSaltoChargeTime = tick()
    
    local anim = Instance.new("Animation")
    anim.AnimationId = "rbxassetid://121288138217304"
    superSaltoTrack = humanoid:LoadAnimation(anim)
    superSaltoTrack.Priority = Enum.AnimationPriority.Action4
    superSaltoTrack:Play()
end

local function releaseSuperSalto()
    if not isSuperSaltoCharged then return end
    local duration = tick() - superSaltoChargeTime
    isSuperSaltoCharged = false
    if superSaltoTrack then superSaltoTrack:Stop(0.1) superSaltoTrack = nil end
    
    local force = math.min(180, 50 + (duration * 65))
    if hrp then
        hrp.AssemblyLinearVelocity = Vector3.new(hrp.AssemblyLinearVelocity.X, force, hrp.AssemblyLinearVelocity.Z)
        humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
    end
    if jumpAnimTrack then 
        jumpAnimTrack:Play(0.1) 
    else
        local anim = Instance.new("Animation")
        anim.AnimationId = "rbxassetid://" .. ANIMATIONS.Jump
        jumpAnimTrack = humanoid:LoadAnimation(anim)
        jumpAnimTrack.Priority = Enum.AnimationPriority.Action4
        jumpAnimTrack:Play(0.1)
    end
end

local function stopSuperJump()
    isSuperJumpActive = false
    if humanoid then
        humanoid.JumpPower = originalJumpPower
    end
end

local function setupSuperHearingAudio()
    if SuperHearing.Sound then
        pcall(function() SuperHearing.Sound:Stop() SuperHearing.Sound:Destroy() end)
    end
    SuperHearing.Sound = Instance.new("Sound")
    SuperHearing.Sound.SoundId = "rbxassetid://1846354746"
    SuperHearing.Sound.Name = "SuperHearingSound"
    SuperHearing.Sound.Looped = true
    SuperHearing.Sound.Volume = 0
    SuperHearing.Sound.Parent = hrp or character or SoundService
end

local function ensureSuperHearingHighlight(char, nameText, isNPC, active)
    if not char or not char.Parent then return end
    local dict = isNPC and SuperHearing.NPCHighlights or SuperHearing.PlayerHighlights
    local cacheDict = isNPC and SuperHearing.NPCCache or SuperHearing.PlayerCache
    local data = dict[char]
    
    if not data and active then
        local highlight = Instance.new("Highlight")
        highlight.Name = isNPC and "SuperHearingNPCHighlight" or "SuperHearingPlayerHighlight"
        highlight.FillColor = Color3.fromRGB(0, 120, 255)
        highlight.OutlineColor = Color3.fromRGB(0, 255, 255)
        highlight.FillTransparency = 0.15
        highlight.OutlineTransparency = 0
        highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        highlight.Parent = char
        
        local billboard = Instance.new("BillboardGui")
        billboard.Name = isNPC and "SuperHearingNPCName" or "SuperHearingPlayerName"
        billboard.Size = UDim2.new(0, 200, 0, 50)
        billboard.StudsOffset = Vector3.new(0, 3, 0)
        billboard.AlwaysOnTop = true
        local adornee = char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart") or char:FindFirstChildWhichIsA("BasePart")
        billboard.Adornee = adornee
        
        local textLabel = Instance.new("TextLabel")
        textLabel.Size = UDim2.new(1, 0, 1, 0)
        textLabel.BackgroundTransparency = 1
        textLabel.Text = nameText
        textLabel.TextColor3 = isNPC and Color3.fromRGB(255, 100, 100) or Color3.fromRGB(0, 150, 255)
        textLabel.TextSize = 14
        textLabel.Font = Enum.Font.GothamBold
        textLabel.Parent = billboard
        billboard.Parent = char
        
        data = {highlight = highlight, billboard = billboard}
        dict[char] = data
        cacheDict[char] = true
        if isNPC then SuperHearing.ActiveNPCs[char] = true end
    elseif data then
        data.highlight.Enabled = active
        data.billboard.Enabled = active
        cacheDict[char] = active
        if isNPC then SuperHearing.ActiveNPCs[char] = active end
    end
end

local function characterIsNoisy(char, hum, root)
    if not root then return false end
    return (hum and hum.Health > 0 and hum.MoveDirection.Magnitude > 0.1) or root.AssemblyLinearVelocity.Magnitude > 1
end

local function updateSuperHearing()
    if not scriptActive or not SuperHearing.Active then return end
    if not character or not hrp then return end
    local now = tick()
    if now - SuperHearing.LastUpdate < 0.1 then return end
    SuperHearing.LastUpdate = now
    
    local detectedAny = false
    
    -- Jugadores
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player then
            local otherChar = otherPlayer.Character
            if otherChar then
                local hum = otherChar:FindFirstChild("Humanoid")
                local root = otherChar:FindFirstChild("HumanoidRootPart") or otherChar:FindFirstChildWhichIsA("BasePart")
                if hum and hum.Health > 0 and root then
                    local noisy = characterIsNoisy(otherChar, hum, root)
                    if noisy then detectedAny = true end
                    ensureSuperHearingHighlight(otherChar, otherPlayer.Name, false, noisy)
                else
                    -- Limpiar si el jugador murió o no tiene root
                    if SuperHearing.PlayerHighlights[otherChar] then
                        local d = SuperHearing.PlayerHighlights[otherChar]
                        if d.highlight then d.highlight:Destroy() end
                        if d.billboard then d.billboard:Destroy() end
                        SuperHearing.PlayerHighlights[otherChar] = nil
                    end
                end
            end
        end
    end

    -- NPCs (Escaneo cada 3 segundos)
    if now - SuperHearing.LastNPCScanTime > 3 then
        SuperHearing.LastNPCScanTime = now
        SuperHearing.CachedNPCList = {}
        for _, desc in ipairs(Workspace:GetDescendants()) do
            if desc:IsA("Humanoid") and desc.Parent and desc.Parent:IsA("Model") then
                local model = desc.Parent
                if not Players:GetPlayerFromCharacter(model) and model:FindFirstChildWhichIsA("BasePart") then
                    table.insert(SuperHearing.CachedNPCList, model)
                end
            end
        end
    end

    for _, model in ipairs(SuperHearing.CachedNPCList) do
        if model and model.Parent then
            local hum = model:FindFirstChild("Humanoid")
            local root = model:FindFirstChild("HumanoidRootPart") or model:FindFirstChildWhichIsA("BasePart")
            if hum and hum.Health > 0 and root then
                local noisy = characterIsNoisy(model, hum, root)
                if noisy then detectedAny = true end
                ensureSuperHearingHighlight(model, "NPC", true, noisy)
            else
                if SuperHearing.NPCHighlights[model] then
                    local d = SuperHearing.NPCHighlights[model]
                    if d.highlight then d.highlight:Destroy() end
                    if d.billboard then d.billboard:Destroy() end
                    SuperHearing.NPCHighlights[model] = nil
                end
            end
        end
    end

    if SuperHearing.Sound then
        if detectedAny then
            if not SuperHearing.Sound.Playing then SuperHearing.Sound:Play() end
            if SuperHearing.Sound.Volume < SUPER_HEARING_MAX_VOLUME then
                fadeAudio(SuperHearing.Sound, SUPER_HEARING_MAX_VOLUME, 0.15)
            end
        else
            if SuperHearing.Sound.Volume > 0 then
                fadeAudio(SuperHearing.Sound, 0, 0.3)
            end
        end
    end
end

local function cleanupSuperHearing()
    if SuperHearing.CleanupTimer then task.cancel(SuperHearing.CleanupTimer) end
    if SuperHearing.Timer then task.cancel(SuperHearing.Timer) end
    if SuperHearing.Blur then SuperHearing.Blur:Destroy() end
    if SuperHearing.ColorCorrection then SuperHearing.ColorCorrection:Destroy() end
    if SuperHearing.Echo then SuperHearing.Echo:Destroy() end
    if SuperHearing.Reverb then SuperHearing.Reverb:Destroy() end
    if SuperHearing.Heartbeat then pcall(function() SuperHearing.Heartbeat:Disconnect() end) SuperHearing.Heartbeat = nil end
    if SuperHearing.Sound then pcall(function() SuperHearing.Sound:Stop() SuperHearing.Sound:Destroy() end) end
    for char, data in pairs(SuperHearing.PlayerHighlights) do pcall(function() data.highlight:Destroy() data.billboard:Destroy() end) end
    for char, data in pairs(SuperHearing.NPCHighlights) do pcall(function() data.highlight:Destroy() data.billboard:Destroy() end) end
    SuperHearing.PlayerHighlights = {}
    SuperHearing.NPCHighlights = {}
    SuperHearing.PlayerCache = {}
    SuperHearing.NPCCache = {}
    SuperHearing.ActiveNPCs = {}
end

local function stopSuperHearing()
    if not SuperHearing.Active then return end
    SuperHearing.Active = false

    -- Ocultar marcadores inmediatamente para evitar que sigan visibles mientras se desvanece la pantalla
    for _, data in pairs(SuperHearing.PlayerHighlights) do
        if data.highlight then data.highlight.Enabled = false end
        if data.billboard then data.billboard.Enabled = false end
    end
    for _, data in pairs(SuperHearing.NPCHighlights) do
        if data.highlight then data.highlight.Enabled = false end
        if data.billboard then data.billboard.Enabled = false end
    end

    local tweenInfo = TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    
    if SuperHearing.Blur then
        TweenService:Create(SuperHearing.Blur, tweenInfo, {Size = 0}):Play()
    end
    
    if SuperHearing.ColorCorrection then
        TweenService:Create(SuperHearing.ColorCorrection, tweenInfo, {Saturation = 0, Contrast = 0}):Play()
    end
    
    if SuperHearing.Echo then
        TweenService:Create(SuperHearing.Echo, tweenInfo, {WetLevel = -80}):Play()
    end
    
    if SuperHearing.Reverb then
        SuperHearing.Reverb.Enabled = false
    end
    
    if SuperHearing.Sound then fadeAudio(SuperHearing.Sound, 0, 0.5) end
    
    if SuperHearing.CleanupTimer then task.cancel(SuperHearing.CleanupTimer) end
    SuperHearing.CleanupTimer = task.delay(0.5, cleanupSuperHearing)
end

local function startSuperHearing()
    if not scriptActive or not character then return end
    if SuperHearing.Active then return end
    if SuperHearing.CleanupTimer then task.cancel(SuperHearing.CleanupTimer) end
    SuperHearing.Active = true
    setupSuperHearingAudio()
    if not SuperHearing.Blur then
        SuperHearing.Blur = Instance.new("BlurEffect")
        SuperHearing.Blur.Name = "SuperHearingBlur"
        SuperHearing.Blur.Parent = Lighting
    end
    SuperHearing.Blur.Size = 0
    SuperHearing.Blur.Enabled = true

    if not SuperHearing.ColorCorrection then
        SuperHearing.ColorCorrection = Instance.new("ColorCorrectionEffect")
        SuperHearing.ColorCorrection.Name = "SuperHearingCC"
        SuperHearing.ColorCorrection.Parent = Lighting
    end
    SuperHearing.ColorCorrection.Saturation = 0
    SuperHearing.ColorCorrection.Contrast = 0
    SuperHearing.ColorCorrection.Enabled = true

    if not SuperHearing.Echo then
        SuperHearing.Echo = Instance.new("EchoSoundEffect")
        SuperHearing.Echo.Name = "SuperHearingEcho"
        SuperHearing.Echo.Parent = SoundService
        SuperHearing.Echo.Delay = 0.4
        SuperHearing.Echo.Feedback = 0.5
        SuperHearing.Echo.WetLevel = -80
    end
    SuperHearing.Echo.Enabled = true

    if not SuperHearing.Reverb then
        SuperHearing.Reverb = Instance.new("ReverbSoundEffect")
        SuperHearing.Reverb.Name = "SuperHearingReverb"
        SuperHearing.Reverb.Parent = SoundService
        SuperHearing.Reverb.DecayTime = 2
        SuperHearing.Reverb.Density = 0.7
    end
    SuperHearing.Reverb.Enabled = true

    local tweenInfo = TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    TweenService:Create(SuperHearing.Blur, tweenInfo, {Size = 9}):Play()
    TweenService:Create(SuperHearing.ColorCorrection, tweenInfo, {Saturation = -0.5, Contrast = 0.2}):Play()
    TweenService:Create(SuperHearing.Echo, tweenInfo, {WetLevel = 0}):Play()

    SuperHearing.Heartbeat = RunService.Heartbeat:Connect(updateSuperHearing)
    table.insert(allConnections, SuperHearing.Heartbeat)
    if SuperHearing.Timer then task.cancel(SuperHearing.Timer) end
    SuperHearing.Timer = task.delay(SUPER_HEARING_DURATION, function()
        SuperHearing.Timer = nil
        stopSuperHearing()
    end)
end

local function ensureThermalHighlight(char, isItem)
    if not char or not char.Parent then return end
    if ThermalVision.Highlights[char] and ThermalVision.Highlights[char].Parent then
        ThermalVision.Highlights[char].Enabled = true
        return
    end
    local highlight = Instance.new("Highlight")
    highlight.Name = "ThermalVisionHighlight"
    highlight.FillColor = Color3.fromRGB(255, 0, 0) -- Rojo puro
    highlight.OutlineColor = Color3.fromRGB(50, 205, 50)
    highlight.FillTransparency = 0.3
    highlight.OutlineTransparency = 0
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.Parent = char
    ThermalVision.Highlights[char] = highlight
end

local function updateThermalVision()
    if not scriptActive or not ThermalVision.Active then return end
    
    local now = tick()
    -- Escaneo de objetos especiales (Libros y Fusibles de DOORS) cada 2 segundos
    if now - ThermalVision.LastItemScanTime > 2 then
        ThermalVision.LastItemScanTime = now
        ThermalVision.CachedItemList = {}
        ThermalVision.CachedNPCList = {}
        
        -- Nombres exactos de ítems para evitar resaltar muebles (whitelist estricta)
        local itemWhitelist = {
            ["key"] = true, ["livehintbook"] = true, ["lever"] = true, ["switch"] = true,
            ["breaker"] = true, ["breakerbank"] = true, ["fuse"] = true, ["fuseitem"] = true, ["livefuse"] = true,
            ["breakerbox"] = true, ["electricalbox"] = true, ["gold"] = true,
            ["goldpile"] = true, ["lighter"] = true, ["flashlight"] = true,
            ["crucifix"] = true, ["lockpick"] = true, ["candle"] = true,
            ["battery"] = true, ["shears"] = true, ["vitamins"] = true,
            ["bandage"] = true
        }

        for _, v in ipairs(Workspace:GetDescendants()) do
            local n = v.Name:lower()
            
            -- 1. JUGADORES Y NPCs (Mantenemos el fondo azul y detección de entidades)
            if v:IsA("Humanoid") and v.Parent and v.Parent:IsA("Model") then
                table.insert(ThermalVision.CachedNPCList, v.Parent)
            elseif v:IsA("Model") and (n:find("rush") or n:find("ambush") or n:find("seek") or 
                n:find("figure") or n:find("halt") or n:find("eyes") or n:find("screech") or 
                n:find("dupe") or n:find("guiding") or n:find("a-60") or n:find("a-120")) then
                table.insert(ThermalVision.CachedNPCList, v)
            
            -- 2. ÍTEMS (Solo si el nombre está en la whitelist estricta)
            elseif itemWhitelist[n] or n:find("fuse") or n:find("breaker") then
                table.insert(ThermalVision.CachedItemList, v)
            end
        end
    end

    -- Aplicar resaltado a todo lo cacheado
    for _, model in ipairs(ThermalVision.CachedNPCList) do
        if model and model.Parent then ensureThermalHighlight(model, false) end
    end
    for _, item in ipairs(ThermalVision.CachedItemList) do
        if item and item.Parent then ensureThermalHighlight(item, true) end
    end

    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= player then
            local char = p.Character
            if char and char:FindFirstChild("HumanoidRootPart") then
                ensureThermalHighlight(char, false)
            end
        end
    end
end

local function cleanupThermalVision()
    if not ThermalVision.Active then return end
    ThermalVision.Active = false
    if ThermalVision.ColorCorrection then ThermalVision.ColorCorrection:Destroy() end
    if ThermalVision.Connection then ThermalVision.Connection:Disconnect() end
    if ThermalVision.Heartbeat then ThermalVision.Heartbeat:Disconnect() end
    if ThermalVision.ChildAddedConnection then ThermalVision.ChildAddedConnection:Disconnect() end
    for char, highlight in pairs(ThermalVision.Highlights) do
        if highlight then highlight:Destroy() end
    end
    ThermalVision.Highlights = {}
    ThermalVision.CachedItemList = {}
    local l = Lighting
    if ThermalVision.OriginalLighting.Ambient then l.Ambient = ThermalVision.OriginalLighting.Ambient end
    if ThermalVision.OriginalLighting.OutdoorAmbient then l.OutdoorAmbient = ThermalVision.OriginalLighting.OutdoorAmbient end
    if ThermalVision.OriginalLighting.Brightness then l.Brightness = ThermalVision.OriginalLighting.Brightness end
    if ThermalVision.OriginalLighting.GlobalShadows ~= nil then l.GlobalShadows = ThermalVision.OriginalLighting.GlobalShadows end
    if ThermalVision.OriginalLighting.ClockTime then l.ClockTime = ThermalVision.OriginalLighting.ClockTime end
    ThermalVision.OriginalLighting = {}
end

local function toggleThermalVision()
    if not scriptActive then return end
    if ThermalVision.Active then
        cleanupThermalVision()
    else
        ThermalVision.Active = true
        local l = Lighting
        ThermalVision.OriginalLighting = {
            Ambient = l.Ambient,
            OutdoorAmbient = l.OutdoorAmbient,
            Brightness = l.Brightness,
            GlobalShadows = l.GlobalShadows,
            ClockTime = l.ClockTime
        }
        local cc = Instance.new("ColorCorrectionEffect")
        cc.Name = "ThermalVisionCC"
        cc.TintColor = Color3.fromRGB(0, 100, 255)
        cc.Contrast = 0.1
        cc.Saturation = -0.5
        cc.Parent = l
        ThermalVision.ColorCorrection = cc
        ThermalVision.Connection = RunService.RenderStepped:Connect(function()
            l.Ambient = Color3.new(1, 1, 1)
            l.OutdoorAmbient = Color3.new(1, 1, 1)
            l.Brightness = 2
            l.GlobalShadows = false
            l.ClockTime = 12
        end)
        ThermalVision.Heartbeat = RunService.Heartbeat:Connect(updateThermalVision)
    end
end

local function toggleInvisibility()
    if not scriptActive then return end
    isInvisibilityActive = not isInvisibilityActive
    local char = player.Character
    if not char then isInvisibilityActive = false return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    if isInvisibilityActive then
        local currentPos = root.CFrame

        -- activar noclip
        startNoclip()

        -- Guardar transparencias/colisiones para restaurar exacto
        invisibilitySeat = { saved = {} }
        for _, v in ipairs(char:GetDescendants()) do
            if v:IsA("BasePart") then
                invisibilitySeat.saved[v] = {
                    Transparency = v.Transparency,
                    CanCollide = v.CanCollide
                }
                v.Transparency = 1
                v.CanCollide = false
            end
        end

        -- pequeña cam offset aleatoria (sin tocar estados físicos)
        task.spawn(function()
            while isInvisibilityActive and scriptActive do
                if humanoid then
                    humanoid.CameraOffset = Vector3.new(
                        math.random(-15, 15)/100,
                        math.random(-15, 15)/100,
                        math.random(-15, 15)/100
                    )
                end
                task.wait(0.1)
            end
        end)

    else
        -- Restaurar transparencias/colisiones
        if invisibilitySeat and invisibilitySeat.saved then
            for inst, st in pairs(invisibilitySeat.saved) do
                if inst and inst.Parent then
                    pcall(function()
                        inst.Transparency = st.Transparency
                        inst.CanCollide = st.CanCollide
                    end)
                end
            end
        end

        if humanoid then
            pcall(function()
                humanoid.CameraOffset = Vector3.new(0,0,0)
            end)
        end

        if noclipConnection then noclipConnection:Disconnect() noclipConnection = nil end
        invisibilitySeat = nil
    end
end

local function startAntiTP()
    stopAntiTP()
    isAntiTPActive = true
    local hrp = character:FindFirstChild("HumanoidRootPart")
    local humanoid = character:FindFirstChild("Humanoid")
    if not hrp or not humanoid then return end
    local conn1 = RunService.Stepped:Connect(function()
        if not isAntiTPActive then
            stopAntiTP()
            return
        end
        -- Se elimina el reseteo de posición para permitir TPs del mapa/juego
    end)
    table.insert(antiTPConnections, conn1)
    table.insert(allConnections, conn1)
end

local function startAntiFall()
    stopAntiFall()
    isAntiFallActive = true
    local hrp = character:FindFirstChild("HumanoidRootPart")
    local humanoid = character:FindFirstChild("Humanoid")
    if not hrp or not humanoid then return end
    local lastGroundPosition = hrp.CFrame
    local lastGroundY = hrp.Position.Y
    local conn1 = RunService.Heartbeat:Connect(function()
        if humanoid.Health > 0 and isAntiFallActive then
            local currentY = hrp.Position.Y
            if humanoid.FloorMaterial ~= Enum.Material.Air then
                lastGroundPosition = hrp.CFrame
                lastGroundY = currentY
            end
            if lastGroundY - currentY > 15 then
                antiFallTeleporting = true
                hrp.CFrame = lastGroundPosition
                hrp.AssemblyLinearVelocity = Vector3.zero
                task.delay(0.1, function() antiFallTeleporting = false end)
            end
        end
    end)
    table.insert(antiFallConnections, conn1)
    table.insert(allConnections, conn1)
end

local function startAntiFling()
    stopAntiFling()
    isAntiFlingActive = true
    antiFlingConnection = RunService.Stepped:Connect(function()
        if not isAntiFlingActive or not character or not character.Parent then
            stopAntiFling()
            return
        end
        if hrp and hrp.Parent and not isFlying and not isWalkSpeedActive then -- NO resetear si estamos volando o con velocidad activa
            hrp.AssemblyLinearVelocity = Vector3.new(0,0,0)
            hrp.AssemblyAngularVelocity = Vector3.new(0,0,0)
        end
        for _, part in ipairs(character:GetDescendants()) do
            if part:IsA("BasePart") and part.CanCollide then
                part.CanCollide = false
            end
        end
    end)
    table.insert(allConnections, antiFlingConnection)
end

local function playSprintAnimation()
    if not scriptActive then return end
    if not humanoid then return end
    if not isSprintActive then return end
    if isCrouching then return end
    if isMirageSpeedActive then return end
    if isWalkSpeedActive then return end
    if sprintAnimTrack and sprintAnimTrack.IsPlaying then return end
    if sprintAnimTrack then sprintAnimTrack:Stop(0.1) end
    local anim = Instance.new("Animation")
    anim.AnimationId = "rbxassetid://138992096476836"
    sprintAnimTrack = humanoid:LoadAnimation(anim)
    sprintAnimTrack.Looped = true
    sprintAnimTrack.Priority = Enum.AnimationPriority.Action2
    sprintAnimTrack:Play()
    animationTracks["sprint_active"] = sprintAnimTrack
end

local function stopSprintAnimation()
    if sprintAnimTrack then
        sprintAnimTrack:Stop(0.1)
        sprintAnimTrack = nil
        animationTracks["sprint_active"] = nil
    end
end

local function toggleSprint()
    if not scriptActive then return end
    isSprintActive = not isSprintActive
    if isSprintActive then
        if humanoid then humanoid.WalkSpeed = 22 end
    else
        if humanoid then humanoid.WalkSpeed = isWalkSpeedActive and walkSpeed or 8 end
        stopSprintAnimation()
    end
end

-- ============ MENÚ SELECTOR UI ============

local function updateMainUI()
    if not modeText or not abilityText then return end
    
    -- Bloqueo visual para R6: No mostrar modo Emotes si es R6 (se maneja internamente)
    modeText.Text = (isR6 and currentMode == MODES.EMOTES) and "HABILIDADES" or modeNames[currentMode]

    local ability = abilities[currentAbilityIndex]
    local isSpeedType = (currentMode == MODES.ABILITIES and ability and (ability.key == "walkspeed" or ability.key == "gravjump"))
    
    if plusBtn and minusBtn then
        plusBtn.Visible = isSpeedType
        minusBtn.Visible = isSpeedType
    end

    local newText = ""
    if currentMode == MODES.ABILITIES and ability then
        local isActive = false
        if ability.key == "walkspeed" then isActive = isWalkSpeedActive
        elseif ability.key == "miragespeed" then isActive = isMirageSpeedActive
        elseif ability.key == "gravjump" then isActive = isGravJumpActive
        elseif ability.key == "supersalto" then isActive = (abilities[currentAbilityIndex].key == "supersalto" and isSuperSaltoCharged)
        elseif ability.key == "superhearing" then isActive = SuperHearing.Active
        elseif ability.key == "thermalvision" then isActive = ThermalVision.Active
        elseif ability.key == "molecularvibration" then isActive = isInvisibilityActive
        elseif ability.key == "flight" then isActive = isFlying
        elseif ability.key == "superstrength" then isActive = SuperStrength.Active
        elseif ability.key == "instantreaction" then isActive = InstantReaction.Active
        end

        local suffix = ""
        newText = ability.name .. suffix .. (isActive and " ✓" or "")
        
        -- Color especial para Super Agarre cuando está activo (Dorado)
        abilityText.TextColor3 = (ability.key == "superstrength" and isActive) and Color3.fromRGB(255, 215, 0) or Color3.fromRGB(255, 255, 255)
    elseif currentMode == MODES.EMOTES then
        if isFlying then
            newText = "Fly Emote " .. currentFlightEmoteIndex .. "/" .. #FLIGHT_EMOTES
        else
            newText = "Emote " .. currentAnimationIndex .. "/" .. #selectableAnimations
        end
    elseif currentMode == MODES.SETTINGS then
        newText = settingOptions[currentSettingIndex]
    end
    
    abilityText.Text = newText
end

local function nextSelection()
    if currentMode == MODES.ABILITIES then
        repeat
            currentAbilityIndex = currentAbilityIndex + 1
            if currentAbilityIndex > #abilities then currentAbilityIndex = 1 end
        until isAbilityAvailable(abilities[currentAbilityIndex].key)
    elseif currentMode == MODES.EMOTES then
        if isFlying then
            currentFlightEmoteIndex = currentFlightEmoteIndex + 1
            if currentFlightEmoteIndex > #FLIGHT_EMOTES then currentFlightEmoteIndex = 1 end
        else
            currentAnimationIndex = currentAnimationIndex + 1
            if currentAnimationIndex > #selectableAnimations then currentAnimationIndex = 1 end
        end
    elseif currentMode == MODES.SETTINGS then
        currentSettingIndex = currentSettingIndex + 1
        if currentSettingIndex > #settingOptions then currentSettingIndex = 1 end
        repeat
            currentSettingIndex = currentSettingIndex + 1
            if currentSettingIndex > #settingOptions then currentSettingIndex = 1 end
        until isR6 or not settingOptions[currentSettingIndex]:find("R6 FLY")
    end
    updateMainUI()
    
    local originalColor = abilityText.TextColor3
    abilityText.TextColor3 = Color3.fromRGB(255, 255, 100)
    task.wait(0.08)
    abilityText.TextColor3 = originalColor
end

local function previousSelection()
    if currentMode == MODES.ABILITIES then
        repeat
            currentAbilityIndex = currentAbilityIndex - 1
            if currentAbilityIndex < 1 then currentAbilityIndex = #abilities end
        until isAbilityAvailable(abilities[currentAbilityIndex].key)
    elseif currentMode == MODES.EMOTES then
        if isFlying then
            currentFlightEmoteIndex = currentFlightEmoteIndex - 1
            if currentFlightEmoteIndex < 1 then currentFlightEmoteIndex = #FLIGHT_EMOTES end
        else
            currentAnimationIndex = currentAnimationIndex - 1
            if currentAnimationIndex < 1 then currentAnimationIndex = #selectableAnimations end
        end
    elseif currentMode == MODES.SETTINGS then
        currentSettingIndex = currentSettingIndex - 1
        if currentSettingIndex < 1 then currentSettingIndex = #settingOptions end
        repeat
            currentSettingIndex = currentSettingIndex - 1
            if currentSettingIndex < 1 then currentSettingIndex = #settingOptions end
        until isR6 or not settingOptions[currentSettingIndex]:find("R6 FLY")
    end
    updateMainUI()
    
    local originalColor = abilityText.TextColor3
    abilityText.TextColor3 = Color3.fromRGB(255, 200, 100)
    task.wait(0.08)
    abilityText.TextColor3 = originalColor
end

local function nextMode()
    if currentMode == MODES.ABILITIES then
        currentMode = MODES.EMOTES
    else
        currentMode = MODES.ABILITIES
    end
    updateMainUI()
    
    local originalColor = modeText.TextColor3
    modeText.TextColor3 = Color3.fromRGB(255, 255, 100)
    if settingsPanel then
        if currentMode == MODES.SETTINGS then
            settingsPanel.Visible = true
            TweenService:Create(settingsPanel, TweenInfo.new(0.2), {BackgroundTransparency = 0.15}):Play()
        else
            TweenService:Create(settingsPanel, TweenInfo.new(0.2), {BackgroundTransparency = 1}):Play()
            task.wait(0.2)
            settingsPanel.Visible = false
        end
    end
    task.wait(0.08)
    modeText.TextColor3 = originalColor
end

local function previousMode()
    if currentMode == MODES.EMOTES then
        currentMode = MODES.ABILITIES
    else
        currentMode = MODES.EMOTES
    end
    updateMainUI()
    
    local originalColor = modeText.TextColor3
    modeText.TextColor3 = Color3.fromRGB(255, 255, 100)
    if settingsPanel then
        if currentMode == MODES.SETTINGS then
            settingsPanel.Visible = true
            TweenService:Create(settingsPanel, TweenInfo.new(0.2), {BackgroundTransparency = 0.15}):Play()
        else
            TweenService:Create(settingsPanel, TweenInfo.new(0.2), {BackgroundTransparency = 1}):Play()
            task.wait(0.2)
            settingsPanel.Visible = false
        end
    end
    task.wait(0.08)
    modeText.TextColor3 = originalColor
end

local function activateCurrent()
    if currentMode == MODES.ABILITIES then
        local ability = abilities[currentAbilityIndex]
        if ability and toggleAbility then
            -- Si es Super Agarre y ya tenemos algo agarrado, lanzamos el objeto (esto ya desactiva la habilidad)
            if ability.key == "superstrength" and SuperStrength.Active and SuperStrength.GrabbedData then
                releaseGrabbedObject(true)
            else
                toggleAbility(ability.key, 1)
            end
        end
    elseif currentMode == MODES.EMOTES then
        if isFlying then
            local animId = FLIGHT_EMOTES[currentFlightEmoteIndex]
            local animIdStr = "rbxassetid://" .. tostring(animId)
            if flightEmoteActive and currentFlightAnimTrack and currentFlightAnimTrack.Animation.AnimationId == animIdStr then
                flightEmoteActive = false
            else
                flightEmoteActive = true
                playFlightAnimation(animId, 0, 1)
            end
        else
            local animId = selectableAnimations[currentAnimationIndex]
            local animIdStr = "rbxassetid://" .. tostring(animId)
            if currentAnimTrack and currentAnimTrack.IsPlaying and currentAnimTrack.Animation.AnimationId == animIdStr then
                stopAnimation()
            else
                playAnimation(animId, 0, 1, true)
            end
        end
    elseif currentMode == MODES.SETTINGS then
        local option = settingOptions[currentSettingIndex]
        if option:find("ESTATICO") then
            flightStaticRotation = not flightStaticRotation
            settingOptions[currentSettingIndex] = "ESTATICO: " .. (flightStaticRotation and "ON" or "OFF")
        elseif option:find("SILENCIAR") then
            muteEmotes = not muteEmotes
            settingOptions[currentSettingIndex] = "SILENCIAR: " .. (muteEmotes and "ON" or "OFF")
        elseif option:find("NOSIT") then
            noSitActive = not noSitActive
            settingOptions[currentSettingIndex] = "NOSIT: " .. (noSitActive and "ON" or "OFF")
            if humanoid then
                humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, not noSitActive)
                if noSitActive then humanoid.Sit = false end
            end
        elseif option:find("R6 FLY") then
            r6FlightIdleMode = (r6FlightIdleMode == 1) and 2 or 1
            settingOptions[currentSettingIndex] = "R6 FLY: " .. r6FlightIdleMode
            if isFlying and isR6 then
                setupR6FlightAnims()
            end
        elseif option == "DESTRUIR" then
            scriptActive = false
            cleanupEverything(true)
        end
    end
    
    updateMainUI()

    local originalColor = logoImage.ImageColor3
    TweenService:Create(logoImage, TweenInfo.new(0.1), {ImageColor3 = Color3.fromRGB(0, 162, 255)}):Play()
    task.wait(0.1)
    TweenService:Create(logoImage, TweenInfo.new(0.1), {ImageColor3 = originalColor}):Play()
end

local function openMenu()
    if menuOpen then return end
    menuOpen = true
    
    if mainPanel then
        mainPanel.Visible = true
        TweenService:Create(mainPanel, TweenInfo.new(0.2), {BackgroundTransparency = 0.3, Position = UDim2.new(1, -210, 0, 10)}):Play()
    end
end

local function closeMenu()
    if not menuOpen then return end
    menuOpen = false
    
    if settingsPanel then
        settingsPanel.Visible = false
    end
    if mainPanel then
        TweenService:Create(mainPanel, TweenInfo.new(0.2), {BackgroundTransparency = 1, Position = UDim2.new(1, -210, 0, -60)}):Play()
        task.wait(0.2)
        mainPanel.Visible = false
    end
end

local function createUI()
    if screenGui then screenGui:Destroy() end
    
    screenGui = Instance.new("ScreenGui")
    screenGui.Name = "SuperGirlUI"
    screenGui.ResetOnSpawn = false
    screenGui.IgnoreGuiInset = true
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
    screenGui.Parent = CoreGui
    
    mainPanel = Instance.new("Frame")
    mainPanel.Name = "MainPanel"
    mainPanel.Size = UDim2.new(0, 220, 0, 85)
    mainPanel.Position = UDim2.new(1, -210, 0, -60)
    mainPanel.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    mainPanel.BackgroundTransparency = 1
    mainPanel.BorderSizePixel = 0
    mainPanel.Visible = false
    mainPanel.Parent = screenGui
    
    local panelCorner = Instance.new("UICorner")
    panelCorner.CornerRadius = UDim.new(0, 12)
    panelCorner.Parent = mainPanel
    
    local panelStroke = Instance.new("UIStroke")
    panelStroke.Thickness = 2
    panelStroke.Color = Color3.fromRGB(255, 215, 0)
    panelStroke.Parent = mainPanel

    local closeX = Instance.new("TextLabel")
    closeX.Name = "CloseVisual"
    closeX.Size = UDim2.new(0, 20, 0, 20)
    closeX.Position = UDim2.new(1, -25, 0, 5)
    closeX.BackgroundTransparency = 1
    closeX.Text = "X"
    closeX.TextColor3 = Color3.fromRGB(255, 100, 100)
    closeX.Font = Enum.Font.GothamBold
    closeX.TextSize = 14
    closeX.ZIndex = 10
    closeX.Parent = mainPanel

    local closeBtn = Instance.new("TextButton")
    closeBtn.Name = "CloseButton"
    closeBtn.Size = UDim2.new(0, 30, 0, 30)
    closeBtn.Position = UDim2.new(1, -30, 0, 0)
    closeBtn.BackgroundTransparency = 1
    closeBtn.Text = ""
    closeBtn.ZIndex = 11
    closeBtn.Parent = mainPanel
    closeBtn.Activated:Connect(function()
        closeMenu()
    end)
    
    logoImage = Instance.new("ImageButton")
    logoImage.Size = UDim2.new(0, 40, 0, 40)
    logoImage.Position = UDim2.new(0, 10, 0.5, -20)
    logoImage.Image = "rbxassetid://2942400020"
    logoImage.BackgroundTransparency = 1
    logoImage.AutoButtonColor = false
    logoImage.Parent = mainPanel
    
    local logoHoldTimer = nil
    local wasLongPress = false

    logoImage.MouseButton1Down:Connect(function()
        wasLongPress = false
        logoHoldTimer = task.delay(1, function()
            if menuOpen then
                wasLongPress = true
                -- Prioridad: Si hay algo agarrado, soltarlo. Si no, cerrar menú.
                if SuperStrength.Active and SuperStrength.GrabbedData then
                    releaseGrabbedObject(false) -- MANTENER 1S: Soltar suavemente
                    toggleAbility("superstrength", 1) -- También desactivar para evitar que vuelva a agarrar solo
                elseif abilities[currentAbilityIndex].key == "supersalto" then
                    -- No cerrar si estamos cargando el salto
                else
                    closeMenu() -- MANTENER 1S: Cerrar menú
                end
            end
        end)
        if menuOpen and abilities[currentAbilityIndex].key == "supersalto" then
            startSuperSaltoCharge()
        end
    end)

    logoImage.MouseButton1Up:Connect(function()
        if logoHoldTimer then
            task.cancel(logoHoldTimer)
            logoHoldTimer = nil
        end
        if abilities[currentAbilityIndex].key == "supersalto" then
            releaseSuperSalto()
        elseif not wasLongPress and menuOpen then
            activateCurrent()
        end
    end)
    
    modeText = Instance.new("TextButton")
    modeText.Size = UDim2.new(1, -60, 0, 20)
    modeText.Position = UDim2.new(0, 55, 0, 5)
    modeText.BackgroundTransparency = 1
    modeText.Font = Enum.Font.GothamBold
    modeText.TextSize = 11
    modeText.TextColor3 = Color3.fromRGB(255, 200, 100)
    modeText.TextXAlignment = Enum.TextXAlignment.Left
    modeText.AutoButtonColor = false

    modeText.MouseButton1Down:Connect(function()
        if UserInputService:GetLastInputType() ~= Enum.UserInputType.Touch then return end
        modeLongPressActive = false
        modeHoldTimer = task.delay(1, function()
            modeLongPressActive = true
            currentMode = MODES.SETTINGS
            updateMainUI()
            if settingsPanel then
                settingsPanel.Visible = true
                TweenService:Create(settingsPanel, TweenInfo.new(0.2), {BackgroundTransparency = 0.15}):Play()
            end
            updateMainUI()
            local originalColor = modeText.TextColor3
            modeText.TextColor3 = Color3.fromRGB(255, 100, 100) -- Feedback visual de pulsación larga
            task.wait(0.2)
            modeText.TextColor3 = originalColor
            modeHoldTimer = nil
        end)
    end)

    modeText.MouseButton1Up:Connect(function()
        if modeHoldTimer then task.cancel(modeHoldTimer) modeHoldTimer = nil end
    end)

    modeText.Activated:Connect(function()
        if modeLongPressActive then modeLongPressActive = false return end
        
        -- Alternar entre Habilidades (1) y Emotes (2)
        if currentMode == MODES.ABILITIES then
            currentMode = MODES.EMOTES
        else
            currentMode = MODES.ABILITIES
        end
        
        -- Si estábamos en ajustes, ocultar el panel
        if settingsPanel and settingsPanel.Visible then
            TweenService:Create(settingsPanel, TweenInfo.new(0.2), {BackgroundTransparency = 1}):Play()
            task.delay(0.2, function() settingsPanel.Visible = false end)
        end
        
        updateMainUI()
        local originalColor = modeText.TextColor3
        modeText.TextColor3 = Color3.fromRGB(255, 255, 100)
        task.wait(0.08)
        modeText.TextColor3 = originalColor
    end)
    modeText.Parent = mainPanel
    
    abilityText = Instance.new("TextButton")
    abilityText.Size = UDim2.new(1, -60, 0, 25)
    abilityText.Position = UDim2.new(0, 55, 0, 25)
    abilityText.BackgroundTransparency = 1
    abilityText.Font = Enum.Font.GothamBold
    abilityText.TextSize = 18
    abilityText.TextColor3 = Color3.fromRGB(255, 255, 255)
    abilityText.TextXAlignment = Enum.TextXAlignment.Left
    abilityText.AutoButtonColor = false

    local abilityLongPressActive = false
    local abilityHoldTimer = nil
    
    abilityText.MouseButton1Down:Connect(function()
        if currentMode ~= MODES.ABILITIES or abilities[currentAbilityIndex].key ~= "walkspeed" then return end
        abilityLongPressActive = false
        abilityHoldTimer = task.delay(1, function()
            abilityLongPressActive = true
            if not isWalkSpeedActive then
                toggleAbility("walkspeed", 1)
            end
            walkSpeed = 21
            updateMainUI()
            if speedIndicatorLabel then
                speedIndicatorLabel.Text = "Super Velocidad:(21)"
                speedIndicatorLabel.TextTransparency = 0
                task.delay(1.5, function()
                    TweenService:Create(speedIndicatorLabel, TweenInfo.new(0.5), {TextTransparency = 1}):Play()
                end)
            end
            abilityHoldTimer = nil
        end)
    end)

    abilityText.MouseButton1Up:Connect(function()
        if abilityHoldTimer then task.cancel(abilityHoldTimer) abilityHoldTimer = nil end
    end)

    abilityText.Activated:Connect(function()
        if abilityLongPressActive then abilityLongPressActive = false return end
        local now = tick()
        if now - lastAbilityTap < 0.25 then
            if abilityTapThread then task.cancel(abilityTapThread) abilityTapThread = nil end
            previousSelection()
        else
            abilityTapThread = task.delay(0.26, function()
                nextSelection()
                abilityTapThread = nil
            end)
        end
        lastAbilityTap = now
    end)
    abilityText.Parent = mainPanel

    minusBtn = Instance.new("TextButton")
    minusBtn.Size = UDim2.new(0, 25, 0, 25)
    minusBtn.Position = UDim2.new(0, 55, 0, 53)
    minusBtn.BackgroundTransparency = 1
    minusBtn.Text = "-"
    minusBtn.TextColor3 = Color3.new(1,1,1)
    minusBtn.Font = Enum.Font.GothamBold
    minusBtn.Parent = mainPanel
    Instance.new("UICorner", minusBtn).CornerRadius = UDim.new(0, 6)
    minusBtn.Activated:Connect(function() adjustSpeed(false) end)

    plusBtn = Instance.new("TextButton")
    plusBtn.Size = UDim2.new(0, 25, 0, 25)
    plusBtn.Position = UDim2.new(0, 85, 0, 53)
    plusBtn.BackgroundTransparency = 1
    plusBtn.Text = "+"
    plusBtn.TextColor3 = Color3.new(1,1,1)
    plusBtn.Font = Enum.Font.GothamBold
    plusBtn.Parent = mainPanel
    Instance.new("UICorner", plusBtn).CornerRadius = UDim.new(0, 6)
    plusBtn.Activated:Connect(function() adjustSpeed(true) end)
    
    settingsPanel = Instance.new("Frame")
    settingsPanel.Name = "SettingsIndicator"
    settingsPanel.Size = UDim2.new(0, 200, 0, 50)
    settingsPanel.Position = UDim2.new(0.5, -100, 0.7, 0)
    settingsPanel.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    settingsPanel.BackgroundTransparency = 1
    settingsPanel.BorderSizePixel = 0
    settingsPanel.Visible = false
    settingsPanel.Parent = screenGui
    
    local settingsCorner = Instance.new("UICorner")
    settingsCorner.CornerRadius = UDim.new(0, 12)
    settingsCorner.Parent = settingsPanel
    
    local settingsStroke = Instance.new("UIStroke")
    settingsStroke.Thickness = 2
    settingsStroke.Color = Color3.fromRGB(255, 215, 0)
    settingsStroke.Parent = settingsPanel
    
    local settingsHint = Instance.new("TextLabel")
    settingsHint.Size = UDim2.new(1, 0, 1, 0)
    settingsHint.BackgroundTransparency = 1
    settingsHint.Font = Enum.Font.GothamBold
    settingsHint.TextSize = 14
    settingsHint.TextColor3 = Color3.fromRGB(255, 255, 255)
    settingsHint.Text = "←  →  cambiar   |   tocar = ejecutar"
    settingsHint.Parent = settingsPanel
    
    speedIndicatorLabel = Instance.new("TextLabel")
    speedIndicatorLabel.Size = UDim2.new(0, 150, 0, 30)
    speedIndicatorLabel.Position = UDim2.new(0.5, -75, 0.85, 0)
    speedIndicatorLabel.BackgroundTransparency = 1
    speedIndicatorLabel.Font = Enum.Font.GothamBold
    speedIndicatorLabel.TextSize = 16
    speedIndicatorLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    speedIndicatorLabel.TextStrokeTransparency = 0.5
    speedIndicatorLabel.TextTransparency = 1
    speedIndicatorLabel.Text = ""
    speedIndicatorLabel.Parent = screenGui
    
    updateMainUI()
end

local function setupGestures()
    local menuSwipeStart = nil
    local longPressTimer = nil
    local swipedElement = nil
    local lastLeftClickTime = 0
    local lastRightClickTime = 0
    local leftClickTask = nil
    local rightClickTask = nil
    local DOUBLE_CLICK_THRESHOLD = 0.25
    
    local m3StartTime = 0
    local function isInside(pos, gui)
        if not gui or not gui.Visible then return false end
        local absPos = gui.AbsolutePosition
        local absSize = gui.AbsoluteSize
        return pos.X >= absPos.X and pos.X <= absPos.X + absSize.X and
               pos.Y >= absPos.Y and pos.Y <= absPos.Y + absSize.Y
    end

    local keyboardMenu = UserInputService.InputBegan:Connect(function(input, gp)
        if gp then return end
        if input.KeyCode == Enum.KeyCode.Comma then
            if menuOpen then closeMenu() else openMenu() end
        elseif input.KeyCode == Enum.KeyCode.Home then
            if menuOpen then nextMode() end
        elseif input.KeyCode == Enum.KeyCode.PageUp then
            if menuOpen then previousSelection() end
        elseif input.KeyCode == Enum.KeyCode.PageDown then
            if menuOpen then nextSelection() end
        elseif input.KeyCode == Enum.KeyCode.Delete then
            if menuOpen then
                if currentMode == MODES.SETTINGS then
                    currentMode = MODES.ABILITIES
                    if settingsPanel then
                        TweenService:Create(settingsPanel, TweenInfo.new(0.2), {BackgroundTransparency = 1}):Play()
                        task.delay(0.2, function() settingsPanel.Visible = false end)
                    end
                else
                    currentMode = MODES.SETTINGS
                    updateMainUI()
                    if settingsPanel then
                        settingsPanel.Visible = true
                        TweenService:Create(settingsPanel, TweenInfo.new(0.2), {BackgroundTransparency = 0.15}):Play()
                    end
                end
            end
        elseif input.KeyCode == Enum.KeyCode.Equals or input.KeyCode == Enum.KeyCode.KeypadPlus then
            adjustSpeed(true)
        elseif input.KeyCode == Enum.KeyCode.Minus or input.KeyCode == Enum.KeyCode.KeypadMinus then
            adjustSpeed(false)
        elseif input.KeyCode == Enum.KeyCode.End then
            if menuOpen and currentMode == MODES.ABILITIES and abilities[currentAbilityIndex].key == "supersalto" then
                startSuperSaltoCharge()
            else
                activateCurrent()
            end
        elseif input.KeyCode == Enum.KeyCode.KeypadMultiply or input.KeyCode == Enum.KeyCode.Asterisk then
            if lockedPlayer then
                lockedPlayer = nil
            else
                lockedPlayer = getTargetFromCamera()
            end
        end
    end)
    table.insert(persistentConnections, keyboardMenu)

    local keyboardEndRelease = UserInputService.InputEnded:Connect(function(input, gp)
        if gp then return end
        if input.KeyCode == Enum.KeyCode.End then
            if currentMode == MODES.ABILITIES and abilities[currentAbilityIndex].key == "supersalto" then
                releaseSuperSalto()
            end
        end
    end)
    table.insert(persistentConnections, keyboardEndRelease)

    local mouseMenuControl = UserInputService.InputBegan:Connect(function(input, gp)
        if gp then return end

        -- 1. Clic botón central: Abrir/Cerrar
        if input.UserInputType == Enum.UserInputType.MouseButton3 then
            m3StartTime = tick()
            return
        end

        if menuOpen then
            -- 2 y 3. Control por clics Izquierdo y Derecho
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                local now = tick()
                if now - lastLeftClickTime < DOUBLE_CLICK_THRESHOLD then
                    if leftClickTask then task.cancel(leftClickTask); leftClickTask = nil end
                    activateCurrent() -- Doble clic izquierdo: Activar
                    lastLeftClickTime = 0
                else
                    lastLeftClickTime = now
                    leftClickTask = task.delay(DOUBLE_CLICK_THRESHOLD, function()
                        previousSelection() -- Clic izquierdo simple: Selección hacia la izquierda
                        leftClickTask = nil
                    end)
                end
            elseif input.UserInputType == Enum.UserInputType.MouseButton2 then
                local now = tick()
                if now - lastRightClickTime < DOUBLE_CLICK_THRESHOLD then
                    if rightClickTask then task.cancel(rightClickTask); rightClickTask = nil end
                    
                    -- Doble clic derecho: Alternar entre Habilidades y Emotes
                    if currentMode == MODES.ABILITIES then
                        currentMode = MODES.EMOTES
                    else
                        currentMode = MODES.ABILITIES
                    end
                    
                    if settingsPanel then settingsPanel.Visible = false; settingsPanel.BackgroundTransparency = 1 end
                    updateMainUI()
                    
                    local originalColor = modeText.TextColor3
                    modeText.TextColor3 = Color3.fromRGB(255, 255, 100)
                    task.delay(0.08, function() modeText.TextColor3 = originalColor end)
                    lastRightClickTime = 0
                else
                    lastRightClickTime = now
                    rightClickTask = task.delay(DOUBLE_CLICK_THRESHOLD, function()
                        nextSelection() -- Clic derecho simple: Selección hacia la derecha
                        rightClickTask = nil
                    end)
                end
            end
        end
    end)
    table.insert(persistentConnections, mouseMenuControl)

    local mouseMenuControlEnded = UserInputService.InputEnded:Connect(function(input, gp)
        if input.UserInputType == Enum.UserInputType.MouseButton3 then
            if m3StartTime > 0 then
                local duration = tick() - m3StartTime
                m3StartTime = 0
                if duration >= 1 then
                    -- Si ya está activa, la desactivamos (independientemente de la velocidad)
                    if isWalkSpeedActive then
                        toggleAbility("walkspeed", 1)
                        if speedIndicatorLabel then
                            speedIndicatorLabel.Text = "Super Velocidad: OFF"
                            speedIndicatorLabel.TextTransparency = 0
                            task.delay(1.5, function()
                                TweenService:Create(speedIndicatorLabel, TweenInfo.new(0.5), {TextTransparency = 1}):Play()
                            end)
                        end
                    else
                        -- Si estaba apagada o en otra velocidad, activamos Fase 2 (22)
                        if not isWalkSpeedActive then
                            toggleAbility("walkspeed", 1)
                        end
                        walkSpeed = 21
                        updateMainUI()
                        if speedIndicatorLabel then
                            speedIndicatorLabel.Text = "Super Velocidad: (21)"
                            speedIndicatorLabel.TextTransparency = 0
                            task.delay(1.5, function()
                                TweenService:Create(speedIndicatorLabel, TweenInfo.new(0.5), {TextTransparency = 1}):Play()
                            end)
                        end
                    end
                else
                    if menuOpen then closeMenu() else openMenu() end
                end
            end
        end
    end)
    table.insert(persistentConnections, mouseMenuControlEnded)

    local strengthInput = UserInputService.InputBegan:Connect(function(input, gp)
        if gp or not SuperStrength.Active or menuOpen then return end

        -- Ignorar Touch aquí para que no interfiera con el giro de cámara en Android
        if input.UserInputType == Enum.UserInputType.Touch then return end
        
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            if tick() - SuperStrength.LastActionTime < 0.3 then return end
            if SuperStrength.GrabbedData then
                releaseGrabbedObject(true) -- Lanzar con clic simple
            end
        end
    end)
    table.insert(persistentConnections, strengthInput)

    -- Soporte específico para Android (TouchTap permite girar la cámara sin activar la fuerza)
    local touchStrengthInput = UserInputService.TouchTap:Connect(function(touchPos, gp)
        if not SuperStrength.Active or menuOpen then return end -- Se eliminó 'gp' para permitir la activación incluso si el juego procesa el toque
        
        if tick() - SuperStrength.LastActionTime < 0.3 then return end
        if SuperStrength.GrabbedData then
            releaseGrabbedObject(true) -- Lanzar con toque simple en Android
        end
    end)
    table.insert(persistentConnections, touchStrengthInput)

    local touchBegan = UserInputService.TouchStarted:Connect(function(input)
        local pos = input.Position
        if menuOpen then
            -- Detectar específicamente sobre qué texto se inicia el gesto
            if isInside(pos, modeText) then
                menuSwipeStart = pos
                swipedElement = "mode"
            elseif isInside(pos, abilityText) then
                menuSwipeStart = pos
                swipedElement = "ability"
            elseif isInside(pos, mainPanel) then
                menuSwipeStart = pos
                swipedElement = "panel"
            end
        else
            -- Área global para abrir el menú (franja central de la pantalla)
            local viewportSize = camera.ViewportSize
            local touchX = pos.X / viewportSize.X
            local touchY = pos.Y / viewportSize.Y
            if touchX > 0.35 and touchX < 0.65 and touchY > 0.3 and touchY < 0.7 then
                menuSwipeStart = pos
            end
        end
    end)
    table.insert(persistentConnections, touchBegan)
    
    local touchMoved = UserInputService.TouchMoved:Connect(function(input)
        if not menuSwipeStart then return end
        
        local delta = input.Position - menuSwipeStart
        if menuOpen then
            -- Los cambios ahora se manejan por Tap/Double Tap en los textos
        elseif delta.Y > 100 and math.abs(delta.Y) > math.abs(delta.X) * 2 then
            openMenu()
            menuSwipeStart = nil
        end
    end)
    table.insert(persistentConnections, touchMoved)
    
    local touchEnded = UserInputService.TouchEnded:Connect(function(input)
        menuSwipeStart = nil
        swipedElement = nil
    end)
    table.insert(persistentConnections, touchEnded)
    
    local doubleSpaceTimer = nil
    local spacePressCount = 0
    
    local doubleSpaceConn = UserInputService.InputBegan:Connect(function(input, gp)
        if gp then return end
        if input.KeyCode == Enum.KeyCode.Space then
            spacePressCount = spacePressCount + 1
            if doubleSpaceTimer then task.cancel(doubleSpaceTimer) end
            doubleSpaceTimer = task.delay(0.3, function()
                spacePressCount = 0
                doubleSpaceTimer = nil
            end)
            if spacePressCount == 2 then
                spacePressCount = 0
                if doubleSpaceTimer then task.cancel(doubleSpaceTimer) end
                doubleSpaceTimer = nil
                toggleFlight()
            end
        end
    end)
    table.insert(persistentConnections, doubleSpaceConn)
end

-- ============ LOAD ANIMATIONS ============

local function waitForCharacter()
    local character = player.Character or player.CharacterAdded:Wait()
    local humanoid = character:WaitForChild("Humanoid", 10)
    return character, humanoid
end

local function waitForAnimate(character, timeout)
    timeout = timeout or 2
    local startTime = tick()
    while tick() - startTime < timeout do
        local animate = character:FindFirstChild("Animate")
        if animate and animate:FindFirstChild("run") and animate:FindFirstChild("idle") and animate:FindFirstChild("jump") and animate:FindFirstChild("fall") then
            return animate
        end
        task.wait()
    end
    return nil
end

local function setAnimation(animationType, animationId, animate)
    if not animate or not animationId then return false end
    pcall(function()
        local idStr = "rbxassetid://" .. tostring(animationId)
        local function findChild(parent, name)
            return parent:FindFirstChild(name) or parent:FindFirstChild(name:lower()) or parent:FindFirstChild(name:upper())
        end

        if animationType == "Idle" then
            local idle = findChild(animate, "idle")
            if idle then
                local anim1 = idle:FindFirstChild("Animation1")
                local anim2 = idle:FindFirstChild("Animation2")
                if anim1 then anim1.AnimationId = idStr end
                if anim2 then anim2.AnimationId = idStr end
            end
        elseif animationType == "Walk" then
            local walk = findChild(animate, "walk")
            local walkAnim = walk and (walk:FindFirstChild("WalkAnim") or walk:FindFirstChild("Walk"))
            if walkAnim then walkAnim.AnimationId = idStr end
        elseif animationType == "Run" then
            local run = findChild(animate, "run")
            local runAnim = run and (run:FindFirstChild("RunAnim") or run:FindFirstChild("Run"))
            if runAnim then runAnim.AnimationId = idStr end
        elseif animationType == "Jump" then
            local jump = findChild(animate, "jump")
            local jumpAnim = jump and (jump:FindFirstChild("JumpAnim") or jump:FindFirstChild("Jump"))
            if jumpAnim then jumpAnim.AnimationId = idStr end
            if jump then
                for _, child in ipairs(jump:GetChildren()) do
                    if child:IsA("Animation") then
                        child.AnimationId = idStr
                    end
                end
            end
        elseif animationType == "Fall" then
            local fall = findChild(animate, "fall")
            local fallAnim = fall and (fall:FindFirstChild("FallAnim") or fall:FindFirstChild("Fall"))
            if fallAnim then fallAnim.AnimationId = idStr end
            if fall then
                for _, child in ipairs(fall:GetChildren()) do
                    if child:IsA("Animation") then
                        child.AnimationId = idStr
                    end
                end
            end
        elseif animationType == "Climb" then
            local climb = findChild(animate, "climb")
            local climbAnim = climb and (climb:FindFirstChild("ClimbAnim") or climb:FindFirstChild("Climb"))
            if climbAnim then climbAnim.AnimationId = idStr end
        elseif animationType == "Swim" then
            local swim = findChild(animate, "swim")
            local swimAnim = swim and (swim:FindFirstChild("Swim") or swim:FindFirstChild("SwimAnim"))
            if swimAnim then swimAnim.AnimationId = idStr end
        elseif animationType == "SwimIdle" then
            local swimIdle = findChild(animate, "swimidle")
            local swimIdleAnim = swimIdle and (swimIdle:FindFirstChild("SwimIdle") or swimIdle:FindFirstChild("SwimIdleAnim"))
            if swimIdleAnim then swimIdleAnim.AnimationId = idStr end
        end
    end)
    return true
end

local function saveOriginalAnimations(animate)
    if not animate then return end
    originalAnimations = {
        Idle1 = nil, Idle2 = nil, Walk = nil, Run = nil,
        Jump = nil, Fall = nil, Climb = nil, Swim = nil, SwimIdle = nil
    }
    if animate.idle then
        local anim1 = animate.idle:FindFirstChild("Animation1")
        local anim2 = animate.idle:FindFirstChild("Animation2")
        if anim1 then originalAnimations.Idle1 = anim1.AnimationId end
        if anim2 then originalAnimations.Idle2 = anim2.AnimationId end
    end
    if animate.walk then
        local walkAnim = animate.walk:FindFirstChild("WalkAnim")
        if walkAnim then originalAnimations.Walk = walkAnim.AnimationId end
    end
    if animate.run then
        local runAnim = animate.run:FindFirstChild("RunAnim")
        if runAnim then originalAnimations.Run = runAnim.AnimationId end
    end
    if animate.jump then
        local jumpAnim = animate.jump:FindFirstChild("JumpAnim")
        if jumpAnim then originalAnimations.Jump = jumpAnim.AnimationId end
    end
    if animate.fall then
        local fallAnim = animate.fall:FindFirstChild("FallAnim")
        if fallAnim then originalAnimations.Fall = fallAnim.AnimationId end
    end
    if animate.climb then
        local climbAnim = animate.climb:FindFirstChild("ClimbAnim")
        if climbAnim then originalAnimations.Climb = climbAnim.AnimationId end
    end
    if animate:FindFirstChild("swim") then
        local swimAnim = animate.swim:FindFirstChild("Swim")
        if swimAnim then originalAnimations.Swim = swimAnim.AnimationId end
    end
    if animate:FindFirstChild("swimidle") then
        local swimIdleAnim = animate.swimidle:FindFirstChild("SwimIdle")
        if swimIdleAnim then originalAnimations.SwimIdle = swimIdleAnim.AnimationId end
    end
end

local function restoreOriginalAnimations()
    local character = player.Character
    if not character then return end
    local humanoid = character:FindFirstChild("Humanoid")
    local animate = character:FindFirstChild("Animate")
    
    if humanoid then
        local animator = humanoid:FindFirstChildOfClass("Animator")
        local target = animator or humanoid
        for _, track in ipairs(target:GetPlayingAnimationTracks()) do
            track:Stop(0)
        end
    end

    if currentAnimTrack then currentAnimTrack:Stop(0) currentAnimTrack = nil end
    if walkAnimTrack then walkAnimTrack:Stop(0) walkAnimTrack = nil end
    
    -- Limpiar cache global
    for _, track in pairs(animationTracks) do
        if track then track:Stop(0) end
    end
    animationTracks = {}

    if not animate then 
        customAnimationsLoaded = false
        return 
    end

    if originalAnimations.Idle1 then
        local anim1 = animate.idle:FindFirstChild("Animation1")
        if anim1 then anim1.AnimationId = originalAnimations.Idle1 end
    end
    if originalAnimations.Idle2 then
        local anim2 = animate.idle:FindFirstChild("Animation2")
        if anim2 then anim2.AnimationId = originalAnimations.Idle2 end
    end
    if originalAnimations.Walk then
        local walkAnim = animate.walk:FindFirstChild("WalkAnim")
        if walkAnim then walkAnim.AnimationId = originalAnimations.Walk end
    end
    if originalAnimations.Run then
        local runAnim = animate.run:FindFirstChild("RunAnim")
        if runAnim then runAnim.AnimationId = originalAnimations.Run end
    end
    if originalAnimations.Jump then
        local jumpAnim = animate.jump:FindFirstChild("JumpAnim")
        if jumpAnim then jumpAnim.AnimationId = originalAnimations.Jump end
    end
    if originalAnimations.Fall then
        local fallAnim = animate.fall:FindFirstChild("FallAnim")
        if fallAnim then fallAnim.AnimationId = originalAnimations.Fall end
    end
    if originalAnimations.Climb then
        local climbAnim = animate.climb:FindFirstChild("ClimbAnim")
        if climbAnim then climbAnim.AnimationId = originalAnimations.Climb end
    end
    if originalAnimations.Swim then
        local swimAnim = animate.swim:FindFirstChild("Swim")
        if swimAnim then swimAnim.AnimationId = originalAnimations.Swim end
    end
    if originalAnimations.SwimIdle then
        local swimIdleAnim = animate.swimidle:FindFirstChild("SwimIdle")
        if swimIdleAnim then swimIdleAnim.AnimationId = originalAnimations.SwimIdle end
    end
    
    -- Forzar reinicio del script de Roblox
    animate.Disabled = true
    task.wait(0.1)
    animate.Disabled = false
    customAnimationsLoaded = false
end

loadAnimations = function()
    local character, humanoid = waitForCharacter()
    if not character or not humanoid then return false end
    if isR6 then return true end -- No cargar animaciones R15 si el rig es R6
    local animate = waitForAnimate(character)
    if not animate then return false end
    animationTracks = {}
    walkAnimTrack = nil
    if not customAnimationsLoaded then saveOriginalAnimations(animate) end
    local animator = humanoid:FindFirstChildOfClass("Animator") or humanoid
    local animTypes = {"Idle", "Walk", "Run", "Jump", "Fall", "Climb", "Swim", "SwimIdle"}
    for _, animType in ipairs(animTypes) do
        if ANIMATIONS[animType] then
            setAnimation(animType, ANIMATIONS[animType], animate)
        end
    end

    -- Reiniciar el script de animaciones nativo para aplicar los nuevos IDs inyectados
    animate.Disabled = true
    task.wait(0.1)
    animate.Disabled = false

    -- Pre-cargar los tracks en el cache para que el loop manual los use al instante
    for _, id in pairs(ANIMATIONS) do
        local anim = Instance.new("Animation")
        anim.AnimationId = "rbxassetid://" .. id
        pcall(function()
            local track = animator:LoadAnimation(anim)
            track.Priority = Enum.AnimationPriority.Movement
            animationTracks[id] = track
        end)
    end

    customAnimationsLoaded = true
    return true
end

local function forceInitialLoad()
    task.spawn(function()
        for i = 1, 5 do
            if loadAnimations() then 
                -- Forzar actualización de estado físico para "despertar" el motor de animaciones
                if humanoid then
                    humanoid:ChangeState(Enum.HumanoidStateType.Landed)
                    task.wait(0.1)
                    humanoid:ChangeState(Enum.HumanoidStateType.Running)
                end
                break 
            end
            task.wait(0.5)
        end
    end)
end

-- ============ CLEANUP ============

cleanupEverything = function(isFullDestruction)
    -- Limpieza de instancias físicas sin apagar los estados lógicos si es solo un respawn
    if flyGyro then flyGyro:Destroy() flyGyro = nil end
    if flyVelocity then flyVelocity:Destroy() flyVelocity = nil end
    if flyPos then flyPos:Destroy() flyPos = nil end
    if invisibilitySeat then invisibilitySeat:Destroy() invisibilitySeat = nil end
    if walkPos then walkPos:Destroy() walkPos = nil end
    
    stopFlightAudio()
    stopFlightAnimation()
    stopAntiTP()
    stopAntiFall()
    stopNoclip()
    stopAntiFling()
    if SuperHearing.Heartbeat then pcall(function() SuperHearing.Heartbeat:Disconnect() end) end
    if SuperHearing.Sound then pcall(function() SuperHearing.Sound:Stop() SuperHearing.Sound:Destroy() end) end
    SuperHearing.PlayerHighlights = {}
    SuperHearing.NPCHighlights = {}
    SuperHearing.ActiveNPCs = {}

    -- IMPORTANTE: el lock-on era una conexión suelta y se acumulaba. Ahora la manejamos.
    if lockOnConn and lockOnConn.Connected then pcall(function() lockOnConn:Disconnect() end) end
    lockOnConn = nil
    lockedPlayer = nil

    if SuperStrength.Connection then SuperStrength.Connection:Disconnect() SuperStrength.Connection = nil end
    if SuperStrength.GrabbedData then
        local d = SuperStrength.GrabbedData
        if d.alignPos then d.alignPos:Destroy() end
        if d.att0 then d.att0:Destroy() end
        SuperStrength.GrabbedData = nil
        stopGrabAnim()
    end
    if isFullDestruction then
        SuperStrength.Active = false
    end

    if isFullDestruction then
        isFlying = false
        isWalkSpeedActive = false
        isMirageSpeedActive = false
        isSuperJumpActive = false
        SuperHearing.Active = false
        isInvisibilityActive = false
        isAntiTPActive = false
        isAntiFallActive = false
        isSprintActive = false
        if mirageSpeedEmoteTrack then mirageSpeedEmoteTrack:Stop() mirageSpeedEmoteTrack = nil end
        lockedPlayer = nil

        -- Detener y destruir el sonido de emote actual si existe
        if currentEmoteSound then
            currentEmoteSound:Stop()
            currentEmoteSound:Destroy()
            currentEmoteSound = nil
        end

        -- Limpieza profunda de todos los sonidos de emotes en el personaje
        if character then
            for _, v in ipairs(character:GetDescendants()) do
                if v:IsA("Sound") and v.Name == "SpecialEmoteSound" then pcall(function() v:Stop() v:Destroy() end) end
            end
        end

        -- Limpiar conexiones del sistema y UI solo en DESTRUCCIÓN TOTAL
        for _, conn in ipairs(persistentConnections) do 
            if conn and conn.Connected then pcall(function() conn:Disconnect() end) end 
        end
        persistentConnections = {}

        if screenGui then
            screenGui:Destroy()
            screenGui = nil
        end

        for _, p in ipairs(Players:GetPlayers()) do
            if p.Character then
                local shHighlight = p.Character:FindFirstChild("SuperHearingPlayerHighlight")
                if shHighlight then shHighlight:Destroy() end
                local thermalHighlight = p.Character:FindFirstChild("ThermalVisionHighlight")
                if thermalHighlight then thermalHighlight:Destroy() end
            end
        end
        for sound, properties in pairs(originalSoundProperties) do
            if sound and sound.Parent then
                sound.Volume = properties.Volume
                if properties.Playing and not sound.Playing then sound:Play() end
                sound.Looped = properties.Looped
            end
        end
    end
    originalSoundProperties = {}
    if humanoid then
        humanoid.Jump = true
        humanoid.WalkSpeed = originalWalkSpeed
        humanoid.JumpPower = originalJumpPower
        humanoid.UseJumpPower = true
        humanoid.PlatformStand = false
        pcall(function() humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, true) end)
        local animate = character:FindFirstChild("Animate")
        if animate then animate.Disabled = false end
        if customAnimationsLoaded then restoreOriginalAnimations() end
    end

    -- Limpiar solo conexiones temporales del personaje (necesario al respawnear)
    for _, conn in ipairs(allConnections) do 
        if conn and conn.Connected then pcall(function() conn:Disconnect() end) end 
    end
    allConnections = {}

    SuperHearing.PlayerCache = {}
    SuperHearing.NPCCache = {}
    stopNoclip()
    for _, track in pairs(animationTracks) do if track then pcall(function() track:Stop() end) end end
    if jumpAnimTrack then jumpAnimTrack:Stop() jumpAnimTrack = nil end
    if fallAnimTrack then fallAnimTrack:Stop() fallAnimTrack = nil end
    if sitAnimTrack then sitAnimTrack:Stop() sitAnimTrack = nil end
    if swimAnimTrack then swimAnimTrack:Stop() swimAnimTrack = nil end
    if swimIdleAnimTrack then swimIdleAnimTrack:Stop() swimIdleAnimTrack = nil end
    if climbAnimTrack then climbAnimTrack:Stop() climbAnimTrack = nil end
    animationTracks = {}
    if walkAnimTrack then walkAnimTrack:Stop() walkAnimTrack = nil end
    if hurtOverlayTrack then hurtOverlayTrack:Stop() hurtOverlayTrack = nil end
end


-- ============ TOGGLE ABILITY ============

local function toggleSuperStrength()
    if not scriptActive then return end
    SuperStrength.Active = not SuperStrength.Active
    SuperStrength.ActivationTime = tick()

    -- ID de Animación R6 para Super Agarre: 56146409 (Brazos arriba)
    local grabAnimId = isR6 and 56146409 or 87088218490918

    if not SuperStrength.Active then
        releaseGrabbedObject(false) -- Asegurar limpieza total de físicas al apagar
        if humanoid then pcall(function() humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, not noSitActive) end) end
        if SuperStrength.Connection then SuperStrength.Connection:Disconnect() SuperStrength.Connection = nil end
    else
        SuperStrength.Connection = RunService.Heartbeat:Connect(function()
            if not SuperStrength.Active or not hrp then return end
            if SuperStrength.GrabbedData then
                -- NoSit constante mientras se carga
                if humanoid then humanoid.Sit = false end

                local d = SuperStrength.GrabbedData
                -- Comprobación más segura: que el objeto siga existiendo en el juego
                if not d.root or not d.root:IsDescendantOf(game) then
                    releaseGrabbedObject(false)
                    return
                end
                
                -- Bloquear estado de sentado solo mientras cargamos algo
                if humanoid then pcall(function() humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, false) end) end

                -- Forzar desanclado en cada frame para que no se suelte solo por físicas del juego
                if d.root.Anchored then pcall(function() d.root.Anchored = false end) end

                -- Lógica: Altura mayor para vehículos (12) y normal para objetos (6)
                local currentOffset = d.isVehicle and 12 or 6
                local headTop = hrp.Position + Vector3.new(0, hrp.Size.Y / 2 + currentOffset, 0)
                
                d.att1.WorldCFrame = CFrame.new(headTop) * (camera.CFrame - camera.CFrame.Position)
                
                if not SuperStrength.AnimTrack then playGrabAnim() end
            else
                -- Si no cargamos nada, permitimos sentarse normalmente
                if humanoid then pcall(function() humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, not noSitActive) end) end

                -- AGARRE AUTOMÁTICO POR PROXIMIDAD
                -- Seguridad: No agarrar nada justo al activar o justo después de soltar
                if tick() - SuperStrength.ActivationTime < 0.8 then return end
                if tick() - SuperStrength.LastActionTime < 1.5 then return end
                
                local params = OverlapParams.new()
                params.FilterDescendantsInstances = {character}
                params.FilterType = Enum.RaycastFilterType.Exclude
                
                -- Elevamos el centro de detección para no tocar el suelo
                local detectionCenter = hrp.Position + Vector3.new(0, 2, 0)
                local nearbyParts = Workspace:GetPartBoundsInRadius(detectionCenter, 10, params)
                
                for _, part in ipairs(nearbyParts) do
                    -- Solo agarrar si el objeto está FRENTE al personaje (Dot product)
                    local toPart = (part.Position - hrp.Position).Unit
                    if hrp.CFrame.LookVector:Dot(toPart) < 0.4 then continue end

                    -- No agarrar nada que esté por debajo de nuestros pies (evita el bug del suelo)
                    if part.Position.Y < hrp.Position.Y - 1.5 and not part.Name:lower():find("car") then continue end

                    local can, mainPart, model, isV = canGrabObject(part)
                    if can then
                        local saved = isV and disconnectVehicleOccupants(model) or {}
                        local att0 = Instance.new("Attachment", mainPart)
                        local att1 = Instance.new("Attachment", hrp)
                        local ap = Instance.new("AlignPosition", mainPart)
                        ap.Attachment0 = att0
                        ap.Attachment1 = att1
                        ap.Responsiveness = 200
                        ap.MaxForce = math.huge
                        local ao = Instance.new("AlignOrientation", mainPart)
                        ao.Attachment0 = att0
                        ao.Attachment1 = att1
                        ao.Responsiveness = 50
                        ao.MaxTorque = math.huge
                        
                        local savedPhysics = {}
                        local targetModel = model or mainPart
                        for _, v in ipairs(targetModel:GetDescendants()) do
                            if v:IsA("BasePart") then
                                savedPhysics[v] = {Massless = v.Massless, CanCollide = v.CanCollide}
                                v.Massless = true -- Quita el peso para que no te aplaste
                                v.CanCollide = false -- IMPORTANTE: Evita que el objeto choque con el suelo y te empuje/hunda
                                
                                local nc = Instance.new("NoCollisionConstraint", v)
                                nc.Name = "SuperGrabNC"
                                nc.Part0 = v; nc.Part1 = hrp
                            end
                        end
                        
                        pcall(function() mainPart.Anchored = false end)
                        pcall(function() mainPart.AssemblyLinearVelocity = Vector3.zero end) -- Detiene tirones bruscos
                        pcall(function() mainPart.AssemblyAngularVelocity = Vector3.zero end)
                        pcall(function() mainPart:SetNetworkOwner(player) end) -- Toma control de las físicas

                        SuperStrength.GrabbedData = {
                            root = mainPart, model = model, isVehicle = isV, savedStates = saved,
                            alignPos = ap, alignOrient = ao, att0 = att0, att1 = att1,
                            savedPhysics = savedPhysics
                        }
                        SuperStrength.LastActionTime = tick()
                        break
                    end
                end
            end
        end)
        table.insert(allConnections, SuperStrength.Connection)
    end
end

toggleAbility = function(key, mode)
    if not scriptActive then return end
    if key == "walkspeed" and humanoid then
        isWalkSpeedActive = not isWalkSpeedActive
        -- Sincronizamos tanto WalkSpeed como empuje físico para mayor fluidez
        humanoid.WalkSpeed = isWalkSpeedActive and walkSpeed or originalWalkSpeed
        
        print("🏃 Velocidad: " .. (isWalkSpeedActive and "ACTIVADA" or "DESACTIVADA"))
        
        if isWalkSpeedActive then
            if not walkPos then
                walkPos = Instance.new("BodyPosition")
                walkPos.Name = "WalkAnchor"
                walkPos.MaxForce = Vector3.new(0, 0, 0) -- Se inicializa vacío
                walkPos.D = 500
                walkPos.P = 10000
                walkPos.Position = hrp.Position
                walkPos.Parent = hrp
            end
            -- Inicializar posición de anclaje inmediatamente para evitar el "pull back"
            lastWalkStationaryPos = hrp.Position
            hrp.AssemblyLinearVelocity = Vector3.new(0, hrp.AssemblyLinearVelocity.Y, 0)
        else
            if walkPos then pcall(function() walkPos:Destroy() end) walkPos = nil end
            lastWalkStationaryPos = nil
            if hrp then
                hrp.AssemblyLinearVelocity = Vector3.new(0, hrp.AssemblyLinearVelocity.Y, 0)
            end
        end
        
    elseif key == "miragespeed" then
        toggleMirageSpeed()
    elseif key == "gravjump" then
        isGravJumpActive = not isGravJumpActive
        if isGravJumpActive then
            humanoid.JumpPower = 70
            humanoid.UseJumpPower = true
        else
            humanoid.JumpPower = originalJumpPower
            Workspace.Gravity = originalGravity
        end
    elseif key == "supersalto" then
        -- Se maneja por eventos de MouseButton1Down/Up en el logo
    elseif key == "superhearing" then
        if SuperHearing.Active then stopSuperHearing() else startSuperHearing() end
    elseif key == "thermalvision" then
        toggleThermalVision()
    elseif key == "molecularvibration" then
        toggleInvisibility()
    elseif key == "flight" then
        toggleFlight()
    elseif key == "superstrength" then
        toggleSuperStrength()
    elseif key == "instantreaction" then
        if isR6 then
            InstantReaction.Active = not InstantReaction.Active
            if InstantReaction.Active then
                InstantReaction.Connection = RunService.RenderStepped:Connect(updateInstantReaction)
            else
                if InstantReaction.Connection then 
                    InstantReaction.Connection:Disconnect() 
                    InstantReaction.Connection = nil 
                end
            end
        end
    end
    updateMainUI()
end

adjustSpeed = function(increase)
    local msg = ""
    if isWalkSpeedActive then
        if increase then
            if walkSpeed < 19 then -- Ahora sube de 1 en 1 hasta el 19
                walkSpeed = walkSpeed + 1
            else
                walkSpeed = math.min(walkSpeed + 10, maxWalkSpeed)
            end
        else
            if walkSpeed <= 19 then -- Baja de 1 en 1 si es 19 o menor
                walkSpeed = math.max(11, walkSpeed - 1)
            else
                walkSpeed = math.max(19, walkSpeed - 10)
            end
        end
        msg = "Velocidad: " .. walkSpeed
    elseif isGravJumpActive then
        if increase then
            superJumpPower = math.min(superJumpPower + superJumpStep, maxSuperJumpPower)
        else
            superJumpPower = math.max(superJumpPower - superJumpStep, minSuperJumpPower)
        end
        if humanoid then humanoid.JumpPower = superJumpPower end
        msg = "Salto: " .. superJumpPower
    end

    if msg ~= "" and speedIndicatorLabel then
        speedIndicatorLabel.Text = msg
        speedIndicatorLabel.TextTransparency = 0
        task.spawn(function()
            task.wait(1.5)
            if speedIndicatorLabel.Text == msg then
                TweenService:Create(speedIndicatorLabel, TweenInfo.new(0.5), {TextTransparency = 1}):Play()
            end
        end)
    end
end

-- ============ EVENTOS ============

startMainAnimationLoop = function()
    local conn = RunService.Heartbeat:Connect(function(deltaTime)
    if not scriptActive then return end

    -- Bloquear actualizaciones de animación si estamos esquivando para que se vea la animación de dodge
    if InstantReaction.Active and InstantReaction.DodgeActive then return end

    if cancelCustomAnims then
        local animate = character:FindFirstChild("Animate")
        if animate then animate.Disabled = false end
        return
    end

    if isGravJumpActive and hrp then
        local velY = hrp.AssemblyLinearVelocity.Y
        if velY > 5 then
            Workspace.Gravity = 30 -- Sin gravedad al subir (pose de caída)
        elseif velY < -5 then
            Workspace.Gravity = 80 -- Gravedad suave al bajar
        else
            Workspace.Gravity = originalGravity
        end
    end

    local healthPercent = (humanoid and humanoid.MaxHealth > 0) and (humanoid.Health / humanoid.MaxHealth) or 1
    local isHurt = healthPercent <= 0.5

    -- [INTELIGENTE] Detección de movimiento por intención del usuario (Teclado o Joystick)
    local moveMag = humanoid and humanoid.MoveDirection.Magnitude or 0
    local hasKeyInput = UserInputService:IsKeyDown(Enum.KeyCode.W) or UserInputService:IsKeyDown(Enum.KeyCode.A) or UserInputService:IsKeyDown(Enum.KeyCode.S) or UserInputService:IsKeyDown(Enum.KeyCode.D)
    
    -- Sensibilidad mejorada para Android/Joystick: detecta movimiento con un toque leve
    local isMoving = moveMag > 0.1
    
    local isEmotePlaying = false

    if humanoid and hrp then
        isCrouching = character:GetAttribute("Crouching") == true or (humanoid.WalkSpeed < 9 and not isHurt and not isWalkSpeedActive and moveMag > 0.1)
        local state = humanoid:GetState()
        local isSeated = (state == Enum.HumanoidStateType.Seated) or (humanoid.Sit == true) or (humanoid.SeatPart ~= nil)
        local isClimbing = (state == Enum.HumanoidStateType.Climbing)
        local isSwimming = (state == Enum.HumanoidStateType.Swimming)
        local isExternal = isExternalAnimPlaying() or isCrouching or (hrp and hrp.Anchored) or (humanoid.PlatformStand and not isFlying) or (state == Enum.HumanoidStateType.None) or (state == Enum.HumanoidStateType.Physics)
        local animate = character:FindFirstChild("Animate")
        
        -- Bypass de Velocidad para Doors:
        -- Para velocidades extremas (hasta 200), subimos el WalkSpeed físico al límite de detección (21.9).
        -- Esto reduce la brecha entre la posición real y la posición calculada por el servidor.
        if isWalkSpeedActive and isMoving and humanoid and not isExternal then
            local baseSpoof = (walkSpeed >= 21) and 21 or 17.5
            humanoid.WalkSpeed = baseSpoof + (math.random(-5, 5) / 20)
        elseif not isWalkSpeedActive and not isHurt then
            humanoid.WalkSpeed = originalWalkSpeed
        end

        -- PRIORIDAD MÁXIMA: Estados nativos de Roblox o Externos (Sentarse, Escalar, Nadar, Agacharse o Cinemáticas)
        if isSeated or isClimbing or isSwimming or isExternal then
            if isFlying and not isExternal then stopFlight() end
            if isExternal and animate then
                animate.Disabled = false -- Devolver el control a las animaciones del juego
            end

            if isExternal and humanoid then
                humanoid.WalkSpeed = 16 -- Devolver control al juego durante interacción
            end
            
            -- Detener fuerzas de velocidad para no salir disparado de la escalera o silla
            if walkPos then walkPos.MaxForce = Vector3.new(0, 0, 0) end
            lastWalkStationaryPos = nil

            -- Detener tracks del mod inmediatamente para no interferir con el estado nativo
            if currentAnimTrack then currentAnimTrack:Stop(0.2) currentAnimTrack = nil end
            if walkAnimTrack then walkAnimTrack:Stop(0.2) walkAnimTrack = nil end
            stopSprintAnimation()
            if jumpAnimTrack then jumpAnimTrack:Stop(0.2) jumpAnimTrack = nil end
            if fallAnimTrack then fallAnimTrack:Stop(0.2) fallAnimTrack = nil end

            -- Manejo manual de la animación de sentado para evitar que el personaje quede "tieso"
            if isSeated then
                local sitIdStr = "rbxassetid://" .. ANIMATIONS.Sit
                if not sitAnimTrack or sitAnimTrack.Animation.AnimationId ~= sitIdStr then
                    if sitAnimTrack then sitAnimTrack:Stop(0.1) end
                    local anim = Instance.new("Animation")
                    anim.AnimationId = sitIdStr
                    sitAnimTrack = humanoid:LoadAnimation(anim)
                    sitAnimTrack.Priority = Enum.AnimationPriority.Action
                    sitAnimTrack.Looped = true
                    sitAnimTrack:Play(0.2)
                elseif not sitAnimTrack.IsPlaying then
                    sitAnimTrack:Play(0.2)
                end
            else
                if sitAnimTrack then sitAnimTrack:Stop(0.2) sitAnimTrack = nil end
            end

            -- Manejo manual de la animación de nadar
            if isSwimming then
                local isSwimmingMoving = humanoid.MoveDirection.Magnitude > 0.1
                local targetSwimAnimId = isSwimmingMoving and ANIMATIONS.Swim or ANIMATIONS.SwimIdle
                local swimIdStr = "rbxassetid://" .. tostring(targetSwimAnimId)

                if isSwimmingMoving then
                    if swimIdleAnimTrack then swimIdleAnimTrack:Stop(0.2) swimIdleAnimTrack = nil end
                else
                    if swimAnimTrack then swimAnimTrack:Stop(0.2) swimAnimTrack = nil end
                end

                local activeTrack = isSwimmingMoving and swimAnimTrack or swimIdleAnimTrack
                if not activeTrack or activeTrack.Animation.AnimationId ~= swimIdStr then
                    if activeTrack then activeTrack:Stop(0.1) end
                    local anim = Instance.new("Animation")
                    anim.AnimationId = swimIdStr
                    activeTrack = humanoid:LoadAnimation(anim)
                    activeTrack.Priority = Enum.AnimationPriority.Action
                    activeTrack.Looped = true
                    activeTrack:Play(0.2)
                    if isSwimmingMoving then swimAnimTrack = activeTrack else swimIdleAnimTrack = activeTrack end
                elseif not activeTrack.IsPlaying then
                    activeTrack:Play(0.2)
                end

                -- Lógica de Super Swim (Velocidad 35 base, o compatible con Habilidad de Velocidad)
                if isSwimmingMoving then
                    local targetSwimSpeed = isWalkSpeedActive and walkSpeed or 35
                    local moveDir = humanoid.MoveDirection
                    
                    -- Método de Bypass: Usamos el mismo sistema de CFrame boost para evitar lag o detección
                    local serverLimit = (targetSwimSpeed > 50) and 21.5 or 17.8
                    local boost = math.max(0, targetSwimSpeed - serverLimit)
                    
                    humanoid.WalkSpeed = serverLimit + (math.random(-5, 5) / 20)
                    hrp.CFrame = hrp.CFrame + (moveDir * boost * deltaTime)
                    hrp.AssemblyLinearVelocity = Vector3.new(moveDir.X * serverLimit, hrp.AssemblyLinearVelocity.Y, moveDir.Z * serverLimit)
                end
            else
                if swimAnimTrack then swimAnimTrack:Stop(0.2) swimAnimTrack = nil end
                if swimIdleAnimTrack then swimIdleAnimTrack:Stop(0.2) swimIdleAnimTrack = nil end
            end

            -- Manejo manual de la animación de escalar
            if isClimbing then
                local climbIdStr = "rbxassetid://" .. tostring(ANIMATIONS.Climb)
                if not climbAnimTrack or climbAnimTrack.Animation.AnimationId ~= climbIdStr then
                    if climbAnimTrack then climbAnimTrack:Stop(0.1) end
                    local anim = Instance.new("Animation")
                    anim.AnimationId = climbIdStr
                    climbAnimTrack = humanoid:LoadAnimation(anim)
                    climbAnimTrack.Priority = Enum.AnimationPriority.Action
                    climbAnimTrack.Looped = true
                    climbAnimTrack:Play(0.2)
                elseif not climbAnimTrack.IsPlaying then
                    climbAnimTrack:Play(0.2)
                end
                local verticalVel = math.abs(hrp.AssemblyLinearVelocity.Y)
                climbAnimTrack:AdjustSpeed(verticalVel > 0.1 and (verticalVel / 5) or 0)
            else
                if climbAnimTrack then climbAnimTrack:Stop(0.2) climbAnimTrack = nil end
            end

            -- Si es una animación externa o agachado SIN velocidad activa, cedemos el control a Roblox
            if isCrouching then return end

            -- Detener tracks del mod para permitir las animaciones de Roblox (Sit/Climb/Swim)
            for _, track in ipairs(humanoid:GetPlayingAnimationTracks()) do
                local anim = track.Animation
                local animId = anim and anim.AnimationId or ""
                local numId = tostring(animId):match("%d+")
                local name = track.Name:lower()

                -- No detener la animación correspondiente al estado actual (climb, swim o sit)
                local isStateAnim = (isClimbing and (name:find("climb") or numId == tostring(ANIMATIONS.Climb))) or
                                   (isSwimming and (name:find("swim") or numId == tostring(ANIMATIONS.Swim) or numId == tostring(ANIMATIONS.SwimIdle))) or
                                   (isSeated and (track == sitAnimTrack or name:find("sit") or name:find("seat")))

                -- Bloqueamos las IDLE y otras gestionadas que no sean la del estado actual para evitar solapamientos
                if not isStateAnim and ((numId and managedAnimationIds[numId]) or name:find("idle")) then
                    track:Stop(0.1)
                end
            end
            
            return -- Salir del loop para este frame si estamos en un estado especial
        end
        
        -- Si hay una animación externa (ej. cinemática de Seek), no forzamos el movimiento del mod para evitar el efecto "tieso"
        -- Pero permitimos que el sistema de anclaje funcione si no nos estamos moviendo para evitar el deslizamiento.
        if isExternal then
            if currentAnimTrack and managedAnimationIds[currentAnimTrack.Animation.AnimationId:match("%d+") or ""] then 
                currentAnimTrack:Stop(0.3) 
                currentAnimTrack = nil 
            end
        end

        -- Si no estamos sentados, nos aseguramos de limpiar la track de sentado
        if sitAnimTrack then sitAnimTrack:Stop(0.2) sitAnimTrack = nil end
        if swimAnimTrack then swimAnimTrack:Stop(0.2) swimAnimTrack = nil end
        if swimIdleAnimTrack then swimIdleAnimTrack:Stop(0.2) swimIdleAnimTrack = nil end
        if climbAnimTrack then climbAnimTrack:Stop(0.2) climbAnimTrack = nil end
    end

    if isFlying then return end
    
    if humanoid and hrp then
        local animate = character:FindFirstChild("Animate")
        -- Solo desactivar el animador de Roblox si no es R6 para evitar el t-pose
        if not isR6 and animate and not animate.Disabled then animate.Disabled = true end

        -- Bypass de velocidad para R6 (Solución para compatibilidad universal)
        if isR6 and not isFlying then
            local currentState = humanoid:GetState()
            -- Lógica de Caída Personalizada R6: Solo si realmente no hay suelo debajo
            local inAir = (humanoid.FloorMaterial == Enum.Material.Air) and (currentState == Enum.HumanoidStateType.Freefall or currentState == Enum.HumanoidStateType.Jumping)
            local vy = hrp.AssemblyLinearVelocity.Y

            -- Filtro agresivo para escaleras R6: Ignorar caídas leves (rebotes o jitter) incluso quieto
            if inAir and currentState ~= Enum.HumanoidStateType.Jumping and vy > -60 then
                inAir = false
            end

            if inAir and vy < -60 then -- Solo cae si la velocidad hacia abajo es mayor a 60
                local fallId = 97170520
                local animIdStr = "rbxassetid://" .. tostring(fallId)
                if not fallAnimTrack or fallAnimTrack.Animation.AnimationId ~= animIdStr then
                    if fallAnimTrack then fallAnimTrack:Stop() end
                    local anim = Instance.new("Animation")
                    anim.AnimationId = animIdStr
                    fallAnimTrack = humanoid:LoadAnimation(anim)
                    fallAnimTrack.Priority = Enum.AnimationPriority.Action4
                    fallAnimTrack:Play(0.2)
                    fallAnimTrack:AdjustSpeed(1.6) -- Velocidad aumentada a petición
                end
            else
                if fallAnimTrack and fallAnimTrack.Animation.AnimationId == "rbxassetid://97170520" then
                    fallAnimTrack:Stop(0.3)
                    fallAnimTrack = nil
                end
            end

            if isMoving then
                lastWalkStationaryPos = nil
                if walkPos then walkPos.MaxForce = Vector3.new(0, 0, 0) end
                if isWalkSpeedActive then
                    local moveDir = humanoid.MoveDirection
                local serverLimit = (walkSpeed > 50) and 21.5 or 17.8
                    local boost = math.max(0, walkSpeed - serverLimit)
                    
                    -- Método de Bypass Escalable (R6) para cualquier experiencia
                    local jitter = (walkSpeed > 100) and 0.04 or 0.01
                    hrp.CFrame = hrp.CFrame + (moveDir * (boost + (math.random(-1, 1) * jitter)) * deltaTime)
                    
                    humanoid.WalkSpeed = serverLimit + (math.random(-5, 5) / 20)
                    hrp.AssemblyLinearVelocity = Vector3.new(moveDir.X * serverLimit, hrp.AssemblyLinearVelocity.Y, moveDir.Z * serverLimit)
                else
                    humanoid.WalkSpeed = originalWalkSpeed
                end
            else
                -- Sistema de anclaje para evitar drifting en R6 cuando se detiene
                if not lastWalkStationaryPos and hrp then lastWalkStationaryPos = hrp.Position end
                if walkPos and lastWalkStationaryPos and (humanoid.FloorMaterial ~= Enum.Material.Air) then
                    local anchorForce = 1e6
                    walkPos.MaxForce = Vector3.new(anchorForce, 0, anchorForce) -- Bloqueo total X, Z
                    walkPos.Position = Vector3.new(lastWalkStationaryPos.X, hrp.Position.Y, lastWalkStationaryPos.Z)
                    hrp.AssemblyLinearVelocity = Vector3.new(0, hrp.AssemblyLinearVelocity.Y, 0)
                end
            end
            return 
        end

        -- [MEJORA] Si te mueves, silenciamos el sonido inmediatamente antes de detener el emote
        if currentEmoteSound then
            currentEmoteSound.Volume = (isMoving) and 0 or 1
        end

        -- Detección de aire y estado de salto
        local currentState = humanoid:GetState()
        local isJumpingState = (currentState == Enum.HumanoidStateType.Jumping)
        local inAir = (humanoid.FloorMaterial == Enum.Material.Air) and (isJumpingState or currentState == Enum.HumanoidStateType.Freefall)
        local verticalVelocity = hrp.AssemblyLinearVelocity.Y

        -- FILTRO AGRESIVO PARA ESCALERAS R15:
        -- Ignoramos caídas si no estamos saltando y la velocidad vertical no es profunda (evita fall en escalones o quieto)
        if inAir and not isJumpingState and verticalVelocity > -60 then
            inAir = false
        end

        if not inAir and jumpAnimTrack then
            jumpAnimTrack.TimePosition = 0
        end
        
        -- Capa superior de herida (Overlay para Torso/Brazos/Cabeza)
        -- Se reproduce siempre que estemos heridos y en movimiento sobre el suelo
        -- Prioridad Action3 para sobreponerse solo al Torso/Brazos
        if isHurt and isMoving and not inAir and not isClimbing and not isSwimming and not isSeated and not isExternal then
            local hurtId = 83860986564910
            if not hurtOverlayTrack or hurtOverlayTrack.Animation.AnimationId ~= "rbxassetid://"..hurtId then
                if hurtOverlayTrack then hurtOverlayTrack:Stop(0.1) end
                local anim = Instance.new("Animation")
                anim.AnimationId = "rbxassetid://"..hurtId
                hurtOverlayTrack = humanoid:LoadAnimation(anim)
                hurtOverlayTrack.Priority = Enum.AnimationPriority.Action3 -- Prioridad alta para sobreponerse
                hurtOverlayTrack.Looped = true
                hurtOverlayTrack:Play(0.2)
            elseif not hurtOverlayTrack.IsPlaying then
                hurtOverlayTrack:Play(0.2)
            end
        else
            if hurtOverlayTrack then hurtOverlayTrack:Stop(0.4) hurtOverlayTrack = nil end
        end

        -- Verificar si el emote actual es uno de los seleccionables para detenerlo al moverte
        local currentActiveId = currentAnimTrack and currentAnimTrack.Animation.AnimationId:match("%d+")
        if currentActiveId and managedAnimationIds[currentActiveId] and currentActiveId ~= string.format("%.0f", idleAnimId) and currentActiveId ~= string.format("%.0f", hurtAnimId) then
            isEmotePlaying = true
        end

        -- Solo detenemos el emote si hay una intención real de caminar (Teclas o Joystick a fondo)
        if isEmotePlaying and moveMag > 0.5 then
            stopAnimation()
            isEmotePlaying = false
        end

        if not inAir and not isSeated and not isClimbing and not isSwimming and not isExternal then
            if isMoving and humanoid.MoveDirection.Magnitude > 0 then
                lastWalkStationaryPos = nil
                if walkPos then walkPos.MaxForce = Vector3.new(0, 0, 0) end

                local moveDir = humanoid.MoveDirection
                local currentY = hrp.AssemblyLinearVelocity.Y
                local finalSpeed = originalWalkSpeed -- Default 15

                if isHurt then
                    finalSpeed = 10
                elseif isWalkSpeedActive then
                    finalSpeed = walkSpeed
                end
                
                -- Sistema de Impulso Suave
                local targetVel = moveDir * finalSpeed
                local lerpWeight = isWalkSpeedActive and 0.12 or 0.25
                currentImpulseVelocity = currentImpulseVelocity:Lerp(targetVel, lerpWeight)

                if isWalkSpeedActive then
                    -- MÉTODO DE BYPASS ESCALABLE (20 - 200 KM/H)
                    -- 1. Obtenemos la velocidad física que el servidor "acepta"
                    local serverLimit = 17.8
                    if walkSpeed >= 21 and walkSpeed < 50 then
                        serverLimit = 21 -- Límite exacto solicitado
                    elseif walkSpeed >= 50 then
                        serverLimit = 21.5
                    end

                    -- 2. Calculamos el excedente (boost) para llegar a la velocidad objetivo
                    local boost = math.max(0, finalSpeed - serverLimit)

                    -- 3. Aplicamos CFrame Boost: Aumentamos el jitter si la velocidad es extrema (>100)
                    -- Esto hace que el movimiento sea menos predecible para el servidor a altas velocidades.
                    local jitterIntensity = (finalSpeed > 100) and 0.04 or 0.01
                    local moveStep = (moveDir * (boost + (math.random(-1, 1) * jitterIntensity)) * deltaTime)
                    hrp.CFrame = hrp.CFrame + moveStep

                    -- 4. Inyectamos la velocidad física máxima permitida para "justificar" el movimiento ante el servidor
                    -- Usamos el vector de movimiento actual multiplicado por el límite seguro
                    hrp.AssemblyLinearVelocity = Vector3.new(moveDir.X * serverLimit, hrp.AssemblyLinearVelocity.Y, moveDir.Z * serverLimit)
                else
                    humanoid.WalkSpeed = finalSpeed
                    hrp.AssemblyLinearVelocity = Vector3.new(currentImpulseVelocity.X, currentY, currentImpulseVelocity.Z)
                end

                -- Sincronización especial para Noclip si está activo
                if isInvisibilityActive and invisibilitySeat then
                    invisibilitySeat.AssemblyLinearVelocity = hrp.AssemblyLinearVelocity
                end
            else
                -- Frenado progresivo para la sensación de impulso
                currentImpulseVelocity = currentImpulseVelocity:Lerp(Vector3.new(0,0,0), 0.15)

                -- IMPORTANTE: evitar “salto pesado” por el anclaje BodyPosition.
                -- Si estamos saltando o en el aire, desactivamos la fuerza del walkPos temporalmente.
                if (isJumpingState or inAir) and walkPos then
                    walkPos.MaxForce = Vector3.new(0, 0, 0)
                    -- opcional: mantenerlo pegado a la posición actual para que no “tire” al volver
                    walkPos.Position = hrp.Position
                end
                
                -- Sistema de anclaje para evitar que la física externa rompa el Idle
                if not lastWalkStationaryPos and hrp then 
                    lastWalkStationaryPos = hrp.Position 
                end
                if not walkPos and hrp then
                    walkPos = Instance.new("BodyPosition")
                    walkPos.Name = "WalkAnchor"
                    walkPos.MaxForce = Vector3.new(0, 0, 0)
                    walkPos.D = 500
                    walkPos.P = 10000
                    walkPos.Parent = hrp
                end
                
                -- Al estar quieto, nos aseguramos que el WalkSpeed sea el base (8)
                humanoid.WalkSpeed = originalWalkSpeed

                if walkPos and lastWalkStationaryPos and not isJumpingState and not inAir then
                    -- Si venimos de una velocidad muy alta, el anclaje debe ser más fuerte para evitar el retroceso por inercia
                    local anchorForce = 1e6
                    walkPos.MaxForce = Vector3.new(anchorForce, 0, anchorForce) 
                    walkPos.P = 20000
                    walkPos.Position = Vector3.new(lastWalkStationaryPos.X, hrp.Position.Y, lastWalkStationaryPos.Z)
                    hrp.AssemblyLinearVelocity = Vector3.new(0, hrp.AssemblyLinearVelocity.Y, 0)
                elseif walkPos then
                    walkPos.MaxForce = Vector3.new(0, 0, 0)
                end

                if isInvisibilityActive and invisibilitySeat then
                    invisibilitySeat.AssemblyLinearVelocity = Vector3.zero
                end
            end
        end

        local canPlayWalk = not isCrouching and (not isSprintActive or isHurt) and not isMirageSpeedActive and not isEmotePlaying and not inAir
        local canSprint = isSprintActive and not isCrouching and not isMirageSpeedActive and not isEmotePlaying and not inAir and not isHurt

        if (isClimbing or isSwimming) and not isEmotePlaying then
            if currentAnimTrack and currentAnimTrack.IsPlaying then currentAnimTrack:Stop(0.2) end
            stopSprintAnimation()
            if walkAnimTrack and walkAnimTrack.IsPlaying then walkAnimTrack:Stop(0.2) end
            if jumpAnimTrack then jumpAnimTrack:Stop(0.2) end
            if fallAnimTrack then fallAnimTrack:Stop(0.2) end
        elseif isSeated then
            stopSprintAnimation()
            if walkAnimTrack and walkAnimTrack.IsPlaying then walkAnimTrack:Stop(0.1) end
            if jumpAnimTrack then jumpAnimTrack:Stop(0.1) end
            if fallAnimTrack then fallAnimTrack:Stop(0.1) end
        elseif inAir and not isExternal and not SuperStrength.GrabbedData then
            lastWalkStationaryPos = nil -- Resetear ancla al estar en el aire
            if isEmotePlaying then 
                if math.abs(verticalVelocity) > 20 or moveMag > 0.5 then
                    stopAnimation() 
                    isEmotePlaying = false 
                else
                    -- Si es un pequeño rebote de física, no interrumpir el emote
                    return 
                end
            end
            
            if currentAnimTrack and currentAnimTrack.IsPlaying then
                currentAnimTrack:Stop(0.2)
                currentAnimTrack = nil
            end
            stopSprintAnimation()
            if walkAnimTrack and walkAnimTrack.IsPlaying then walkAnimTrack:Stop(0.2) end

            -- Aseguramos que los tracks de aire estén cargados
            if not jumpAnimTrack then
                local anim = Instance.new("Animation")
                anim.AnimationId = "rbxassetid://" .. ANIMATIONS.Jump
                jumpAnimTrack = humanoid:LoadAnimation(anim)
                jumpAnimTrack.Looped = true
                jumpAnimTrack.Priority = Enum.AnimationPriority.Action4
            end
            if not fallAnimTrack then
                local anim = Instance.new("Animation")
                anim.AnimationId = "rbxassetid://" .. ANIMATIONS.Fall
                fallAnimTrack = humanoid:LoadAnimation(anim)
                fallAnimTrack.Looped = true
                fallAnimTrack.Priority = Enum.AnimationPriority.Action4
            end

            -- Pose de Caída forzada para Salto Gravitatorio (la de caída, no la de salto)
            if isGravJumpActive then
                if jumpAnimTrack and jumpAnimTrack.IsPlaying then jumpAnimTrack:Stop(0.2) end
                if fallAnimTrack and not fallAnimTrack.IsPlaying then
                    fallAnimTrack:Play(0.2)
                end
            elseif verticalVelocity > 0 then
                if fallAnimTrack and fallAnimTrack.IsPlaying then fallAnimTrack:Stop(0.2) end
                if jumpAnimTrack and not jumpAnimTrack.IsPlaying then
                    jumpAnimTrack:Play(0.2)
                    jumpAnimTrack:AdjustSpeed(1)
                end
            else
                if jumpAnimTrack and jumpAnimTrack.IsPlaying then jumpAnimTrack:Stop(0.2) end
                if fallAnimTrack and not fallAnimTrack.IsPlaying then
                    fallAnimTrack:Play(0.2)
                end
            end
        elseif isMoving and canSprint then
            if currentAnimTrack and currentAnimTrack.IsPlaying then 
                currentAnimTrack:Stop(0.2)
                currentAnimTrack = nil
            end
            playSprintAnimation()
            if walkAnimTrack and walkAnimTrack.IsPlaying then walkAnimTrack:Stop(0.2) end
            if jumpAnimTrack then jumpAnimTrack:Stop(0.2) end
            if fallAnimTrack then fallAnimTrack:Stop(0.2) end
        elseif isMoving and canPlayWalk then
            if currentAnimTrack and currentAnimTrack.IsPlaying then
                currentAnimTrack:Stop(0.1)
                currentAnimTrack = nil
            end
            stopSprintAnimation()
            if jumpAnimTrack then jumpAnimTrack:Stop(0.2) end
            if fallAnimTrack then fallAnimTrack:Stop(0.2) end
            local currentSpeed = isWalkSpeedActive and walkSpeed or humanoid.WalkSpeed
            -- Si está herido, forzamos la velocidad efectiva a 11 para usar la animación de caminata base
            -- Forzamos velocidad 11 (ID 11) si está herido para las piernas
            local effectiveSpeed = isHurt and 11 or (isWalkSpeedActive and walkSpeed or originalWalkSpeed)
            local animIdToUse = ANIMATIONS.Walk
            
            -- Determinamos la animación de las piernas según la fase de velocidad
            if effectiveSpeed >= 50 then -- Fase 5
                animIdToUse = 78510387198062
            elseif effectiveSpeed >= 23 then -- Fase 4 (Run) (Cubre 23 y 45)
                animIdToUse = ANIMATIONS.Run
            elseif effectiveSpeed >= 20 then -- Fase 3
                animIdToUse = 72301599441680
            else
                animIdToUse = ANIMATIONS.Walk
            end

            local animIdStr = "rbxassetid://" .. animIdToUse
            if not walkAnimTrack or walkAnimTrack.Animation.AnimationId ~= animIdStr then
                if walkAnimTrack then walkAnimTrack:Stop() end
                walkAnimTrack = animationTracks[animIdToUse] or (function()
                    local a = Instance.new("Animation") a.AnimationId = animIdStr
                    local t = humanoid:LoadAnimation(a) 

                    -- Prioridad alta para herida para que sea fluida
                    if animIdToUse == 83860986564910 or hasTool then
                        t.Priority = Enum.AnimationPriority.Action2
                    elseif animIdToUse == 78510387198062 then
                        t.Priority = Enum.AnimationPriority.Action2
                    else
                        t.Priority = Enum.AnimationPriority.Movement
                    end
                    t.Looped = true
                    animationTracks[animIdToUse] = t return t
                end)()
                walkAnimTrack:Play(0.05)
            elseif not walkAnimTrack.IsPlaying then
                walkAnimTrack:Play(0.05)
            end
            
            local animSpeedMultiplier = 1 
            
            -- Ajuste para caminata herida (valor 10)
            if isHurt then
                animSpeedMultiplier = 0.9 -- Un poco más lenta para realismo
            end

            -- Lógica de velocidad de animación según fase
            if isWalkSpeedActive then
                if effectiveSpeed == 20 then -- Fase 3
                    animSpeedMultiplier = 1
                elseif effectiveSpeed == 23 then -- Fase 4
                    animSpeedMultiplier = 1 -- Original
                elseif effectiveSpeed >= 45 and effectiveSpeed < 50 then -- Fase 4 (45)
                    animSpeedMultiplier = 2.8 -- Animación rápida activada en 45
                elseif effectiveSpeed >= 50 then -- Fase 5 (50+)
                    animSpeedMultiplier = 3.5 + (effectiveSpeed / 50) -- Muy rápida y escala
                end
            end

            walkAnimTrack:AdjustSpeed(math.clamp(animSpeedMultiplier, 0.1, 35))
        else
            stopSprintAnimation()
            if walkAnimTrack and walkAnimTrack.IsPlaying then walkAnimTrack:Stop(0.1) end
            if jumpAnimTrack and jumpAnimTrack.IsPlaying then jumpAnimTrack:Stop(0.1) end
            if fallAnimTrack and fallAnimTrack.IsPlaying then fallAnimTrack:Stop(0.1) end

            if isHurt and not isEmotePlaying then
                playAnimation(hurtAnimId, 0, 1, true)
            else
                if not isEmotePlaying and not isCrouching and not isMirageSpeedActive and not isExternal then
                if isExternal then -- Se eliminó 'hasTool' de esta condición
                        -- Si hay herramienta o animación externa, detenemos nuestro Idle por completo.
                        -- Esto permite que Brookhaven use sus propias poses (Menú F) sin conflicto.
                        if currentAnimTrack and currentAnimTrack.Animation.AnimationId == "rbxassetid://" .. idleAnimId then
                            currentAnimTrack:Stop(0.3)
                            currentAnimTrack = nil
                        end
                    elseif not (currentAnimTrack and currentAnimTrack.IsPlaying and currentAnimTrack.Animation.AnimationId == "rbxassetid://" .. idleAnimId) then
                        playAnimation(idleAnimId, 0, 1, true)
                        if currentAnimTrack then
                            currentAnimTrack.Priority = Enum.AnimationPriority.Movement
                        end
                    end
                end
            end
        end

        for _, track in ipairs(humanoid:GetPlayingAnimationTracks()) do
            if track.Animation then
                local animId = track.Animation.AnimationId
                if LOOP_ANIMATION_IDS[animId] and not track.Looped then track.Looped = true end
                if animId == "rbxassetid://16738340646" then track:AdjustSpeed(1) end
            end
        end
    end
    end)
    table.insert(allConnections, conn)
end

-- LOOP DE LOCK-ON (Cámara)
-- (Conexión administrada globalmente para evitar acumulación al respawnear)
local lockOnConn = nil
lockOnConn = RunService.RenderStepped:Connect(function()

    if scriptActive and lockedPlayer then
        if lockedPlayer.Parent and lockedPlayer.Character and lockedPlayer.Character:FindFirstChild("HumanoidRootPart") then
            local targetHrp = lockedPlayer.Character.HumanoidRootPart
            local hum = lockedPlayer.Character:FindFirstChild("Humanoid")
            if hum and hum.Health > 0 then
                -- Suavizado ligero para que no sea una cámara "tiesa"
                local targetPos = targetHrp.Position
                camera.CFrame = CFrame.lookAt(camera.CFrame.Position, targetPos)
            else
                lockedPlayer = nil
            end
        else
            lockedPlayer = nil
        end
    end
end)


player.CharacterAdded:Connect(function(newChar)
    if not scriptActive then return end
    -- Limpiar conexiones previas sin resetear los valores de las habilidades
    cleanupEverything(false)

    -- Re-crear lock-on tras el respawn (para no perder la función)
    if not lockOnConn or not lockOnConn.Connected then
        lockOnConn = RunService.RenderStepped:Connect(function()
            if scriptActive and lockedPlayer then
                if lockedPlayer.Parent and lockedPlayer.Character and lockedPlayer.Character:FindFirstChild("HumanoidRootPart") then
                    local targetHrp = lockedPlayer.Character.HumanoidRootPart
                    local hum = lockedPlayer.Character:FindFirstChild("Humanoid")
                    if hum and hum.Health > 0 then
                        local targetPos = targetHrp.Position
                        camera.CFrame = CFrame.lookAt(camera.CFrame.Position, targetPos)
                    else
                        lockedPlayer = nil
                    end
                else
                    lockedPlayer = nil
                end
            end
        end)
    end

    
    character = newChar
    humanoid = newChar:WaitForChild("Humanoid")
    isR6 = (humanoid.RigType == Enum.HumanoidRigType.R6)
    humanoid.WalkSpeed = originalWalkSpeed 
    hrp = newChar:FindFirstChild("HumanoidRootPart")
    scriptActive = true
    
    hideIdentity()
    local reloadSuccess = loadAnimations()
    startMainAnimationLoop()
    startReplicationLoop()
    if reloadSuccess then
        task.wait(0.2)
        -- Restaurar todas las habilidades que estaban activas de forma independiente
        if isSuperJumpActive then startSuperJump() end
        if SuperHearing.Active then startSuperHearing() end
        if isMirageSpeedActive then playMirageSpeedEmote() end
        if isInvisibilityActive then toggleInvisibility() end
        if ThermalVision.Active then toggleThermalVision() end
        if isFlying then startFlight() end
        if noSitActive and humanoid then pcall(function() humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, false) end) end
        if not isAbilityAvailable(abilities[currentAbilityIndex].key) then
            currentAbilityIndex = 1
        end
        if isR6 and currentMode == MODES.EMOTES then
            currentMode = MODES.ABILITIES
        end
        if not isR6 and settingOptions[currentSettingIndex]:find("R6 FLY") then
            currentSettingIndex = 1
        end
    end
    updateMainUI()
end)

-- INICIALIZACIÓN
createUI()
setupGestures()
forceInitialLoad()
startMainAnimationLoop()
startReplicationLoop()

-- Anti-AFK (parche anti-limit 200: 0 conexiones extra)
-- Nota: en algunos exploit loaders, VirtualUser/Idled:Connect se acumula y rompe el límite.
-- Para evitar el crash, se desactiva el Anti-AFK pero el resto del script queda intacto.
-- (Si quieres Anti-AFK real, hay que hacerlo con una única conexión persistente por sesión del executor.)
