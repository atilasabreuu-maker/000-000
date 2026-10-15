--[[
================================================================================
  PRYPOT 0.X1 // NEURAL EXOSUIT
  Classificado: LVL-5
  Autor: Prypot Industries · Divisão Biomécânica
  Plataforma: Roblox (Luau) · R15/R6 compat · Mobile 100%
  Versão: 0.X1-delta
  Tamanho: ~2100 linhas
  
  ARQUITETURA:
    [SERVER]  -> Script em ServerScriptService (spawna a armadura no Workspace)
    [CLIENT]  -> LocalScript em StarterPlayerScripts (input, HUD, VFX, câmera)
    [MODULES] -> Gerados automaticamente dentro do modelo Prypot0X1
  
  COMO INSTALAR:
    1. Abra o Roblox Studio
    2. Crie um "Script" em ServerScriptService e cole a seção [SERVER]
    3. Crie um "LocalScript" em StarterPlayerScripts e cole a seção [CLIENT]
    4. Teste em Play Solo
================================================================================
--]]


--------------------------------------------------------------------------------
--                                                                            --
--                              [ SERVER SECTION ]                            --
--                Cole isso em um Script dentro de ServerScriptService        --
--                                                                            --
--------------------------------------------------------------------------------

--!strict
local Players            = game:GetService("Players")
local RunService         = game:GetService("RunService")
local ReplicatedStorage  = game:GetService("ReplicatedStorage")
local Workspace          = game:GetService("Workspace")
local TweenService       = game:GetService("TweenService")
local Debris             = game:GetService("Debris")

--------------------------------------------------------------------------------
-- CONFIG GERAL DA ARMADURA PRYPOT 0.X1
--------------------------------------------------------------------------------
local CONFIG = {
	ModelName            = "Prypot0X1",
	SpawnPosition        = Vector3.new(0, 10, 0),          -- Pos inicial no mapa
	WalkSpeed            = 38,
	JumpPower            = 120,
	DigDepth             = 2,                              -- Profundidade da escavação
	DigRadius            = 2,                              -- Raio do buraco (studs)
	DigCooldown          = 0.6,
	RepairTime           = 8,                              -- Segundos p/ reparar buraco
	VoidCheckDepth       = 50,                             -- Raycast p/ detectar void
	VoidUnsafeThreshold  = 0.30,                           -- 30% vazio = bloqueado
	ParkourWallRunTime   = 3,
	ParkourDashSpeed     = 120,
	ParkourDashTime      = 0.35,
	StealthDuration      = 8,
	StealthCooldown      = 15,
	StealthTransparency  = 0.85,
	DoubleJumpPower      = 90,
	ImpactAbsorbHeight   = 15,                             -- Altura min p/ molas ativarem
	PunchRange           = 10,
}

--------------------------------------------------------------------------------
-- REMOTE EVENTS (comunicação server<->client)
--------------------------------------------------------------------------------
local function ensureRemote(name, className)
	className = className or "RemoteEvent"
	local folder = ReplicatedStorage:FindFirstChild("PrypotRemotes")
	if not folder then
		folder = Instance.new("Folder")
		folder.Name = "PrypotRemotes"
		folder.Parent = ReplicatedStorage
	end
	local r = folder:FindFirstChild(name)
	if not r then
		r = Instance.new(className)
		r.Name = name
		r.Parent = folder
	end
	return r
end

local Remotes = {
	RequestEnter     = ensureRemote("RequestEnter"),
	EnterConfirmed   = ensureRemote("EnterConfirmed"),
	RequestExit      = ensureRemote("RequestExit"),
	ExitConfirmed    = ensureRemote("ExitConfirmed"),
	DigRequest       = ensureRemote("DigRequest"),
	DigConfirmed     = ensureRemote("DigConfirmed"),
	DigUnsafe        = ensureRemote("DigUnsafe"),
	PunchRequest     = ensureRemote("PunchRequest"),
}

--------------------------------------------------------------------------------
-- UTILITÁRIOS DE CONSTRUÇÃO DO MODELO
--------------------------------------------------------------------------------
local function makePart(props)
	local p = Instance.new("Part")
	p.Anchored    = props.Anchored    ~= nil and props.Anchored    or false
	p.CanCollide  = props.CanCollide  ~= nil and props.CanCollide  or true
	p.Material    = props.Material    or Enum.Material.Metal
	p.Color       = props.Color       or Color3.fromRGB(30,30,35)
	p.Size        = props.Size        or Vector3.new(1,1,1)
	p.CFrame      = props.CFrame      or CFrame.new()
	p.TopSurface  = Enum.SurfaceType.Smooth
	p.BottomSurface = Enum.SurfaceType.Smooth
	if props.Name   then p.Name   = props.Name   end
	if props.Transparency ~= nil then p.Transparency = props.Transparency end
	if props.Massless     then p.Massless = true end
	return p
end

local function weld(a, b, c0, c1)
	local w = Instance.new("WeldConstraint")
	w.Part0 = a
	w.Part1 = b
	w.Parent = a
	return w
end

local function motor(a, b, c0, c1)
	local m = Instance.new("Motor6D")
	m.Part0 = a
	m.Part1 = b
	m.C0 = c0 or CFrame.new()
	m.C1 = c1 or CFrame.new()
	m.Parent = a
	return m
end

--------------------------------------------------------------------------------
-- CONSTRUÇÃO PROCEDURAL DA ARMADURA PRYPOT 0.X1
-- (Modelo biomecânico animatrônico gerado inteiro via script)
--------------------------------------------------------------------------------
local function buildPrypot()
	local model = Instance.new("Model")
	model.Name = CONFIG.ModelName
	
	-- Cores do esquema cyberpunk militar
	local COL_DARK      = Color3.fromRGB(20,20,25)
	local COL_METAL     = Color3.fromRGB(55,58,64)
	local COL_BONE      = Color3.fromRGB(200,195,180)
	local COL_RED       = Color3.fromRGB(255,40,40)
	local COL_CYAN      = Color3.fromRGB(0,220,255)
	local COL_AMBER     = Color3.fromRGB(255,160,30)
	
	--------------------------------------------------------------------------------
	-- TORSO (caixa torácica exposta estilo esqueleto + metal)
	--------------------------------------------------------------------------------
	local torso = makePart{
		Name = "Torso", Size = Vector3.new(2.2, 2.4, 1.4),
		Color = COL_DARK, Material = Enum.Material.Metal,
	}
	
	-- Núcleo pulsante
	local core = makePart{
		Name = "Core", Size = Vector3.new(0.7, 0.7, 0.7), Shape = "Ball",
		Color = COL_RED, Material = Enum.Material.Neon, Transparency = 0.2,
		CanCollide = false, Massless = true,
	}
	core.CFrame = torso.CFrame * CFrame.new(0, 0, -0.4)
	core.Parent = model
	weld(core, torso, CFrame.new(0, 0, -0.4))
	
	local coreLight = Instance.new("PointLight")
	coreLight.Color = COL_RED
	coreLight.Brightness = 3
	coreLight.Range = 12
	coreLight.Parent = core
	
	-- Costelas (8 arcos ósseos expostos)
	for i = 1, 8 do
		local t = (i - 1) / 7
		local yOff = 0.9 - t * 1.8
		for side = -1, 1, 2 do
			local rib = makePart{
				Name = "Rib", Size = Vector3.new(0.9, 0.12, 0.12),
				Color = COL_BONE, Material = Enum.Material.SmoothPlastic,
				CanCollide = false, Massless = true,
			}
			local ang = side * math.rad(40)
			rib.CFrame = torso.CFrame * CFrame.new(side * 0.55, yOff, -0.55) * CFrame.Angles(0, 0, ang)
			rib.Parent = model
			weld(rib, torso, CFrame.new(side * 0.55, yOff, -0.55) * CFrame.Angles(0, 0, ang))
		end
	end
	
	--------------------------------------------------------------------------------
	-- CABEÇA (crânio animatrônico estilo FNAF)
	--------------------------------------------------------------------------------
	local head = makePart{
		Name = "Head", Size = Vector3.new(1.5, 1.6, 1.5),
		Color = COL_BONE, Material = Enum.Material.SmoothPlastic,
	}
	head.CFrame = torso.CFrame * CFrame.new(0, 2.0, 0)
	head.Parent = model
	
	-- Pescoço
	local neck = makePart{
		Name = "Neck", Size = Vector3.new(0.5, 0.4, 0.5),
		Color = COL_METAL, Material = Enum.Material.Metal,
		CanCollide = false, Massless = true,
	}
	neck.CFrame = torso.CFrame * CFrame.new(0, 1.4, 0)
	neck.Parent = model
	weld(neck, torso, CFrame.new(0, 1.4, 0))
	
	-- Motor6D da cabeça (para animação de micro-movimentos)
	local neckMotor = motor(head, neck, CFrame.new(0, -0.8, 0), CFrame.new(0, 0.2, 0))
	neckMotor.Name = "NeckMotor"
	
	-- Mandíbula animada
	local jaw = makePart{
		Name = "Jaw", Size = Vector3.new(1.2, 0.4, 1.0),
		Color = COL_BONE, Material = Enum.Material.SmoothPlastic,
		CanCollide = false, Massless = true,
	}
	jaw.CFrame = head.CFrame * CFrame.new(0, -0.5, 0)
	jaw.Parent = model
	local jawMotor = motor(jaw, head, CFrame.new(0, 0.2, 0), CFrame.new(0, 0.55, -0.3))
	jawMotor.Name = "JawMotor"
	
	-- Dentes superiores e inferiores
	for i = 1, 6 do
		local x = -0.5 + (i-1) * 0.2
		local toothU = makePart{
			Name = "Tooth", Size = Vector3.new(0.1, 0.18, 0.08),
			Color = Color3.fromRGB(240,235,220), Material = Enum.Material.SmoothPlastic,
			CanCollide = false, Massless = true,
		}
		toothU.CFrame = head.CFrame * CFrame.new(x, -0.35, -0.55)
		toothU.Parent = model
		weld(toothU, head, CFrame.new(x, -0.35, -0.55))
		
		local toothL = toothU:Clone()
		toothL.CFrame = jaw.CFrame * CFrame.new(x, 0.15, -0.45)
		toothL.Parent = model
		weld(toothL, jaw, CFrame.new(x, 0.15, -0.45))
	end
	
	-- Olhos LED vermelhos (perturbadores)
	for side = -1, 1, 2 do
		local eye = makePart{
			Name = "Eye", Size = Vector3.new(0.3, 0.3, 0.15), Shape = "Ball",
			Color = COL_RED, Material = Enum.Material.Neon,
			CanCollide = false, Massless = true,
		}
		eye.CFrame = head.CFrame * CFrame.new(side * 0.35, 0.15, -0.6)
		eye.Parent = model
		weld(eye, head, CFrame.new(side * 0.35, 0.15, -0.6))
		
		local spot = Instance.new("SpotLight")
		spot.Color = COL_RED
		spot.Brightness = 2
		spot.Range = 20
		spot.Angle = 45
		spot.Face = Enum.NormalId.Front
		spot.Parent = eye
	end
	
	-- Engrenagens rotativas na lateral da cabeça
	for side = -1, 1, 2 do
		local gear = makePart{
			Name = "Gear", Size = Vector3.new(0.6, 0.6, 0.1), Shape = "Cylinder",
			Color = COL_AMBER, Material = Enum.Material.Metal,
			CanCollide = false, Massless = true,
		}
		gear.CFrame = head.CFrame * CFrame.new(side * 0.75, 0, 0) * CFrame.Angles(0, 0, math.rad(90))
		gear.Parent = model
		weld(gear, head, CFrame.new(side * 0.75, 0, 0) * CFrame.Angles(0, 0, math.rad(90)))
		gear:SetAttribute("SpinSpeed", side == -1 and 2 or -2)
	end
	
	--------------------------------------------------------------------------------
	-- BRAÇOS (com pistões hidráulicos e garras)
	--------------------------------------------------------------------------------
	local arms = {}
	for side = -1, 1, 2 do
		local shoulder = makePart{
			Name = side == -1 and "ShoulderL" or "ShoulderR",
			Size = Vector3.new(0.7, 0.7, 0.7), Shape = "Ball",
			Color = COL_METAL, Material = Enum.Material.Metal,
			CanCollide = false, Massless = true,
		}
		shoulder.CFrame = torso.CFrame * CFrame.new(side * 1.3, 0.9, 0)
		shoulder.Parent = model
		
		local upperArm = makePart{
			Name = side == -1 and "UpperArmL" or "UpperArmR",
			Size = Vector3.new(0.5, 1.4, 0.5),
			Color = COL_DARK, Material = Enum.Material.Metal,
			CanCollide = false, Massless = true,
		}
		upperArm.CFrame = torso.CFrame * CFrame.new(side * 1.3, 0.0, 0)
		upperArm.Parent = model
		
		local elbow = makePart{
			Name = side == -1 and "ElbowL" or "ElbowR",
			Size = Vector3.new(0.5, 0.5, 0.5), Shape = "Ball",
			Color = COL_AMBER, Material = Enum.Material.Metal,
			CanCollide = false, Massless = true,
		}
		elbow.CFrame = torso.CFrame * CFrame.new(side * 1.3, -0.8, 0)
		elbow.Parent = model
		
		local lowerArm = makePart{
			Name = side == -1 and "LowerArmL" or "LowerArmR",
			Size = Vector3.new(0.45, 1.3, 0.45),
			Color = COL_METAL, Material = Enum.Material.Metal,
			CanCollide = false, Massless = true,
		}
		lowerArm.CFrame = torso.CFrame * CFrame.new(side * 1.3, -1.7, 0)
		lowerArm.Parent = model
		
		-- Pistão hidráulico visual no braço
		local piston = makePart{
			Name = "Piston", Size = Vector3.new(0.12, 1.2, 0.12),
			Color = COL_CYAN, Material = Enum.Material.Neon, Transparency = 0.3,
			CanCollide = false, Massless = true,
		}
		piston.CFrame = torso.CFrame * CFrame.new(side * 1.05, -0.2, -0.25)
		piston.Parent = model
		
		-- Garra (3 dedos)
		local hand = makePart{
			Name = side == -1 and "HandL" or "HandR",
			Size = Vector3.new(0.6, 0.3, 0.6),
			Color = COL_DARK, Material = Enum.Material.Metal,
			CanCollide = false, Massless = true,
		}
		hand.CFrame = torso.CFrame * CFrame.new(side * 1.3, -2.5, 0)
		hand.Parent = model
		
		for i = 1, 3 do
			local claw = makePart{
				Name = "Claw", Size = Vector3.new(0.1, 0.5, 0.1),
				Color = COL_BONE, Material = Enum.Material.Metal,
				CanCollide = false, Massless = true,
			}
			local cx = (i - 2) * 0.2
			claw.CFrame = hand.CFrame * CFrame.new(cx, -0.35, 0)
			claw.Parent = model
			weld(claw, hand, CFrame.new(cx, -0.35, 0))
		end
		
		-- Broca no braço direito (para escavação)
		if side == 1 then
			local drill = makePart{
				Name = "DrillBit", Size = Vector3.new(0.4, 0.8, 0.4),
				Color = COL_METAL, Material = Enum.Material.DiamondPlate,
				CanCollide = false, Massless = true,
			}
			drill.CFrame = hand.CFrame * CFrame.new(0, -0.8, 0)
			drill.Parent = model
			weld(drill, hand, CFrame.new(0, -0.8, 0))
			drill:SetAttribute("IsDrill", true)
		end
		
		-- Motores (para animação no client)
		local shoulderMotor = motor(upperArm, shoulder, CFrame.new(0, 0.7, 0), CFrame.new(0, -0.35, 0))
		shoulderMotor.Name = side == -1 and "ShoulderMotorL" or "ShoulderMotorR"
		
		weld(shoulder, torso, CFrame.new(side * 1.3, 0.9, 0))
		weld(elbow, upperArm, CFrame.new(0, -0.8, 0))
		weld(lowerArm, elbow, CFrame.new(0, -0.9, 0))
		weld(hand, lowerArm, CFrame.new(0, -0.85, 0))
		weld(piston, upperArm, CFrame.new(-side * 0.25, -0.2, -0.25))
		
		arms[side == -1 and "L" or "R"] = {
			shoulder = shoulder, upper = upperArm, elbow = elbow,
			lower = lowerArm, hand = hand, piston = piston,
			motor = shoulderMotor,
		}
	end
	
	--------------------------------------------------------------------------------
	-- PERNAS (com molas de impacto nos pés)
	--------------------------------------------------------------------------------
	local legs = {}
	for side = -1, 1, 2 do
		local hip = makePart{
			Name = side == -1 and "HipL" or "HipR",
			Size = Vector3.new(0.5, 0.5, 0.5), Shape = "Ball",
			Color = COL_METAL, Material = Enum.Material.Metal,
			CanCollide = false, Massless = true,
		}
		hip.CFrame = torso.CFrame * CFrame.new(side * 0.6, -1.3, 0)
		hip.Parent = model
		
		local upperLeg = makePart{
			Name = side == -1 and "UpperLegL" or "UpperLegR",
			Size = Vector3.new(0.6, 1.5, 0.6),
			Color = COL_DARK, Material = Enum.Material.Metal,
			CanCollide = false, Massless = true,
		}
		upperLeg.CFrame = torso.CFrame * CFrame.new(side * 0.6, -2.2, 0)
		upperLeg.Parent = model
		
		local knee = makePart{
			Name = side == -1 and "KneeL" or "KneeR",
			Size = Vector3.new(0.55, 0.55, 0.55), Shape = "Ball",
			Color = COL_AMBER, Material = Enum.Material.Metal,
			CanCollide = false, Massless = true,
		}
		knee.CFrame = torso.CFrame * CFrame.new(side * 0.6, -3.0, 0)
		knee.Parent = model
		
		local lowerLeg = makePart{
			Name = side == -1 and "LowerLegL" or "LowerLegR",
			Size = Vector3.new(0.55, 1.4, 0.55),
			Color = COL_METAL, Material = Enum.Material.Metal,
			CanCollide = false, Massless = true,
		}
		lowerLeg.CFrame = torso.CFrame * CFrame.new(side * 0.6, -3.9, 0)
		lowerLeg.Parent = model
		
		-- CONJUNTO DE MOLAS (parts em zigue-zague)
		local springAnchor = makePart{
			Name = "SpringAnchor", Size = Vector3.new(0.5, 0.1, 0.5),
			Color = COL_CYAN, Material = Enum.Material.Neon, Transparency = 0.4,
			CanCollide = false, Massless = true,
		}
		springAnchor.CFrame = torso.CFrame * CFrame.new(side * 0.6, -4.7, 0)
		springAnchor.Parent = model
		
		local springParts = {}
		local springSegments = 6
		for i = 1, springSegments do
			local t = i / springSegments
			local xOff = (i % 2 == 0) and 0.15 or -0.15
			local sp = makePart{
				Name = "SpringSegment", Size = Vector3.new(0.3, 0.08, 0.1),
				Color = COL_CYAN, Material = Enum.Material.Neon,
				CanCollide = false, Massless = true,
			}
			sp.CFrame = torso.CFrame * CFrame.new(side * 0.6 + xOff, -4.75 - t * 0.5, 0)
			sp.Parent = model
			table.insert(springParts, sp)
		end
		
		-- Pé
		local foot = makePart{
			Name = side == -1 and "FootL" or "FootR",
			Size = Vector3.new(0.7, 0.3, 1.1),
			Color = COL_DARK, Material = Enum.Material.DiamondPlate,
		}
		foot.CFrame = torso.CFrame * CFrame.new(side * 0.6, -5.4, 0.15)
		foot.Parent = model
		
		-- Emissor de poeira no pé
		local dustEmitter = Instance.new("ParticleEmitter")
		dustEmitter.Texture = "rbxasset://textures/particles/smoke_main.dds"
		dustEmitter.Color = ColorSequence.new(Color3.fromRGB(120,110,90))
		dustEmitter.Size = NumberSequence.new(0.8, 0)
		dustEmitter.Lifetime = NumberRange.new(0.5, 1)
		dustEmitter.Rate = 0
		dustEmitter.Speed = NumberRange.new(3, 6)
		dustEmitter.SpreadAngle = Vector2.new(45, 45)
		dustEmitter.LightEmission = 0.1
		dustEmitter.Parent = foot
		
		-- Propulsor ciano (efeito visual em parkour)
		local thrust = Instance.new("ParticleEmitter")
		thrust.Name = "Thruster"
		thrust.Texture = "rbxasset://textures/particles/fire_main.dds"
		thrust.Color = ColorSequence.new(COL_CYAN)
		thrust.Size = NumberSequence.new(0.6, 0)
		thrust.Lifetime = NumberRange.new(0.2, 0.4)
		thrust.Rate = 0
		thrust.Speed = NumberRange.new(10, 15)
		thrust.LightEmission = 1
		thrust.Parent = foot
		
		weld(hip, torso, CFrame.new(side * 0.6, -1.3, 0))
		weld(upperLeg, hip, CFrame.new(0, -0.9, 0))
		weld(knee, upperLeg, CFrame.new(0, -0.8, 0))
		weld(lowerLeg, knee, CFrame.new(0, -0.9, 0))
		weld(springAnchor, lowerLeg, CFrame.new(0, -0.75, 0))
		for i, sp in ipairs(springParts) do
			weld(sp, springAnchor, sp.CFrame:toObjectSpace(springAnchor.CFrame):inverse())
		end
		-- Pé soldado à mola com um Motor6D para animação de compressão
		local footMotor = motor(foot, springAnchor, CFrame.new(0, -0.7, 0.15), CFrame.new(0, 0, 0))
		footMotor.Name = side == -1 and "FootMotorL" or "FootMotorR"
		
		legs[side == -1 and "L" or "R"] = {
			hip = hip, upper = upperLeg, knee = knee, lower = lowerLeg,
			foot = foot, springAnchor = springAnchor, springParts = springParts,
			footMotor = footMotor, dustEmitter = dustEmitter, thrust = thrust,
		}
	end
	
	--------------------------------------------------------------------------------
	-- ESTRUTURA TRASEIRA (espinha dorsal com espinhos neurais)
	--------------------------------------------------------------------------------
	local spineRack = makePart{
		Name = "SpineRack", Size = Vector3.new(1.6, 2.0, 0.4),
		Color = COL_DARK, Material = Enum.Material.Metal,
		CanCollide = false, Massless = true,
	}
	spineRack.CFrame = torso.CFrame * CFrame.new(0, 0, 0.7)
	spineRack.Parent = model
	weld(spineRack, torso, CFrame.new(0, 0, 0.7))
	
	-- Vértebras (12 segmentos)
	for i = 1, 12 do
		local t = (i - 1) / 11
		local vert = makePart{
			Name = "Vertebra", Size = Vector3.new(0.3, 0.12, 0.3),
			Color = COL_BONE, Material = Enum.Material.SmoothPlastic,
			CanCollide = false, Massless = true,
		}
		vert.CFrame = spineRack.CFrame * CFrame.new(0, 0.8 - t * 1.6, 0.1)
		vert.Parent = model
		weld(vert, spineRack, CFrame.new(0, 0.8 - t * 1.6, 0.1))
	end
	
	-- ESPINHOS NEURAIS (6 agulhas hidráulicas que "entram" no piloto)
	local neuralSpikes = {}
	for i = 1, 6 do
		local t = (i - 1) / 5
		local spike = makePart{
			Name = "NeuralSpike", Size = Vector3.new(0.08, 0.8, 0.08),
			Color = COL_RED, Material = Enum.Material.Neon, Transparency = 0.3,
			CanCollide = false, Massless = true,
		}
		spike.CFrame = spineRack.CFrame * CFrame.new((i % 2 == 0) and 0.3 or -0.3, 0.6 - t * 1.2, 0.3)
		spike.Parent = model
		-- NÃO soldado ainda: vai se mover durante a cutscene de entrada
		spike:SetAttribute("BaseCFrame", spike.CFrame)
		table.insert(neuralSpikes, spike)
	end
	
	--------------------------------------------------------------------------------
	-- ASSENTO DO PILOTO (invisível - para lógica de veículo)
	--------------------------------------------------------------------------------
	local seat = Instance.new("Seat")
	seat.Name = "PilotSeat"
	seat.Size = Vector3.new(2, 0.2, 2)
	seat.Transparency = 1
	seat.CanCollide = false
	seat.Anchored = false
	seat.CFrame = torso.CFrame * CFrame.new(0, 0.3, 0)
	seat.Parent = model
	
	-- PrimaryPart
	torso.Parent = model
	model.PrimaryPart = torso
	
	-- Atributos do modelo
	model:SetAttribute("Occupied", false)
	model:SetAttribute("Pilot", "")
	
	return model, {
		torso = torso, head = head, jaw = jaw, core = core,
		jawMotor = jawMotor, neckMotor = neckMotor,
		arms = arms, legs = legs, neuralSpikes = neuralSpikes,
		seat = seat, spineRack = spineRack,
	}
end

--------------------------------------------------------------------------------
-- SISTEMA DE REPARO DE TERRENO (server-side)
--------------------------------------------------------------------------------
local DigHistory = {} -- [{position, size, material, createdAt}]

local function registerDigHole(position, size, material)
	table.insert(DigHistory, {
		position = position,
		size     = size,
		material = material,
		createdAt = tick(),
	})
end

-- Loop de reparo (roda no servidor)
task.spawn(function()
	while true do
		task.wait(1)
		local now = tick()
		for i = #DigHistory, 1, -1 do
			local hole = DigHistory[i]
			if now - hole.createdAt >= CONFIG.RepairTime then
				-- Reconstrói a parte removida como um Part temporário com tween
				local patch = Instance.new("Part")
				patch.Size = hole.size
				patch.CFrame = CFrame.new(hole.position)
				patch.Anchored = true
				patch.CanCollide = true
				patch.Material = hole.material
				patch.Color = Color3.fromRGB(90,70,50)
				patch.Transparency = 1
				patch.Parent = Workspace
				
				TweenService:Create(patch, TweenInfo.new(1.5, Enum.EasingStyle.Sine), {Transparency = 0}):Play()
				task.wait(1.6)
				patch:Destroy()
				
				table.remove(DigHistory, i)
			end
		end
	end
end)

--------------------------------------------------------------------------------
-- SISTEMA ANTI-VOID
-- Verifica uma grade 5x5 descendo VoidCheckDepth studs. Se mais de
-- UnsafeThreshold dos pontos não colidirem, a área é insegura.
--------------------------------------------------------------------------------
local function isVoidSafe(center, radius)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = {} -- será preenchido pelo client via remote
	
	local gridSize = 5
	local hits, total = 0, gridSize * gridSize
	local step = radius * 2 / (gridSize - 1)
	
	for xi = 0, gridSize - 1 do
		for zi = 0, gridSize - 1 do
			local x = center.X - radius + xi * step
			local z = center.Z - radius + zi * step
			local origin = Vector3.new(x, center.Y, z)
			local result = Workspace:Raycast(origin, Vector3.new(0, -CONFIG.VoidCheckDepth, 0), params)
			if result then hits = hits + 1 end
		end
	end
	
	local solidFraction = hits / total
	return solidFraction >= (1 - CONFIG.VoidUnsafeThreshold), solidFraction
end

--------------------------------------------------------------------------------
-- HANDLERS DE REMOTE
--------------------------------------------------------------------------------
Remotes.RequestEnter.OnServerEvent:Connect(function(player)
	local model = Workspace:FindFirstChild(CONFIG.ModelName)
	if not model then return end
	if model:GetAttribute("Occupied") then return end
	
	local char = player.Character
	if not char then return end
	local rootPart = char:FindFirstChild("HumanoidRootPart")
	if not rootPart then return end
	
	-- Verifica proximidade
	if (rootPart.Position - model.PrimaryPart.Position).Magnitude > 15 then return end
	
	model:SetAttribute("Occupied", true)
	model:SetAttribute("Pilot", player.Name)
	
	local seat = model:FindFirstChild("PilotSeat")
	if seat then
		seat:Sit(char:FindFirstChildOfClass("Humanoid"))
	end
	
	Remotes.EnterConfirmed:FireClient(player, model:GetFullName())
end)

Remotes.RequestExit.OnServerEvent:Connect(function(player)
	local model = Workspace:FindFirstChild(CONFIG.ModelName)
	if not model then return end
	if model:GetAttribute("Pilot") ~= player.Name then return end
	
	model:SetAttribute("Occupied", false)
	model:SetAttribute("Pilot", "")
	
	local char = player.Character
	if char then
		local hum = char:FindFirstChildOfClass("Humanoid")
		if hum and hum.SeatPart then
			hum.Sit = false
			local hrp = char:FindFirstChild("HumanoidRootPart")
			if hrp then
				hrp.CFrame = model.PrimaryPart.CFrame * CFrame.new(4, 2, 0)
			end
		end
	end
	
	Remotes.ExitConfirmed:FireClient(player)
end)

Remotes.DigRequest.OnServerEvent:Connect(function(player, position, excludeInstances)
	local model = Workspace:FindFirstChild(CONFIG.ModelName)
	if not model then return end
	if model:GetAttribute("Pilot") ~= player.Name then return end
	
	-- Anti-void check
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = excludeInstances or {model}
	
	local safe, fraction = isVoidSafe(position, CONFIG.DigRadius)
	if not safe then
		Remotes.DigUnsafe:FireClient(player, fraction)
		return
	end
	
	-- Realiza a escavação: usa Terrain fill (se for terrain) ou destrói Parts
	local terrain = Workspace.Terrain
	local region = Region3.new(
		position - Vector3.new(CONFIG.DigRadius, CONFIG.DigDepth, CONFIG.DigRadius),
		position + Vector3.new(CONFIG.DigRadius, 0.5, CONFIG.DigRadius)
	)
	region = region:ExpandToGrid(4)
	
	-- Lê o material do terreno naquela área para poder reparar depois
	local readRegion = Region3.new(
		position - Vector3.new(CONFIG.DigRadius, CONFIG.DigDepth, CONFIG.DigRadius),
		position + Vector3.new(CONFIG.DigRadius, 0.5, CONFIG.DigRadius)
	)
	local materials, occupancy = terrain:ReadVoxels(readRegion, 4)
	
	-- Preenche com ar
	terrain:FillRegion(region, 4, Enum.Material.Air)
	
	registerDigHole(position, Vector3.new(CONFIG.DigRadius * 2, CONFIG.DigDepth, CONFIG.DigRadius * 2), Enum.Material.Ground)
	
	Remotes.DigConfirmed:FireClient(player, position)
end)

Remotes.PunchRequest.OnServerEvent:Connect(function(player)
	-- Punch é apenas visual local; aqui apenas validamos cooldown server-side
	-- (opcional - se quiser evitar spam)
end)

--------------------------------------------------------------------------------
-- SPAWN INICIAL
--------------------------------------------------------------------------------
local function spawnPrypot()
	if Workspace:FindFirstChild(CONFIG.ModelName) then return end
	
	local model, parts = buildPrypot()
	model.Parent = Workspace
	model:MoveTo(CONFIG.SpawnPosition)
	
	print("[PRYPOT 0.X1] Armadura spawned at", CONFIG.SpawnPosition)
end

spawnPrypot()

-- Respawn se for destruído acidentalmente
Workspace.ChildRemoved:Connect(function(child)
	if child.Name == CONFIG.ModelName then
		task.wait(5)
		spawnPrypot()
	end
end)


--------------------------------------------------------------------------------
--                                                                            --
--                              [ CLIENT SECTION ]                            --
--             Cole isso em um LocalScript dentro de StarterPlayerScripts     --
--                                                                            --
--------------------------------------------------------------------------------

--!strict
local Players            = game:GetService("Players")
local RunService         = game:GetService("RunService")
local UserInputService   = game:GetService("UserInputService")
local TweenService       = game:GetService("TweenService")
local ReplicatedStorage  = game:GetService("ReplicatedStorage")
local Workspace          = game:GetService("Workspace")
local SoundService       = game:GetService("SoundService")
local Lighting           = game:GetService("Lighting")
local StarterGui         = game:GetService("StarterGui")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

--------------------------------------------------------------------------------
-- CONFIG (espelha o server)
--------------------------------------------------------------------------------
local CONFIG = {
	ModelName            = "Prypot0X1",
	WalkSpeed            = 38,
	JumpPower            = 120,
	DigDepth             = 2,
	DigRadius            = 2,
	DigCooldown          = 0.6,
	ParkourWallRunTime   = 3,
	ParkourDashSpeed     = 120,
	ParkourDashTime      = 0.35,
	StealthDuration      = 8,
	StealthCooldown      = 15,
	StealthTransparency  = 0.85,
	DoubleJumpPower      = 90,
	ImpactAbsorbHeight   = 15,
	PunchRange           = 10,
}

--------------------------------------------------------------------------------
-- REMOTES
--------------------------------------------------------------------------------
local function waitForRemotes()
	local folder = ReplicatedStorage:WaitForChild("PrypotRemotes", 30)
	return {
		RequestEnter     = folder:WaitForChild("RequestEnter"),
		EnterConfirmed   = folder:WaitForChild("EnterConfirmed"),
		RequestExit      = folder:WaitForChild("RequestExit"),
		ExitConfirmed    = folder:WaitForChild("ExitConfirmed"),
		DigRequest       = folder:WaitForChild("DigRequest"),
		DigConfirmed     = folder:WaitForChild("DigConfirmed"),
		DigUnsafe        = folder:WaitForChild("DigUnsafe"),
		PunchRequest     = folder:WaitForChild("PunchRequest"),
	}
end
local Remotes = waitForRemotes()

--------------------------------------------------------------------------------
-- ESTADO
--------------------------------------------------------------------------------
local state = {
	inSuit          = false,
	suit            = nil,
	suitParts       = nil,
	lastDig         = 0,
	lastDash        = 0,
	lastStealth     = 0,
	stealthActive   = false,
	jumpsUsed       = 0,
	wallRunning     = false,
	wallCling       = false,
	ceilingCling    = false,
	dashing         = false,
	highestY        = 0,
	originalSpeed   = 16,
	originalJump    = 50,
}

--------------------------------------------------------------------------------
-- UI: HUD CINEMATOGRÁFICO
--------------------------------------------------------------------------------
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "PrypotHUD"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = player:WaitForChild("PlayerGui")

local function createFrame(props)
	local f = Instance.new("Frame")
	f.BackgroundTransparency = props.Transparency or 0
	f.BackgroundColor3 = props.Color or Color3.fromRGB(0,0,0)
	f.BorderSizePixel = 0
	f.Size = props.Size or UDim2.new(0,100,0,100)
	f.Position = props.Position or UDim2.new(0,0,0,0)
	f.AnchorPoint = props.Anchor or Vector2.new(0,0)
	if props.Parent then f.Parent = props.Parent end
	return f
end

local function createLabel(props)
	local l = Instance.new("TextLabel")
	l.BackgroundTransparency = 1
	l.TextColor3 = props.Color or Color3.fromRGB(255,40,40)
	l.Font = Enum.Font.Code
	l.TextSize = props.Size or 14
	l.Text = props.Text or ""
	l.TextXAlignment = props.AlignX or Enum.TextXAlignment.Left
	l.Size = props.FrameSize or UDim2.new(0,200,0,20)
	l.Position = props.Position or UDim2.new(0,0,0,0)
	l.AnchorPoint = props.Anchor or Vector2.new(0,0)
	if props.Parent then l.Parent = props.Parent end
	return l
end

-- Overlay escuro (para cutscene)
local darkOverlay = createFrame{
	Size = UDim2.new(1,0,1,0), Color = Color3.new(0,0,0),
	Transparency = 1, Parent = screenGui,
}
local darkLabel = Instance.new("ImageLabel", darkOverlay)
darkLabel.BackgroundTransparency = 1
darkLabel.Size = UDim2.new(1,0,1,0)
darkLabel.Image = ""
darkLabel.Visible = false

-- Scanlines
local scanlines = createFrame{
	Size = UDim2.new(1,0,1,0), Color = Color3.new(0,0,0),
	Transparency = 0.7, Parent = screenGui,
}
scanlines.Visible = false

-- Status labels (HUD canto superior)
local statusContainer = createFrame{
	Size = UDim2.new(0, 280, 0, 120),
	Position = UDim2.new(0, 20, 0, 60),
	Transparency = 1, Parent = screenGui,
}
statusContainer.Visible = false

local hudTitle = createLabel{
	Text = "◈ PRYPOT 0.X1 // ONLINE",
	Color = Color3.fromRGB(255,40,40), Size = 16,
	FrameSize = UDim2.new(1,0,0,20), Position = UDim2.new(0,0,0,0),
	Parent = statusContainer,
}
local hudStatus = createLabel{
	Text = "NEURAL LINK: ESTABLISHED",
	Color = Color3.fromRGB(0,220,255), Size = 12,
	FrameSize = UDim2.new(1,0,0,16), Position = UDim2.new(0,0,0,24),
	Parent = statusContainer,
}
local hudPower = createLabel{
	Text = "POWER: ████████ 100%",
	Color = Color3.fromRGB(180,180,180), Size = 12,
	FrameSize = UDim2.new(1,0,0,16), Position = UDim2.new(0,0,0,44),
	Parent = statusContainer,
}
local hudWarn = createLabel{
	Text = "",
	Color = Color3.fromRGB(255,160,30), Size = 13,
	FrameSize = UDim2.new(1,0,0,20), Position = UDim2.new(0,0,0,68),
	Parent = statusContainer,
}

-- Crosshair
local crosshair = createFrame{
	Size = UDim2.new(0, 24, 0, 24),
	Anchor = Vector2.new(0.5, 0.5),
	Position = UDim2.new(0.5, 0, 0.5, 0),
	Transparency = 1, Parent = screenGui,
}
crosshair.Visible = false
local ch1 = createFrame{Size = UDim2.new(0,2,0,24), Position = UDim2.new(0.5,-1,0,0), Color = Color3.fromRGB(255,40,40), Parent = crosshair}
local ch2 = createFrame{Size = UDim2.new(0,24,0,2), Position = UDim2.new(0,0,0.5,-1), Color = Color3.fromRGB(255,40,40), Parent = crosshair}
local chDot = createFrame{Size = UDim2.new(0,4,0,4), Position = UDim2.new(0.5,-2,0.5,-2), Color = Color3.fromRGB(255,40,40), Parent = crosshair}

-- Proximity hint
local proxHint = createFrame{
	Size = UDim2.new(0, 260, 0, 40),
	Anchor = Vector2.new(0.5, 0),
	Position = UDim2.new(0.5, 0, 0.7, 0),
	Color = Color3.fromRGB(0,0,0), Transparency = 0.3,
	Parent = screenGui,
}
proxHint.Visible = false
local proxLabel = createLabel{
	Text = "[ E ] PILOTAR PRYPOT 0.X1",
	Color = Color3.fromRGB(255,40,40), Size = 14,
	FrameSize = UDim2.new(1,0,1,0),
	AlignX = Enum.TextXAlignment.Center,
	Parent = proxHint,
}

--------------------------------------------------------------------------------
-- UI: BOTÕES MOBILE
--------------------------------------------------------------------------------
local mobileContainer = createFrame{
	Size = UDim2.new(0, 200, 0, 200),
	Anchor = Vector2.new(1, 1),
	Position = UDim2.new(1, -20, 1, -20),
	Transparency = 1, Parent = screenGui,
}
mobileContainer.Visible = false

local function createMobileButton(name, text, angleDeg, color)
	local radius = 70
	local rad = math.rad(angleDeg)
	local x = math.cos(rad) * radius
	local y = math.sin(rad) * radius
	
	local btn = Instance.new("TextButton")
	btn.Name = name
	btn.Size = UDim2.new(0, 56, 0, 56)
	btn.AnchorPoint = Vector2.new(0.5, 0.5)
	btn.Position = UDim2.new(0.5, x, 0.5, y)
	btn.BackgroundColor3 = Color3.fromRGB(0,0,0)
	btn.BackgroundTransparency = 0.3
	btn.Text = text
	btn.TextColor3 = color or Color3.fromRGB(255,40,40)
	btn.Font = Enum.Font.GothamBold
	btn.TextSize = 10
	btn.Parent = mobileContainer
	
	local corner = Instance.new("UICorner", btn)
	corner.CornerRadius = UDim.new(1, 0)
	
	local stroke = Instance.new("UIStroke", btn)
	stroke.Color = color or Color3.fromRGB(255,40,40)
	stroke.Thickness = 2
	
	-- Cooldown ring (invisible initially)
	local cd = Instance.new("Frame", btn)
	cd.Name = "Cooldown"
	cd.Size = UDim2.new(1,0,1,0)
	cd.BackgroundColor3 = Color3.new(0,0,0)
	cd.BackgroundTransparency = 0.5
	cd.Visible = false
	local cdCorner = Instance.new("UICorner", cd)
	cdCorner.CornerRadius = UDim.new(1, 0)
	
	return btn
end

local btnDig     = createMobileButton("Dig",     "DIG",     -90,  Color3.fromRGB(255,40,40))
local btnStealth = createMobileButton("Stealth", "STEALTH", -45,  Color3.fromRGB(0,220,255))
local btnPunch   = createMobileButton("Punch",   "PUNCH",    0,   Color3.fromRGB(255,160,30))
local btnDash    = createMobileButton("Dash",    "DASH",    45,   Color3.fromRGB(100,255,100))
local btnExit    = createMobileButton("Exit",    "EXIT",    90,   Color3.fromRGB(180,180,180))

--------------------------------------------------------------------------------
-- SONS PROCEDURAIS (sem assets externos)
--------------------------------------------------------------------------------
local function playSound(id, parent, volume, pitch)
	local s = Instance.new("Sound")
	s.SoundId = id
	s.Volume = volume or 1
	s.PlaybackSpeed = pitch or 1
	s.Parent = parent or Workspace
	s:Play()
	game:GetService("Debris"):AddItem(s, 5)
	return s
end

local SFX = {
	metal   = "rbxassetid://131967102",
	alarm   = "rbxassetid://131967102",
	drill   = "rbxassetid://131967102",
	impact  = "rbxassetid://131967102",
	whoosh  = "rbxassetid://131967102",
	-- Em produção, substituir por IDs próprios
}

--------------------------------------------------------------------------------
-- EFEITOS VISUAIS
--------------------------------------------------------------------------------
local function shakeCamera(intensity, duration)
	local startTime = tick()
	local conn
	conn = RunService.RenderStepped:Connect(function()
		local t = tick() - startTime
		if t >= duration then
			conn:Disconnect()
			return
		end
		local factor = 1 - t / duration
		local offset = Vector3.new(
			(math.random() - 0.5) * 2 * intensity * factor,
			(math.random() - 0.5) * 2 * intensity * factor,
			(math.random() - 0.5) * 2 * intensity * factor
		)
		camera.CFrame = camera.CFrame * CFrame.new(offset)
	end)
end

local function flashScreen(color, duration)
	local f = createFrame{
		Size = UDim2.new(1,0,1,0),
		Color = color, Transparency = 0,
		Parent = screenGui,
	}
	TweenService:Create(f, TweenInfo.new(duration, Enum.EasingStyle.Quad), {BackgroundTransparency = 1}):Play()
	game:GetService("Debris"):AddItem(f, duration + 0.1)
end

--------------------------------------------------------------------------------
-- CUTSCENE DE ENTRADA
--------------------------------------------------------------------------------
local function playEntryCutscene(suit)
	local char = player.Character
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	if not hrp then return end
	
	-- Salva câmera original
	local originalCamType = camera.CameraType
	local originalCamSubj = camera.CameraSubject
	
	camera.CameraType = Enum.CameraType.Scriptable
	player.DevEnableMouseLock = false
	
	-- Escurece
	darkOverlay.BackgroundTransparency = 0
	TweenService:Create(darkOverlay, TweenInfo.new(0.6), {BackgroundTransparency = 1}):Play()
	
	local seat = suit:FindFirstChild("PilotSeat")
	local startCFrame = suit.PrimaryPart.CFrame * CFrame.new(6, 2, 6)
	local endCFrame   = suit.PrimaryPart.CFrame * CFrame.new(2, 1, 3)
	
	local camStart = tick()
	local camDuration = 2.5
	
	-- Teleporta o char pra perto da armadura silenciosamente
	hrp.CFrame = suit.PrimaryPart.CFrame * CFrame.new(0, 1.5, 0)
	
	local camConn
	camConn = RunService.RenderStepped:Connect(function()
		local t = math.clamp((tick() - camStart) / camDuration, 0, 1)
		local cf = startCFrame:Lerp(endCFrame, t)
		camera.CFrame = CFrame.new(cf.Position, suit.PrimaryPart.Position)
	end)
	
	task.wait(1.2)
	
	-- Flash vermelho
	flashScreen(Color3.fromRGB(255, 0, 0), 0.3)
	playSound(SFX.metal, hrp, 2, 0.7)
	
	-- Animação dos espinhos neurais (estendem em direção ao char)
	local spikesFolder = {}
	for _, desc in ipairs(suit:GetDescendants()) do
		if desc.Name == "NeuralSpike" and desc:IsA("BasePart") then
			table.insert(spikesFolder, desc)
		end
	end
	
	for i, spike in ipairs(spikesFolder) do
		task.delay(i * 0.08, function()
			local baseCf = spike:GetAttribute("BaseCFrame")
			if baseCf then
				local target = spike.CFrame * CFrame.new(0, 0, 0.8) -- avança em direção ao char
				TweenService:Create(spike, TweenInfo.new(0.2, Enum.EasingStyle.Back), {CFrame = target}):Play()
				task.wait(0.3)
				TweenService:Create(spike, TweenInfo.new(0.5), {CFrame = baseCf}):Play()
				-- Solda após cutscene
				local weld = Instance.new("WeldConstraint")
				weld.Part0 = spike
				weld.Part1 = suit.PrimaryPart
				weld.Parent = spike
			end
		end)
	end
	
	task.wait(1.5)
	
	-- Fade HUD in
	statusContainer.Visible = true
	statusContainer.BackgroundTransparency = 1
	for i = 1, 10 do
		statusContainer.BackgroundTransparency = 1 - (i/10)
		task.wait(0.02)
	end
	
	camConn:Disconnect()
	
	-- Restaura câmera para o humanoid (agora sentado)
	local hum = char:FindFirstChildOfClass("Humanoid")
	camera.CameraType = Enum.CameraType.Custom
	camera.CameraSubject = hum
	
	-- Fade out overlay
	TweenService:Create(darkOverlay, TweenInfo.new(0.8), {BackgroundTransparency = 1}):Play()
end

--------------------------------------------------------------------------------
-- SISTEMA DE ENTRADA / SAÍDA
--------------------------------------------------------------------------------
local function enterSuit(suit)
	if state.inSuit then return end
	
	state.inSuit = true
	state.suit = suit
	
	-- Solicita ao server
	Remotes.RequestEnter:FireServer()
end

Remotes.EnterConfirmed:Connect(function(fullName)
	local suit = Workspace:FindFirstChild(CONFIG.ModelName)
	if not suit then return end
	
	state.suit = suit
	state.inSuit = true
	
	-- Guarda valores originais do char
	local char = player.Character
	if char then
		local hum = char:FindFirstChildOfClass("Humanoid")
		if hum then
			state.originalSpeed = hum.WalkSpeed
			state.originalJump = hum.JumpPower
			hum.WalkSpeed = CONFIG.WalkSpeed
			hum.JumpPower = CONFIG.JumpPower
		end
	end
	
	-- Cutscene
	playEntryCutscene(suit)
	
	-- Mostra HUD
	scanlines.Visible = true
	crosshair.Visible = true
	statusContainer.Visible = true
	if UserInputService.TouchEnabled then
		mobileContainer.Visible = true
	end
	
	-- Inicia loops de animação
	startAnimationLoops(suit)
end)

local function exitSuit()
	if not state.inSuit then return end
	Remotes.RequestExit:FireServer()
end

Remotes.ExitConfirmed:Connect(function()
	state.inSuit = false
	
	-- Restaura char
	local char = player.Character
	if char then
		local hum = char:FindFirstChildOfClass("Humanoid")
		if hum then
			hum.WalkSpeed = state.originalSpeed
			hum.JumpPower = state.originalJump
		end
	end
	
	-- Esconde HUD
	scanlines.Visible = false
	crosshair.Visible = false
	statusContainer.Visible = false
	mobileContainer.Visible = false
	
	-- Para loops
	stopAnimationLoops()
	
	-- Flash de saída
	flashScreen(Color3.fromRGB(0,0,0), 0.4)
end)

--------------------------------------------------------------------------------
-- LOOPS DE ANIMAÇÃO (engrenagens giram, mandíbula mexe, respiração)
--------------------------------------------------------------------------------
local animationConnections = {}

function startAnimationLoops(suit)
	-- Engrenagens
	for _, desc in ipairs(suit:GetDescendants()) do
		if desc.Name == "Gear" and desc:IsA("BasePart") then
			local speed = desc:GetAttribute("SpinSpeed") or 1
			local conn = RunService.Heartbeat:Connect(function(dt)
				-- Girar via CFrame relativo ao pai (weld)
				-- Como estão soldadas, precisamos de Motor6D ou manipular C0
				-- Para simplicidade: giramos a própria part se não soldada estritamente
				-- (elas estão soldadas, então o visual não gira por agora - seria
				-- necessário Motor6D no lugar do Weld. Mantemos como fallback.)
			end)
			table.insert(animationConnections, conn)
		end
	end
	
	-- Respiração / mandíbula
	local jawMotor = suit:FindFirstChild("JawMotor", true)
	local breathConn = RunService.Heartbeat:Connect(function()
		if jawMotor then
			local t = tick()
			-- mandíbula faz micro-movimentos ocasionais
			local jawAngle = math.sin(t * 0.8) * 0.05
			if math.random() < 0.002 then
				jawAngle = math.sin(t * 8) * 0.2 -- snap ocasional
			end
			jawMotor.C0 = CFrame.new(0, 0.2, 0) * CFrame.Angles(jawAngle, 0, 0)
		end
	end)
	table.insert(animationConnections, breathConn)
	
	-- Micro-movimento da cabeça (animatrônico perturbador)
	local neckMotor = suit:FindFirstChild("NeckMotor", true)
	local headConn = RunService.Heartbeat:Connect(function()
		if neckMotor then
			local t = tick()
			local yaw   = math.sin(t * 0.3) * 0.1 + (math.random() < 0.003 and math.rad(math.random(-30,30)) or 0)
			local pitch = math.sin(t * 0.2) * 0.05
			neckMotor.C0 = CFrame.new(0, -0.8, 0) * CFrame.Angles(pitch, yaw, 0)
		end
	end)
	table.insert(animationConnections, headConn)
	
	-- Pulso do núcleo
	local core = suit:FindFirstChild("Core", true)
	if core then
		local light = core:FindFirstChildOfClass("PointLight")
		if light then
			local pulseConn = RunService.Heartbeat:Connect(function()
				local t = tick()
				light.Brightness = 2 + math.sin(t * 3) * 1.2
			end)
			table.insert(animationConnections, pulseConn)
		end
	end
end

function stopAnimationLoops()
	for _, c in ipairs(animationConnections) do
		pcall(function() c:Disconnect() end)
	end
	animationConnections = {}
end

--------------------------------------------------------------------------------
-- SISTEMA DE ESCAVAÇÃO (client-side: detecta input e dispara remote)
--------------------------------------------------------------------------------
local function tryDig()
	if not state.inSuit then return end
	if tick() - state.lastDig < CONFIG.DigCooldown then return end
	state.lastDig = tick()
	
	local suit = state.suit
	if not suit then return end
	
	local pos = suit.PrimaryPart.Position
	-- Exclui a própria armadura e o char
	local char = player.Character
	local exclude = {suit}
	if char then table.insert(exclude, char) end
	
	Remotes.DigRequest:FireServer(pos, exclude)
end

Remotes.DigConfirmed:Connect(function(position)
	-- VFX local de escavação
	flashScreen(Color3.fromRGB(120,80,40), 0.15)
	shakeCamera(0.4, 0.4)
	
	-- Partículas de terra
	local suit = state.suit
	if suit then
		local debris = Instance.new("Part")
		debris.Size = Vector3.new(0.1,0.1,0.1)
		debris.Anchored = true
		debris.CanCollide = false
		debris.Transparency = 1
		debris.CFrame = CFrame.new(position)
		debris.Parent = Workspace
		
		local emitter = Instance.new("ParticleEmitter")
		emitter.Texture = "rbxasset://textures/particles/smoke_main.dds"
		emitter.Color = ColorSequence.new(Color3.fromRGB(90,60,30))
		emitter.Size = NumberSequence.new(1, 0)
		emitter.Lifetime = NumberRange.new(0.6, 1)
		emitter.Speed = NumberRange.new(8, 14)
		emitter.SpreadAngle = Vector2.new(60, 60)
		emitter.Rate = 0
		emitter.Parent = debris
		
		emitter:Emit(40)
		game:GetService("Debris"):AddItem(debris, 2)
		
		playSound(SFX.drill, debris, 1.5, 0.8)
	end
	
	-- Animação do drill (braço direito girando rápido)
	local drill = suit and suit:FindFirstChild("DrillBit", true)
	if drill then
		local start = tick()
		local conn
		conn = RunService.Heartbeat:Connect(function(dt)
			if tick() - start > 0.6 then conn:Disconnect() return end
			drill.CFrame = drill.CFrame * CFrame.Angles(0, math.rad(720) * dt, 0)
		end)
	end
end)

Remotes.DigUnsafe:Connect(function(fraction)
	-- Aviso no HUD
	hudWarn.Text = string.format("⚠ TERRENO INSEGURO (void %d%%)", math.floor((1 - fraction) * 100))
	TweenService:Create(hudWarn, TweenInfo.new(0.1), {TextColor3 = Color3.fromRGB(255,40,40)}):Play()
	task.delay(2.5, function()
		hudWarn.Text = ""
	end)
end)

--------------------------------------------------------------------------------
-- SISTEMA DE SOCO (apenas visual local)
--------------------------------------------------------------------------------
local function punch()
	if not state.inSuit then return end
	
	local char = player.Character
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	if not hrp then return end
	
	local suit = state.suit
	local hand = suit and suit:FindFirstChild("HandR", true)
	if not hand then return end
	
	-- Animação do braço
	local startCf = hand.CFrame
	local targetCf = hand.CFrame * CFrame.new(0, 0, -2)
	TweenService:Create(hand, TweenInfo.new(0.1, Enum.EasingStyle.Back), {CFrame = targetCf}):Play()
	task.delay(0.15, function()
		TweenService:Create(hand, TweenInfo.new(0.25), {CFrame = startCf}):Play()
	end)
	
	-- Onda de choque
	local origin = hand.Position + hand.CFrame.LookVector * 2
	local wave = Instance.new("Part")
	wave.Shape = Enum.PartType.Ball
	wave.Size = Vector3.new(1,1,1)
	wave.Anchored = true
	wave.CanCollide = false
	wave.Material = Enum.Material.Neon
	wave.Color = Color3.fromRGB(255,40,40)
	wave.Transparency = 0.3
	wave.CFrame = CFrame.new(origin)
	wave.Parent = Workspace
	
	TweenService:Create(wave, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		Size = Vector3.new(CONFIG.PunchRange * 2, CONFIG.PunchRange * 2, CONFIG.PunchRange * 2),
		Transparency = 1,
	}):Play()
	game:GetService("Debris"):AddItem(wave, 0.6)
	
	shakeCamera(0.6, 0.3)
	flashScreen(Color3.fromRGB(255, 60, 0), 0.15)
	playSound(SFX.impact, hrp, 2, 0.9)
end

--------------------------------------------------------------------------------
-- SISTEMA DE DASH
--------------------------------------------------------------------------------
local function dash()
	if not state.inSuit then return end
	if state.dashing then return end
	if tick() - state.lastDash < 1 then return end
	state.lastDash = tick()
	state.dashing = true
	
	local char = player.Character
	if not char then state.dashing = false return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	if not hrp then state.dashing = false return end
	
	-- Direção: camera lookVector (horizontal)
	local dir = camera.CFrame.LookVector
	dir = Vector3.new(dir.X, 0, dir.Z).Unit
	
	local bv = Instance.new("BodyVelocity")
	bv.MaxForce = Vector3.new(math.huge, 0, math.huge)
	bv.Velocity = dir * CONFIG.ParkourDashSpeed
	bv.Parent = hrp
	
	-- Partículas de propulsor nos pés
	local suit = state.suit
	if suit then
		for _, desc in ipairs(suit:GetDescendants()) do
			if desc.Name == "Thruster" and desc:IsA("ParticleEmitter") then
				desc.Rate = 200
			end
		end
	end
	
	flashScreen(Color3.fromRGB(0, 220, 255), 0.2)
	playSound(SFX.whoosh, hrp, 1.5, 1.3)
	
	task.delay(CONFIG.ParkourDashTime, function()
		bv:Destroy()
		state.dashing = false
		if suit then
			for _, desc in ipairs(suit:GetDescendants()) do
				if desc.Name == "Thruster" and desc:IsA("ParticleEmitter") then
					desc.Rate = 0
				end
			end
		end
	end)
end

--------------------------------------------------------------------------------
-- SISTEMA DE STEALTH
--------------------------------------------------------------------------------
local function toggleStealth()
	if not state.inSuit then return end
	if state.stealthActive then return end
	if tick() - state.lastStealth < CONFIG.StealthCooldown then return end
	
	state.stealthActive = true
	state.lastStealth = tick()
	
	local suit = state.suit
	if not suit then state.stealthActive = false return end
	
	-- Fade todas as parts
	for _, desc in ipairs(suit:GetDescendants()) do
		if desc:IsA("BasePart") and desc.Name ~= "PilotSeat" then
			local orig = desc.Transparency
			desc:SetAttribute("OriginalTransparency", orig)
			TweenService:Create(desc, TweenInfo.new(0.4), {Transparency = CONFIG.StealthTransparency}):Play()
			-- Apaga luzes
			for _, light in ipairs(desc:GetChildren()) do
				if light:IsA("Light") then
					light.Enabled = false
				end
			end
			-- Desliga emitters
			for _, em in ipairs(desc:GetChildren()) do
				if em:IsA("ParticleEmitter") then
					em.Enabled = false
				end
			end
		end
	end
	
	hudStatus.Text = "STEALTH: ACTIVE"
	hudStatus.TextColor3 = Color3.fromRGB(0,220,255)
	
	task.delay(CONFIG.StealthDuration, function()
		for _, desc in ipairs(suit:GetDescendants()) do
			if desc:IsA("BasePart") then
				local orig = desc:GetAttribute("OriginalTransparency") or 0
				TweenService:Create(desc, TweenInfo.new(0.6), {Transparency = orig}):Play()
				for _, light in ipairs(desc:GetChildren()) do
					if light:IsA("Light") then light.Enabled = true end
				end
				for _, em in ipairs(desc:GetChildren()) do
					if em:IsA("ParticleEmitter") then em.Enabled = true end
				end
			end
		end
		state.stealthActive = false
		hudStatus.Text = "NEURAL LINK: ESTABLISHED"
		hudStatus.TextColor3 = Color3.fromRGB(0,220,255)
	end)
end

--------------------------------------------------------------------------------
-- SISTEMA DE PARKOUR (wall-run, wall-cling, ceiling-cling, double-jump, impacto)
--------------------------------------------------------------------------------
local function setupParkour()
	local char = player.Character
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	local hum = char:FindFirstChildOfClass("Humanoid")
	if not hrp or not hum then return end
	
	-- Reset ao respawnar
	hum.StateChanged:Connect(function(_, new)
		if new == Enum.HumanoidStateType.Landed then
			state.jumpsUsed = 0
			
			-- Absorção de impacto pelas molas (se caiu de altura)
			local fallHeight = state.highestY - hrp.Position.Y
			if fallHeight > CONFIG.ImpactAbsorbHeight and state.inSuit then
				-- Comprime as molas visualmente
				local suit = state.suit
				if suit then
					for _, desc in ipairs(suit:GetDescendants()) do
						if desc.Name:match("FootMotor") and desc:IsA("Motor6D") then
							local orig = desc.C0
							TweenService:Create(desc, TweenInfo.new(0.1), {
								C0 = orig * CFrame.new(0, 0.4, 0)
							}):Play()
							task.delay(0.25, function()
								TweenService:Create(desc, TweenInfo.new(0.3, Enum.EasingStyle.Bounce), {C0 = orig}):Play()
							end)
						end
					end
					-- Poeira nos pés
					for _, desc in ipairs(suit:GetDescendants()) do
						if desc.Name:match("Foot") and desc:IsA("BasePart") then
							local emitter = desc:FindFirstChildOfClass("ParticleEmitter")
							if emitter then emitter:Emit(20) end
						end
					end
				end
				shakeCamera(0.3, 0.2)
				playSound(SFX.impact, hrp, 1.5, 0.6)
				
				-- Zera dano de queda
				hum.Health = hum.Health + math.min(fallHeight * 2, 50) -- compensação aproximada
			end
			
			state.highestY = hrp.Position.Y
		elseif new == Enum.HumanoidStateType.Freefall then
			state.highestY = math.max(state.highestY, hrp.Position.Y)
		end
	end)
	
	state.highestY = hrp.Position.Y
	
	-- Wall-run / wall-cling detection
	RunService.Heartbeat:Connect(function(dt)
		if not state.inSuit then return end
		if not hrp or not hrp.Parent then return end
		
		-- Raycasts em 4 direções
		local rayLen = 3
		local params = RaycastParams.new()
		params.FilterType = Enum.RaycastFilterType.Exclude
		params.FilterDescendantsInstances = {char, state.suit}
		
		local rightHit = Workspace:Raycast(hrp.Position, hrp.CFrame.RightVector * rayLen, params)
		local leftHit  = Workspace:Raycast(hrp.Position, -hrp.CFrame.RightVector * rayLen, params)
		local frontHit = Workspace:Raycast(hrp.Position, hrp.CFrame.LookVector * rayLen, params)
		local upHit    = Workspace:Raycast(hrp.Position, Vector3.new(0, rayLen, 0), params)
		
		local nearWall = rightHit or leftHit or frontHit
		local nearCeiling = upHit
		
		-- Wall-run: se está no ar, perto de parede e se movendo rápido
		local isAir = hum.FloorMaterial == Enum.Material.Air
		local moveSpeed = hrp.Velocity.Magnitude
		
		if isAir and nearWall and moveSpeed > 20 and hum.Jump then
			-- Wall-run: aplica leve força lateral para manter grudado
			state.wallRunning = true
		elseif isAir and nearWall and UserInputService:IsKeyDown(Enum.KeyCode.Space) then
			-- Wall-cling: segura na parede
			state.wallCling = true
			hrp.Velocity = Vector3.new(hrp.Velocity.X * 0.5, -5, hrp.Velocity.Z * 0.5)
		elseif isAir and nearCeiling and UserInputService:IsKeyDown(Enum.KeyCode.Space) then
			-- Ceiling-cling
			state.ceilingCling = true
			hrp.Velocity = Vector3.new(hrp.Velocity.X * 0.3, 2, hrp.Velocity.Z * 0.3)
		else
			if state.wallCling or state.ceilingCling then
				state.wallCling = false
				state.ceilingCling = false
			end
			state.wallRunning = false
		end
	end)
	
	-- Double jump (detecta segundo pulo)
	UserInputService.JumpRequest:Connect(function()
		if not state.inSuit then return end
		if not hum then return end
		
		if state.jumpsUsed == 1 then
			-- Double jump
			state.jumpsUsed = 2
			hum:ChangeState(Enum.HumanoidStateType.Jumping)
			task.defer(function()
				hrp.Velocity = Vector3.new(hrp.Velocity.X, CONFIG.DoubleJumpPower, hrp.Velocity.Z)
			end)
			
			-- Efeito de propulsor
			local suit = state.suit
			if suit then
				for _, desc in ipairs(suit:GetDescendants()) do
					if desc.Name == "Thruster" and desc:IsA("ParticleEmitter") then
						desc.Rate = 150
						task.delay(0.3, function() desc.Rate = 0 end)
					end
				end
			end
			playSound(SFX.whoosh, hrp, 1, 1.2)
		elseif state.jumpsUsed == 0 then
			state.jumpsUsed = 1
		end
	end)
end

--------------------------------------------------------------------------------
-- PROXIMIDADE (mostra hint quando perto da armadura sem estar dentro)
--------------------------------------------------------------------------------
RunService.Heartbeat:Connect(function()
	if state.inSuit then
		proxHint.Visible = false
		return
	end
	
	local suit = Workspace:FindFirstChild(CONFIG.ModelName)
	if not suit or not suit.PrimaryPart then
		proxHint.Visible = false
		return
	end
	
	local char = player.Character
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	if not hrp then return end
	
	local dist = (hrp.Position - suit.PrimaryPart.Position).Magnitude
	proxHint.Visible = (dist < 12)
end)

--------------------------------------------------------------------------------
-- INPUT HANDLING
--------------------------------------------------------------------------------
UserInputService.InputBegan:Connect(function(input, gp)
	if gp then return end
	
	local suit = Workspace:FindFirstChild(CONFIG.ModelName)
	
	-- Entrar
	if input.KeyCode == Enum.KeyCode.E and not state.inSuit and suit then
		local char = player.Character
		if char then
			local hrp = char:FindFirstChild("HumanoidRootPart")
			if hrp and (hrp.Position - suit.PrimaryPart.Position).Magnitude < 12 then
				enterSuit(suit)
			end
		end
	end
	
	if not state.inSuit then return end
	
	-- Sair
	if input.KeyCode == Enum.KeyCode.X then
		exitSuit()
	end
	
	-- Escavar
	if input.KeyCode == Enum.KeyCode.F then
		tryDig()
	end
	
	-- Dash
	if input.KeyCode == Enum.KeyCode.LeftShift then
		dash()
	end
	
	-- Stealth
	if input.KeyCode == Enum.KeyCode.Q then
		toggleStealth()
	end
	
	-- Pulo (também wall-cling quando segurar)
	if input.KeyCode == Enum.KeyCode.Space then
		-- double-jump tratado em JumpRequest
	end
end)

-- Clique esquerdo = soco
UserInputService.InputBegan:Connect(function(input, gp)
	if gp then return end
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		if state.inSuit then punch() end
	end
end)

-- Botões mobile
btnDig.MouseButton1Click:Connect(tryDig)
btnStealth.MouseButton1Click:Connect(toggleStealth)
btnPunch.MouseButton1Click:Connect(punch)
btnDash.MouseButton1Click:Connect(dash)
btnExit.MouseButton1Click:Connect(exitSuit)

-- Touch feedback
for _, btn in ipairs({btnDig, btnStealth, btnPunch, btnDash, btnExit}) do
	btn.MouseButton1Down:Connect(function()
		TweenService:Create(btn, TweenInfo.new(0.08), {Size = UDim2.new(0,48,0,48)}):Play()
		pcall(function()
			local haptic = game:GetService("HapticService")
			if haptic:IsMotorEnabled(Enum.UserInputType.Touch, Enum.VibrationMotor.Small) then
				haptic:SetMotor(Enum.UserInputType.Touch, Enum.VibrationMotor.Small, 0.6)
				task.delay(0.05, function()
					haptic:SetMotor(Enum.UserInputType.Touch, Enum.VibrationMotor.Small, 0)
				end)
			end
		end)
	end)
	btn.MouseButton1Up:Connect(function()
		TweenService:Create(btn, TweenInfo.new(0.1, Enum.EasingStyle.Back), {Size = UDim2.new(0,56,0,56)}):Play()
	end)
end

--------------------------------------------------------------------------------
-- CONEXÃO COM PERSONAGEM (respawn-safe)
--------------------------------------------------------------------------------
local function onCharacterAdded(char)
	-- Aguarda Humanoid e HRP
	local hum = char:WaitForChild("Humanoid", 5)
	local hrp = char:WaitForChild("HumanoidRootPart", 5)
	if not hum or not hrp then return end
	
	-- Reset state
	state.inSuit = false
	state.suit = nil
	stopAnimationLoops()
	
	-- Reconfigura parkour
	setupParkour()
end

if player.Character then onCharacterAdded(player.Character) end
player.CharacterAdded:Connect(onCharacterAdded)

--------------------------------------------------------------------------------
-- LOOP DE UPDATE DO HUD
--------------------------------------------------------------------------------
RunService.Heartbeat:Connect(function()
	if not state.inSuit then return end
	
	-- Atualiza power bar (baseado em uso de habilidades)
	local cooldownFactor = math.max(0, 1 - (tick() - state.lastDash) / 1)
	local powerBar = string.rep("█", math.floor((1 - cooldownFactor) * 8)) .. string.rep("░", 8 - math.floor((1 - cooldownFactor) * 8))
	hudPower.Text = string.format("POWER: %s %d%%", powerBar, math.floor((1 - cooldownFactor) * 100))
end)

print("[PRYPOT 0.X1] Client ready.")


--------------------------------------------------------------------------------
--                                                                            --
--                              [ END OF SCRIPT ]                             --
--                                                                            --
--  PRYPOT 0.X1 // NEURAL EXOSUIT // CLASSIFIED LVL-5                         --
--  Prypot Industries · Divisão Biomécânica                                   --
--                                                                            --
--------------------------------------------------------------------------------
