-- Script para StarterPlayer > StarterPlayerScripts
-- Sistema de validación API para Lux (inspirado en Gemini AI system)
repeat task.wait() until game:IsLoaded()
local HttpService = game:GetService("HttpService")
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local HttpRequest = (syn and syn.request) or (http and http.request) or http_request or (fluxus and fluxus.request) or request
local LP = Players.LocalPlayer
local playerGui = LP:WaitForChild("PlayerGui")
-- Configuración
local KEY_FILE_NAME = "Lux_API_Key.txt"
local API_KEY = ""
local API_TIMEOUT_SECONDS = 15
local API_PROVIDER = "gemini" -- "gemini" o "groq"
local GROQ_MODEL_URL = "https://api.groq.com/openai/v1/chat/completions"
local GROQ_MODEL = "llama-3.3-70b-versatile" -- Modelo por defecto de Groq
local GEMINI_MODEL_URL = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent"
-- ============================================
-- SISTEMA DE EJECUCIÓN DE SCRIPTS
-- ============================================
-- Scripts disponibles con sus keywords
local AVAILABLE_SCRIPTS = {
	{
		keywords = {"grip", "gripscript"},
		url = "https://pastebin.com/raw/cCRTZzbP",
		name = "GRIP Script",
		description = "Script de agarre mejorado"
	},
	{
		keywords = {"proteger", "no daños", "evitar daños", "proteger jugador", "god", "godmode"},
		url = "https://pastebin.com/raw/3QRfTGkG",
		name = "God Mode",
		description = "Protección contra daños"
	},
	{
		keywords = {"vuelo", "volar", "permitir vuelo", "permitir volar", "levitar", "fly"},
		url = "https://pastebin.com/raw/GjCJ5e8u",
		name = "Fly Script",
		description = "Script de vuelo"
	},
	{
		keywords = {"inf yield", "yield", "infinite", "infinite yield", "iy"},
		url = "https://raw.githubusercontent.com/EdgeIY/infiniteyield/master/source",
		name = "Infinite Yield",
		description = "Admin commands"
	},
	{
		keywords = {"nameless", "less", "namelessscript", "nameless admin"},
		url = "https://rawscripts.net/raw/Universal-Script-Nameless-admin-REWORKED-43502",
		name = "Nameless Admin",
		description = "Admin script"
	}
}
-- Función para ejecutar código de forma segura
local function SafeExecuteScript(url)
	if not url or url == "" then
		return false, "URL inválida"
	end
	-- Verificar que tengamos las funciones necesarias
	if not game.HttpGet or not loadstring then
		return false, "Funciones de ejecución no disponibles"
	end
	-- Intentar ejecutar
	local success, result = pcall(function()
		local scriptContent = game:HttpGet(url, true)
		if not scriptContent or scriptContent == "" then
			error("No se pudo descargar el script")
		end
		-- Ejecutar con loadstring (más seguro que directamente)
		local executeFunc = loadstring(scriptContent)
		if executeFunc then
			executeFunc()
			return true
		else
			error("Error al compilar el script")
		end
	end)
	if success then
		return true, "Script ejecutado correctamente"
	else
		return false, "Error al ejecutar: " .. tostring(result)
	end
end
-- Función para buscar script por keyword
local function FindScriptByKeyword(userInput)
	userInput = userInput:lower():gsub("%s+", " "):gsub("^%s+", ""):gsub("%s+$", "")
	-- Buscar coincidencia exacta
	for _, script in ipairs(AVAILABLE_SCRIPTS) do
		for _, keyword in ipairs(script.keywords) do
			if userInput == keyword:lower() then
				return script, nil
			end
		end
	end
	-- Buscar coincidencia parcial (para sugerir correcciones)
	local suggestions = {}
	for _, script in ipairs(AVAILABLE_SCRIPTS) do
		for _, keyword in ipairs(script.keywords) do
			if keyword:lower():find(userInput, 1, true) or userInput:find(keyword:lower(), 1, true) then
				table.insert(suggestions, {script = script, keyword = keyword})
			end
		end
	end
	-- Si hay sugerencias, devolverlas
	if #suggestions > 0 then
		return nil, suggestions
	end
	return nil, nil
end
-- Estados del sistema
local Settings = {
	AI_Enabled = false,
	IsThinking = false,
	IsServerBusy = false,
	IsProbing = false,
	NextReqTime = 0
}
local ThinkingStartTime = 0
local BusyStartTime = 0
-- Variables UI
local ScreenGui, AuthFrame, MainFrame, StatusIndicator, APIKeyBox, ValidateButton, ToggleButton, ConsoleLabel, ShowButton
local VALIDATION_FILE_NAME = "Lux_Validation.txt"
-- ⬇️ PEGA AQUÍ:
local PRIVATE_MODE = true -- true = solo el dueño puede hablar
local scriptOwner = LP -- El jugador que ejecuta el script
-- ============================================
-- SISTEMA DE PERCEPCIÓN DE ENTORNO
-- ============================================
local EnvironmentData = {
	LastUpdate = 0,
	UpdateInterval = 2, -- Actualizar cada 2 segundos (no causa lag)
	CachedData = nil,
	PlayersData = {},
	OwnerPosition = Vector3.new(0, 0, 0)
}
-- Función para obtener distancia en studs entre dos posiciones
local function GetDistance(pos1, pos2)
	return math.floor((pos1 - pos2).Magnitude)
end
-- Función para obtener datos de un jugador específico
local function GetPlayerData(player)
	if not player or not player.Character then
		return nil
	end
	local char = player.Character
	local humanoid = char:FindFirstChild("Humanoid")
	local rootPart = char:FindFirstChild("HumanoidRootPart")
	if not rootPart then
		return nil
	end
	local ownerRoot = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
	local distanceFromOwner = ownerRoot and GetDistance(rootPart.Position, ownerRoot.Position) or 9999
	return {
		name = player.DisplayName or player.Name,
		username = player.Name,
		userId = player.UserId,
		position = rootPart.Position,
		health = humanoid and math.floor(humanoid.Health) or 0,
		maxHealth = humanoid and math.floor(humanoid.MaxHealth) or 100,
		distanceFromYou = distanceFromOwner,
		isAlive = humanoid and humanoid.Health > 0 or false,
		team = player.Team and player.Team.Name or "No team"
	}
end
-- Función para escanear el entorno completo
local function ScanEnvironment()
	local currentTime = tick()
	-- Solo actualizar si ha pasado el intervalo
	if currentTime - EnvironmentData.LastUpdate < EnvironmentData.UpdateInterval then
		return EnvironmentData.CachedData
	end
	-- Actualizar timestamp
	EnvironmentData.LastUpdate = currentTime
	-- Obtener datos del dueño (LocalPlayer)
	local ownerChar = LP.Character
	local ownerRoot = ownerChar and ownerChar:FindFirstChild("HumanoidRootPart")
	local ownerHumanoid = ownerChar and ownerChar:FindFirstChild("Humanoid")
	if ownerRoot then
		EnvironmentData.OwnerPosition = ownerRoot.Position
	end
	-- Datos del dueño
	local ownerData = {
		health = ownerHumanoid and math.floor(ownerHumanoid.Health) or 0,
		maxHealth = ownerHumanoid and math.floor(ownerHumanoid.MaxHealth) or 100,
		position = ownerRoot and ownerRoot.Position or Vector3.new(0, 0, 0),
		isAlive = ownerHumanoid and ownerHumanoid.Health > 0 or false
	}
	-- Datos de otros jugadores
	local playersData = {}
	local playerCount = 0
	local playersNearby = 0 -- Jugadores a menos de 50 studs
	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= LP then
			local data = GetPlayerData(player)
			if data then
				table.insert(playersData, data)
				playerCount = playerCount + 1
				if data.distanceFromYou < 50 then
					playersNearby = playersNearby + 1
				end
			end
		end
	end
	-- Ordenar jugadores por distancia (más cercano primero)
	table.sort(playersData, function(a, b)
		return a.distanceFromYou < b.distanceFromYou
	end)
	-- Guardar solo los 5 jugadores más cercanos (optimización)
	local nearestPlayers = {}
	for i = 1, math.min(5, #playersData) do
		table.insert(nearestPlayers, playersData[i])
	end
	-- Construir datos del entorno
	local environmentInfo = {
		owner = ownerData,
		totalPlayers = playerCount,
		playersNearby = playersNearby,
		nearestPlayers = nearestPlayers,
		gameName = game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId).Name or "Unknown",
		timestamp = os.date("%H:%M:%S")
	}
	-- Cachear datos
	EnvironmentData.CachedData = environmentInfo
	EnvironmentData.PlayersData = playersData
	return environmentInfo
end
-- Función para obtener información de un jugador por nombre (búsqueda inteligente)
local function FindPlayerByName(searchName)
	if not searchName or searchName == "" then
		return nil
	end
	searchName = searchName:lower()
	-- Actualizar datos primero
	ScanEnvironment()
	-- Buscar en todos los jugadores
	for _, playerData in ipairs(EnvironmentData.PlayersData) do
		local name = playerData.name:lower()
		local username = playerData.username:lower()
		-- Coincidencia exacta
		if name == searchName or username == searchName then
			return playerData
		end
		-- Coincidencia parcial
		if name:find(searchName, 1, true) or username:find(searchName, 1, true) then
			return playerData
		end
	end
	return nil
end
-- Función para generar resumen del entorno (texto corto para Lux)
local function GetEnvironmentSummary()
	local env = ScanEnvironment()
	if not env then
		return "No puedo ver el entorno en este momento."
	end
	local summary = string.format(
		"Tu salud: %d/%d | Jugadores: %d total, %d cercanos",
		env.owner.health,
		env.owner.maxHealth,
		env.totalPlayers,
		env.playersNearby
	)
	-- Agregar jugadores más cercanos
	if #env.nearestPlayers > 0 then
		summary = summary .. " | Más cerca: "
		local names = {}
		for i = 1, math.min(3, #env.nearestPlayers) do
			local p = env.nearestPlayers[i]
			table.insert(names, string.format("%s (%d studs)", p.name, p.distanceFromYou))
		end
		summary = summary .. table.concat(names, ", ")
	end
	return summary
end
-- Función para formatear datos de entorno para Lux (JSON compacto)
local function FormatEnvironmentForLux()
	local env = ScanEnvironment()
	if not env then
		return "Environment data unavailable"
	end
	-- Crear versión compacta para el contexto de Lux
	local compactData = {
		your_health = env.owner.health .. "/" .. env.owner.maxHealth,
		alive = env.owner.isAlive,
		total_players = env.totalPlayers,
		players_nearby = env.playersNearby,
		game = env.gameName
	}
	-- Agregar jugadores cercanos (solo los 3 más cercanos)
	if #env.nearestPlayers > 0 then
		compactData.nearest_players = {}
		for i = 1, math.min(3, #env.nearestPlayers) do
			local p = env.nearestPlayers[i]
			table.insert(compactData.nearest_players, {
				name = p.name,
				distance = p.distanceFromYou .. " studs",
				health = p.health .. "/" .. p.maxHealth,
				alive = p.isAlive
			})
		end
	end
	-- Convertir a JSON (será agregado al contexto de Lux)
	return HttpService:JSONEncode(compactData)
end
-- Función para logs en consola
local function UpdateConsole(text, status)
	if not ScreenGui or not ScreenGui.Parent then return end
	local displayText = text or ""
	if status then displayText = displayText .. " | " .. status end
	if #displayText > 50 then displayText = displayText:sub(1, 47) .. "..." end
	ConsoleLabel.Text = displayText
end
-- Función para guardar API Key
local function SaveAPIKey(key)
	if writefile then
		pcall(function()
			writefile(KEY_FILE_NAME, key)
		end)
	end
end
-- Función para cargar API Key guardada
local function LoadAPIKey()
	if isfile and isfile(KEY_FILE_NAME) then
		local success, key = pcall(function()
			return readfile(KEY_FILE_NAME):gsub("%s+", "")
		end)
		if success and #key > 10 then
			return key
		end
	end
	return nil
end
-- Función para guardar estado de validación
local function SaveValidation(apiKey, provider)
	if writefile then
		pcall(function()
			local validationData = {
				apiKey = apiKey,
				provider = provider or "gemini", -- NUEVO
				validated = true,
				timestamp = os.time()
			}
			writefile(VALIDATION_FILE_NAME, HttpService:JSONEncode(validationData))
		end)
	end
end
-- Función para cargar validación guardada
local function LoadValidation()
	if isfile and isfile(VALIDATION_FILE_NAME) then
		local success, data = pcall(function()
			local content = readfile(VALIDATION_FILE_NAME)
			return HttpService:JSONDecode(content)
		end)
		if success and data and data.validated and data.apiKey and #data.apiKey > 10 then
			return data.apiKey, data.provider or "gemini" -- MODIFICADO
		end
	end
	return nil, nil
end
-- Función para limpiar validación
local function WipeValidation()
	if isfile and isfile(VALIDATION_FILE_NAME) then 
		delfile(VALIDATION_FILE_NAME) 
	end
	WipeKeyData()
end
-- Función para limpiar API Key
local function WipeKeyData()
	if isfile and isfile(KEY_FILE_NAME) then 
		delfile(KEY_FILE_NAME) 
	end
	API_KEY = ""
end
-- Función de detección de estado del servidor (probe)
local function ProbeServer()
	if Settings.IsProbing then return end
	Settings.IsProbing = true
	UpdateConsole("Rate Limit Hit", "Cooling Down")
	task.wait(5)
	local backoff = 5
	while Settings.IsServerBusy and ScreenGui.Parent do
		task.wait(backoff)
		UpdateConsole("Probing Server", "Backoff: " .. backoff .. "s")
		-- Petición mínima de prueba
		local probeBody = HttpService:JSONEncode({
			contents = {{role = "user", parts = {{text = "."}}}},
			generationConfig = { maxOutputTokens = 1 }
		})
		local success, res = pcall(function()
			return HttpRequest({
				Url = GEMINI_MODEL_URL .. "?key=" .. API_KEY,
				Method = "POST",
				Headers = { ["Content-Type"] = "application/json" },
				Body = probeBody
			})
		end)
		local recovered = false
		if success and res then
			if res.StatusCode and res.StatusCode ~= 429 and res.StatusCode ~= 503 and res.StatusCode ~= 500 then
				recovered = true
			elseif res.Body then
				if not res.Body:find("429") and not res.Body:find("503") then 
					recovered = true 
				end
			end
		end
		if recovered then
			Settings.IsServerBusy = false
			Settings.IsProbing = false
			UpdateConsole("Server Recovered", "Online")
			Settings.NextReqTime = tick() + 2
			BusyStartTime = 0
		else
			backoff = math.min(backoff + 5, 30)
			UpdateConsole("Still Busy", "Wait " .. backoff .. "s")
		end
		-- Timeout de probe
		if Settings.IsServerBusy and (tick() - BusyStartTime) > 90 then
			Settings.IsServerBusy = false
			Settings.IsProbing = false
			UpdateConsole("Probe Timeout", "Forced Reset")
			BusyStartTime = 0
			break
		end
	end
	Settings.IsProbing = false
end
-- Función principal de validación de API Key
local function ValidateAPIKey(apiKey, bypassSettings)
	if not bypassSettings and tick() < Settings.NextReqTime then 
		return nil, "COOLDOWN" 
	end
	
	if apiKey == "" or #apiKey < 10 then
		return false, "API Key muy corta o vacía"
	end
	
	if Settings.IsThinking then 
		return nil, "THINKING" 
	end
	
	Settings.IsThinking = true
	ThinkingStartTime = tick()
	
	if Settings.IsServerBusy then
		if (tick() - BusyStartTime) > 60 then
			Settings.IsServerBusy = false
			UpdateConsole("Reset Busy State", "Retry")
		else
			UpdateConsole("Server Busy", "Probing...")
			Settings.IsThinking = false
			if not Settings.IsProbing then 
				task.spawn(ProbeServer) 
			end
			return nil, "BUSY"
		end
	end
	
	UpdateConsole("Validating API", "Detecting provider...")
	
	-- NUEVO: Detectar automáticamente si es Gemini o Groq
	local detectedProvider = "gemini"
	if apiKey:match("^gsk_") then
		detectedProvider = "groq"
	elseif apiKey:match("^AI") then
		detectedProvider = "gemini"
	end
	
	local url, body, headers
	
	if detectedProvider == "groq" then
		-- Request de prueba para Groq
		url = GROQ_MODEL_URL
		headers = {
			["Content-Type"] = "application/json",
			["Authorization"] = "Bearer " .. apiKey
		}
		body = HttpService:JSONEncode({
			model = GROQ_MODEL,
			messages = {{role = "user", content = "test"}},
			max_tokens = 5
		})
	else
		-- Request de prueba para Gemini
		url = GEMINI_MODEL_URL .. "?key=" .. apiKey
		headers = {["Content-Type"] = "application/json"}
		body = HttpService:JSONEncode({
			contents = {{role = "user", parts = {{text = "test"}}}},
			generationConfig = {maxOutputTokens = 5}
		})
	end
	
	local success, res = pcall(function()
		return HttpRequest({
			Url = url,
			Method = "POST",
			Headers = headers,
			Body = body
		})
	end)
	
	Settings.IsThinking = false
	
	if success and res and res.Body then
		local data
		local decodeSuccess = pcall(function() 
			data = HttpService:JSONDecode(res.Body) 
		end)
		
		if not decodeSuccess then
			return false, "Respuesta inválida del servidor"
		end
		
		-- Validar respuesta según proveedor
		local isValid = false
		
		if detectedProvider == "groq" then
			if data.choices and data.choices[1] then
				isValid = true
			elseif data.error then
				if data.error.code == "invalid_api_key" then
					return false, "API Key de Groq inválida"
				end
				return false, data.error.message or "Error de Groq"
			end
		else
			if data.candidates and data.candidates[1] then
				isValid = true
			elseif data.error then
				if data.error.code == 400 then
					return false, "API Key de Gemini inválida"
				elseif data.error.code == 429 or data.error.code == 503 then
					Settings.IsServerBusy = true
					BusyStartTime = tick()
					if not Settings.IsProbing then 
						task.spawn(ProbeServer) 
					end
					return nil, "BUSY"
				end
				return false, data.error.message or "Error de Gemini"
			end
		end
		
		if isValid then
			API_PROVIDER = detectedProvider
			UpdateConsole("Validation Success", detectedProvider:upper() .. " OK")
			Settings.NextReqTime = tick() + 4
			return true, "API Key válida (" .. detectedProvider .. ")"
		else
			return false, "Respuesta vacía del servidor"
		end
	else
		UpdateConsole("Network Timeout", "Check Connection")
		return false, "Error de conexión HTTP"
	end
end
-- ============================================
-- FUNCIONES DE ANIMACIÓN (DEFINIR ANTES DE LA UI)
-- ============================================
-- Función para animación de cierre elegante del UI de autenticación
local function CloseAuthWithAnimation(frame, callback)
	-- Animación: cerrar desde arriba y abajo mientras se hace transparente
	local tweenInfo = TweenInfo.new(0.6, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
	-- Crear tweens para todos los elementos del frame
	local frameTween = TweenService:Create(frame, tweenInfo, {
		Size = UDim2.new(frame.Size.X.Scale, frame.Size.X.Offset, 0, 0),
		Position = UDim2.new(frame.Position.X.Scale, frame.Position.X.Offset, 0.5, 0),
		BackgroundTransparency = 1
	})
	-- Hacer transparentes todos los hijos
	for _, child in pairs(frame:GetDescendants()) do
		if child:IsA("GuiObject") then
			local childTween = TweenService:Create(child, tweenInfo, {
				BackgroundTransparency = 1,
				TextTransparency = 1
			})
			childTween:Play()
		end
	end
	frameTween:Play()
	frameTween.Completed:Connect(function()
		frame.Visible = false
		if callback then callback() end
	end)
end
-- Función para mostrar MainFrame con animación (CORREGIDA)
local function ShowMainFrameWithAnimation()
	if not MainFrame then 
		warn("[Lux] MainFrame no existe aún")
		return 
	end
	MainFrame.Size = UDim2.new(0, 300, 0, 0)
	MainFrame.Position = UDim2.new(0.5, -150, 0.5, 0)
	MainFrame.BackgroundTransparency = 1
	MainFrame.Visible = true
	-- Hacer todos los elementos transparentes primero
	for _, child in pairs(MainFrame:GetDescendants()) do
		if child:IsA("GuiObject") then
			pcall(function()
				child.BackgroundTransparency = 1
			end)
			if child:IsA("TextLabel") or child:IsA("TextButton") or child:IsA("TextBox") then
				pcall(function()
					child.TextTransparency = 1
				end)
			end
		end
	end
	-- Animar apertura
	local tweenInfo = TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
	local frameTween = TweenService:Create(MainFrame, tweenInfo, {
		Size = UDim2.new(0, 300, 0, 180),
		Position = UDim2.new(0.5, -150, 0.5, -90),
		BackgroundTransparency = 0
	})
	frameTween:Play()
	-- ✅ USAR SPAWN EN LUGAR DE WAIT (no bloquea el hilo principal)
	task.spawn(function()
		task.wait(0.2)
		for _, child in pairs(MainFrame:GetDescendants()) do
			if child:IsA("GuiObject") and child ~= MainFrame then
				pcall(function()
					local originalBgTrans = 0
					local originalTextTrans = 0
					if child:IsA("Frame") and child.Name == "StatusIndicator" then
						originalBgTrans = 0
					elseif child.Name:match("Corner") then
						originalBgTrans = 1
					elseif child:IsA("TextButton") then
						originalBgTrans = 0
						originalTextTrans = 0
					elseif child:IsA("TextLabel") then
						originalBgTrans = 1
						originalTextTrans = 0
					end
					local childTween = TweenService:Create(child, TweenInfo.new(0.3), {
						BackgroundTransparency = originalBgTrans,
						TextTransparency = originalTextTrans
					})
					childTween:Play()
				end)
			end
		end
	end)
end
-- Función para ocultar MainFrame con animación (CORREGIDA)
local function HideMainFrameWithAnimation(callback)
	if not MainFrame then 
		warn("[Lux] MainFrame no existe")
		if callback then callback() end
		return 
	end
	local tweenInfo = TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.In)
	-- Animar todos los elementos a transparente CON VALIDACIÓN
	for _, child in pairs(MainFrame:GetDescendants()) do
		if child:IsA("GuiObject") then
			pcall(function()
				-- Crear tabla de propiedades a animar
				local tweenProperties = {}
				-- Solo agregar BackgroundTransparency si el objeto la tiene
				if child:IsA("Frame") or child:IsA("TextButton") or child:IsA("TextBox") or child:IsA("TextLabel") or child:IsA("ImageLabel") or child:IsA("ImageButton") then
					tweenProperties.BackgroundTransparency = 1
				end
				-- Solo agregar TextTransparency si el objeto es de texto
				if child:IsA("TextLabel") or child:IsA("TextButton") or child:IsA("TextBox") then
					tweenProperties.TextTransparency = 1
				end
				-- Solo agregar ImageTransparency si el objeto es imagen
				if child:IsA("ImageLabel") or child:IsA("ImageButton") then
					tweenProperties.ImageTransparency = 1
				end
				-- Solo crear tween si hay propiedades para animar
				if next(tweenProperties) then
					local childTween = TweenService:Create(child, tweenInfo, tweenProperties)
					childTween:Play()
				end
			end)
		end
	end
	-- Animar el frame principal
	local frameTween = TweenService:Create(MainFrame, tweenInfo, {
		Size = UDim2.new(0, 300, 0, 0),
		Position = UDim2.new(0.5, -150, 0.5, 0),
		BackgroundTransparency = 1
	})
	frameTween:Play()
	frameTween.Completed:Connect(function()
		MainFrame.Visible = false
		if callback then callback() end
	end)
end
-- ============================================
-- CREAR UI MODERNA CON SISTEMA DE DOS FASES
-- ============================================
ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "LuxAPISystem"
ScreenGui.ResetOnSpawn = false
if game:GetService("CoreGui"):FindFirstChild("LuxAPISystem") then 
	game:GetService("CoreGui").LuxAPISystem:Destroy() 
end
ScreenGui.Parent = game:GetService("CoreGui")
-- ============================================
-- FASE 1: AUTH FRAME (Validación de API Key)
-- ============================================
AuthFrame = Instance.new("Frame")
AuthFrame.Name = "AuthFrame"
AuthFrame.Size = UDim2.new(0, 360, 0, 260)
AuthFrame.Position = UDim2.new(0.5, -180, 0.5, -130)
AuthFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
AuthFrame.BorderSizePixel = 0
AuthFrame.Visible = false -- Se mostrará si no hay validación guardada
AuthFrame.Parent = ScreenGui
local authCorner = Instance.new("UICorner")
authCorner.CornerRadius = UDim.new(0, 14)
authCorner.Parent = AuthFrame
local authGradient = Instance.new("UIGradient")
authGradient.Color = ColorSequence.new{
	ColorSequenceKeypoint.new(0, Color3.fromRGB(25, 25, 30)),
	ColorSequenceKeypoint.new(1, Color3.fromRGB(15, 15, 20))
}
authGradient.Rotation = 45
authGradient.Parent = AuthFrame
local function MakeDraggable(frame)
	local dragging, dragInput, dragStart, startPos
	local function update(input)
		local delta = input.Position - dragStart
		frame.Position = UDim2.new(
			startPos.X.Scale, 
			startPos.X.Offset + delta.X, 
			startPos.Y.Scale, 
			startPos.Y.Offset + delta.Y
		)
	end
	frame.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or 
		   input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			dragStart = input.Position
			startPos = frame.Position
			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then 
					dragging = false 
				end
			end)
		end
	end)
	frame.InputChanged:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseMovement or 
		   input.UserInputType == Enum.UserInputType.Touch then 
			dragInput = input 
		end
	end)
	UserInputService.InputChanged:Connect(function(input)
		if input == dragInput and dragging then 
			update(input) 
		end
	end)
end
MakeDraggable(AuthFrame)
local authTitle = Instance.new("TextLabel")
authTitle.Size = UDim2.new(1, -40, 0, 45)
authTitle.Position = UDim2.new(0, 20, 0, 15)
authTitle.BackgroundTransparency = 1
authTitle.Text = "⚡ LUX AI - SETUP"
authTitle.TextColor3 = Color3.fromRGB(100, 200, 255)
authTitle.Font = Enum.Font.GothamBold
authTitle.TextSize = 20
authTitle.TextXAlignment = Enum.TextXAlignment.Left
authTitle.Parent = AuthFrame
local authInstruction = Instance.new("TextLabel")
authInstruction.Size = UDim2.new(1, -40, 0, 25)
authInstruction.Position = UDim2.new(0, 20, 0, 70)
authInstruction.BackgroundTransparency = 1
authInstruction.Text = "Enter your Gemini or Groq API Key:"
authInstruction.TextColor3 = Color3.fromRGB(180, 180, 185)
authInstruction.Font = Enum.Font.Gotham
authInstruction.TextSize = 13
authInstruction.TextXAlignment = Enum.TextXAlignment.Left
authInstruction.Parent = AuthFrame
APIKeyBox = Instance.new("TextBox")
APIKeyBox.Name = "APIKeyBox"
APIKeyBox.Size = UDim2.new(1, -40, 0, 42)
APIKeyBox.Position = UDim2.new(0, 20, 0, 100)
APIKeyBox.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
APIKeyBox.BorderSizePixel = 0
APIKeyBox.Text = ""
APIKeyBox.PlaceholderText = "Paste your API key here..."
APIKeyBox.PlaceholderColor3 = Color3.fromRGB(120, 120, 125)
APIKeyBox.TextColor3 = Color3.fromRGB(255, 255, 255)
APIKeyBox.Font = Enum.Font.GothamMedium
APIKeyBox.TextSize = 12
APIKeyBox.ClearTextOnFocus = false
APIKeyBox.Parent = AuthFrame
local keyBoxCorner = Instance.new("UICorner")
keyBoxCorner.CornerRadius = UDim.new(0, 8)
keyBoxCorner.Parent = APIKeyBox
local getKeyButton = Instance.new("TextButton")
getKeyButton.Size = UDim2.new(1, -40, 0, 32)
getKeyButton.Position = UDim2.new(0, 20, 0, 155)
getKeyButton.BackgroundColor3 = Color3.fromRGB(70, 130, 200)
getKeyButton.Text = "📋 Get API Key (Copy Link)"
getKeyButton.TextColor3 = Color3.fromRGB(255, 255, 255)
getKeyButton.Font = Enum.Font.GothamSemibold
getKeyButton.TextSize = 12
getKeyButton.Parent = AuthFrame
local getKeyCorner = Instance.new("UICorner")
getKeyCorner.CornerRadius = UDim.new(0, 8)
getKeyCorner.Parent = getKeyButton
getKeyButton.MouseButton1Click:Connect(function()
	if setclipboard then
		setclipboard("https://aistudio.google.com/app/apikey")
		getKeyButton.Text = "✓ Link Copied!"
		task.wait(1.5)
		getKeyButton.Text = "📋 Get API Key (Copy Link)"
	end
end)
ValidateButton = Instance.new("TextButton")
ValidateButton.Name = "ValidateButton"
ValidateButton.Size = UDim2.new(1, -40, 0, 45)
ValidateButton.Position = UDim2.new(0, 20, 0, 200)
ValidateButton.BackgroundColor3 = Color3.fromRGB(80, 160, 255)
ValidateButton.Text = "✓ VALIDATE & START"
ValidateButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ValidateButton.Font = Enum.Font.GothamBold
ValidateButton.TextSize = 14
ValidateButton.Parent = AuthFrame
local validateCorner = Instance.new("UICorner")
validateCorner.CornerRadius = UDim.new(0, 8)
validateCorner.Parent = ValidateButton
-- ============================================
-- FASE 2: MAIN FRAME (Control de IA)
-- ============================================
MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 300, 0, 180)
MainFrame.Position = UDim2.new(0.5, -150, 0.5, -90)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
MainFrame.BorderSizePixel = 0
MainFrame.Visible = false
MainFrame.Parent = ScreenGui
local mainCorner = Instance.new("UICorner")
mainCorner.CornerRadius = UDim.new(0, 14)
mainCorner.Parent = MainFrame
local mainGradient = Instance.new("UIGradient")
mainGradient.Color = ColorSequence.new{
	ColorSequenceKeypoint.new(0, Color3.fromRGB(25, 25, 30)),
	ColorSequenceKeypoint.new(1, Color3.fromRGB(15, 15, 20))
}
mainGradient.Rotation = 45
mainGradient.Parent = MainFrame
MakeDraggable(MainFrame)
local mainTitle = Instance.new("TextLabel")
mainTitle.Size = UDim2.new(1, -60, 0, 40)
mainTitle.Position = UDim2.new(0, 15, 0, 10)
mainTitle.BackgroundTransparency = 1
mainTitle.Text = "⚡ LUX AI"
mainTitle.TextColor3 = Color3.fromRGB(100, 200, 255)
mainTitle.Font = Enum.Font.GothamBold
mainTitle.TextSize = 18
mainTitle.TextXAlignment = Enum.TextXAlignment.Left
mainTitle.Parent = MainFrame
StatusIndicator = Instance.new("Frame")
StatusIndicator.Name = "StatusIndicator"
StatusIndicator.Size = UDim2.new(0, 12, 0, 12)
StatusIndicator.Position = UDim2.new(1, -25, 0, 24)
StatusIndicator.BackgroundColor3 = Color3.fromRGB(80, 220, 120)
StatusIndicator.BorderSizePixel = 0
StatusIndicator.Parent = MainFrame
local indicatorCorner = Instance.new("UICorner")
indicatorCorner.CornerRadius = UDim.new(1, 0)
indicatorCorner.Parent = StatusIndicator
task.spawn(function()
	while ScreenGui.Parent do
		TweenService:Create(StatusIndicator, TweenInfo.new(0.8, Enum.EasingStyle.Sine), {
			Size = UDim2.new(0, 14, 0, 14)
		}):Play()
		task.wait(0.8)
		TweenService:Create(StatusIndicator, TweenInfo.new(0.8, Enum.EasingStyle.Sine), {
			Size = UDim2.new(0, 12, 0, 12)
		}):Play()
		task.wait(0.8)
	end
end)
local closeButton = Instance.new("TextButton")
closeButton.Size = UDim2.new(0, 28, 0, 28)
closeButton.Position = UDim2.new(1, -40, 0, 12)
closeButton.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
closeButton.Text = "✕"
closeButton.TextColor3 = Color3.fromRGB(255, 255, 255)
closeButton.Font = Enum.Font.GothamBold
closeButton.TextSize = 14
closeButton.Parent = MainFrame
local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(0, 6)
closeCorner.Parent = closeButton
closeButton.MouseButton1Click:Connect(function()
	HideMainFrameWithAnimation(function()
		-- Mostrar botón "Mostrar"
		ShowButton.Visible = true
		local showTween = TweenService:Create(ShowButton, TweenInfo.new(0.3), {
			BackgroundTransparency = 0.5,
			TextTransparency = 0
		})
		showTween:Play()
	end)
end)
ToggleButton = Instance.new("TextButton")
ToggleButton.Name = "ToggleButton"
ToggleButton.Size = UDim2.new(0.9, 0, 0, 45)
ToggleButton.Position = UDim2.new(0.05, 0, 0.35, 0)
ToggleButton.BackgroundColor3 = Color3.fromRGB(90, 90, 95)
ToggleButton.Text = "💡 ACTIVATE AI"
ToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleButton.Font = Enum.Font.GothamBold
ToggleButton.TextSize = 14
ToggleButton.Parent = MainFrame
local toggleCorner = Instance.new("UICorner")
toggleCorner.CornerRadius = UDim.new(0, 8)
toggleCorner.Parent = ToggleButton
local modeButton = Instance.new("TextButton")
modeButton.Size = UDim2.new(0.43, 0, 0, 35)
modeButton.Position = UDim2.new(0.05, 0, 0.68, 0)
modeButton.BackgroundColor3 = PRIVATE_MODE and Color3.fromRGB(150, 80, 200) or Color3.fromRGB(80, 150, 200)
modeButton.Text = PRIVATE_MODE and "🔐 PRIVATE" or "🌍 PUBLIC"
modeButton.TextColor3 = Color3.fromRGB(255, 255, 255)
modeButton.Font = Enum.Font.GothamSemibold
modeButton.TextSize = 11
modeButton.Parent = MainFrame
local modeCorner = Instance.new("UICorner")
modeCorner.CornerRadius = UDim.new(0, 8)
modeCorner.Parent = modeButton
modeButton.MouseButton1Click:Connect(function()
	PRIVATE_MODE = not PRIVATE_MODE
	modeButton.BackgroundColor3 = PRIVATE_MODE and Color3.fromRGB(150, 80, 200) or Color3.fromRGB(80, 150, 200)
	modeButton.Text = PRIVATE_MODE and "🔐 PRIVATE" or "🌍 PUBLIC"
	print("[Lux] Mode:", PRIVATE_MODE and "PRIVATE" or "PUBLIC")
end)
local clearHistoryButton = Instance.new("TextButton")
clearHistoryButton.Size = UDim2.new(0.43, 0, 0, 35)
clearHistoryButton.Position = UDim2.new(0.52, 0, 0.68, 0)
clearHistoryButton.BackgroundColor3 = Color3.fromRGB(200, 100, 60)
clearHistoryButton.Text = "🗑️ CLEAR"
clearHistoryButton.TextColor3 = Color3.fromRGB(255, 255, 255)
clearHistoryButton.Font = Enum.Font.GothamSemibold
clearHistoryButton.TextSize = 11
clearHistoryButton.Parent = MainFrame
local clearCorner = Instance.new("UICorner")
clearCorner.CornerRadius = UDim.new(0, 8)
clearCorner.Parent = clearHistoryButton
clearHistoryButton.MouseButton1Click:Connect(function()
	ClearChatHistory()
	clearHistoryButton.Text = "✓ DONE"
	task.wait(1)
	clearHistoryButton.Text = "🗑️ CLEAR"
end)
ConsoleLabel = Instance.new("TextLabel")
ConsoleLabel.Name = "ConsoleLabel"
ConsoleLabel.Size = UDim2.new(0.9, 0, 0, 15)
ConsoleLabel.Position = UDim2.new(0.05, 0, 0.90, 0)
ConsoleLabel.BackgroundTransparency = 1
ConsoleLabel.Text = "System Ready"
ConsoleLabel.TextColor3 = Color3.fromRGB(140, 140, 145)
ConsoleLabel.Font = Enum.Font.Code
ConsoleLabel.TextSize = 9
ConsoleLabel.TextXAlignment = Enum.TextXAlignment.Left
ConsoleLabel.Parent = MainFrame
-- ============================================
-- BOTÓN "MOSTRAR" (aparece cuando se oculta el MainFrame)
-- ============================================
ShowButton = Instance.new("TextButton")
ShowButton.Name = "ShowButton"
ShowButton.Size = UDim2.new(0, 100, 0, 35)
ShowButton.Position = UDim2.new(0.5, -50, 0.02, 0)
ShowButton.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
ShowButton.BackgroundTransparency = 1
ShowButton.Text = "MOSTRAR"
ShowButton.TextColor3 = Color3.fromRGB(100, 200, 255)
ShowButton.TextTransparency = 1
ShowButton.Font = Enum.Font.GothamBold
ShowButton.TextSize = 12
ShowButton.Visible = false
ShowButton.Parent = ScreenGui
local showCorner = Instance.new("UICorner")
showCorner.CornerRadius = UDim.new(0, 8)
showCorner.Parent = ShowButton
ShowButton.MouseButton1Click:Connect(function()
	-- Ocultar botón mostrar
	local hideTween = TweenService:Create(ShowButton, TweenInfo.new(0.2), {
		BackgroundTransparency = 1,
		TextTransparency = 1
	})
	hideTween:Play()
	hideTween.Completed:Connect(function()
		ShowButton.Visible = false
	end)
	-- Mostrar MainFrame con animación
	ShowMainFrameWithAnimation()
end)
-- Función para mostrar feedback
local function ShowFeedback(message, isSuccess)
	local feedbackLabel = Instance.new("TextLabel")
	feedbackLabel.Size = UDim2.new(1, -40, 0, 28)
	feedbackLabel.Position = UDim2.new(0, 20, 0, 165)
	feedbackLabel.BackgroundColor3 = isSuccess and Color3.fromRGB(60, 180, 100) or Color3.fromRGB(220, 70, 70)
	feedbackLabel.Text = message
	feedbackLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
	feedbackLabel.Font = Enum.Font.GothamBold
	feedbackLabel.TextSize = 11
	feedbackLabel.TextTransparency = 0
	feedbackLabel.BackgroundTransparency = 0
	feedbackLabel.Parent = MainFrame
	local feedbackCorner = Instance.new("UICorner")
	feedbackCorner.CornerRadius = UDim.new(0, 6)
	feedbackCorner.Parent = feedbackLabel
	task.wait(2.5)
	local tween = TweenService:Create(feedbackLabel, TweenInfo.new(0.5), {
		TextTransparency = 1,
		BackgroundTransparency = 1
	})
	tween:Play()
	tween.Completed:Connect(function()
		feedbackLabel:Destroy()
	end)
end
-- Función para habilitar/deshabilitar el botón toggle
local function SetToggleEnabled(enabled)
	if enabled then
		ToggleButton.BackgroundColor3 = Color3.fromRGB(90, 90, 95)
		ToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
		ToggleButton.Text = "💡 ACTIVATE AI"
		ToggleButton.AutoButtonColor = true
	else
		ToggleButton.BackgroundColor3 = Color3.fromRGB(70, 70, 75)
		ToggleButton.TextColor3 = Color3.fromRGB(160, 160, 165)
		ToggleButton.Text = "🔒 LOCKED"
		ToggleButton.AutoButtonColor = false
	end
end
-- Evento del botón de validación CON ANIMACIÓN
ValidateButton.MouseButton1Click:Connect(function()
	local apiKey = APIKeyBox.Text:gsub("%s+", "")
	if apiKey == "" then
		APIKeyBox.PlaceholderText = "⚠️ Please enter a key!"
		APIKeyBox.PlaceholderColor3 = Color3.fromRGB(220, 70, 70)
		task.wait(2)
		APIKeyBox.PlaceholderText = "Paste your API key here..."
		APIKeyBox.PlaceholderColor3 = Color3.fromRGB(120, 120, 125)
		return
	end
	ValidateButton.Text = "⏳ VALIDATING..."
	ValidateButton.BackgroundColor3 = Color3.fromRGB(120, 120, 125)
	task.spawn(function()
		local isValid, message = ValidateAPIKey(apiKey, true)
		if isValid then
			API_KEY = apiKey
			SaveAPIKey(apiKey)
			SaveValidation(apiKey, API_PROVIDER) -- ✅ GUARDAR VALIDACIÓN CON PROVEEDOR
			ValidateButton.Text = "✅ VALIDATED!"
			ValidateButton.BackgroundColor3 = Color3.fromRGB(60, 180, 100)
			_G.LuxAPIKey = apiKey
			_G.LuxAPIReady = true
			print("[Lux] API Key validated and saved")
			task.wait(0.8)
			-- 🎬 ANIMACIÓN: Cerrar AuthFrame elegantemente
			CloseAuthWithAnimation(AuthFrame, function()
				-- Después de cerrar AuthFrame, mostrar MainFrame
				ShowMainFrameWithAnimation()
				-- Activar IA automáticamente
				Settings.AI_Enabled = true
				ToggleButton.Text = "💡 AI ACTIVE"
				ToggleButton.BackgroundColor3 = Color3.fromRGB(60, 180, 100)
				_G.LuxAIEnabled = true
				UpdateConsole("AI Active", "Ready")
				print("[Lux] AI system activated")
			end)
		elseif message == "BUSY" then
			ValidateButton.Text = "⏳ Server busy..."
			task.wait(2)
			ValidateButton.Text = "✓ VALIDATE & START"
			ValidateButton.BackgroundColor3 = Color3.fromRGB(80, 160, 255)
		else
			ValidateButton.Text = "❌ " .. (message or "Error")
			ValidateButton.BackgroundColor3 = Color3.fromRGB(220, 70, 70)
			task.wait(2)
			ValidateButton.Text = "✓ VALIDATE & START"
			ValidateButton.BackgroundColor3 = Color3.fromRGB(80, 160, 255)
			warn("[Lux] Validation error:", message)
		end
	end)
end)
-- Evento del botón toggle
ToggleButton.MouseButton1Click:Connect(function()
	if API_KEY == "" or not _G.LuxAPIReady then
		ShowFeedback("⚠️ Validate API Key first", false)
		return
	end
	Settings.AI_Enabled = not Settings.AI_Enabled
	if Settings.AI_Enabled then
		ToggleButton.Text = "💡 AI ACTIVE"
		ToggleButton.BackgroundColor3 = Color3.fromRGB(60, 180, 100)
		ShowFeedback("✅ Lux AI activated", true)
		UpdateConsole("AI Status", "ACTIVE")
		-- Guardar estado
		_G.LuxAIEnabled = true
		print("[Lux] AI system activated")
	else
		ToggleButton.Text = "💡 ACTIVATE AI"
		ToggleButton.BackgroundColor3 = Color3.fromRGB(90, 90, 95)
		ShowFeedback("⏸️ Lux AI deactivated", true)
		UpdateConsole("AI Status", "STANDBY")
		_G.LuxAIEnabled = false
		print("[Lux] AI system deactivated")
	end
end)
-- Auto-reset de "thinking" si se queda pegado
task.spawn(function()
	while ScreenGui.Parent do
		task.wait(2)
		if Settings.IsThinking and (tick() - ThinkingStartTime > API_TIMEOUT_SECONDS + 1) then
			Settings.IsThinking = false
			UpdateConsole("Auto-Reset", "Idle")
		end
	end
end)
-- ============================================
-- INICIO AUTOMÁTICO CON VALIDACIÓN GUARDADA
-- ============================================
task.spawn(function()
	task.wait(0.5) -- Esperar que todo cargue
	local savedValidation, savedProvider = LoadValidation()
	if savedValidation then
		-- ✅ HAY VALIDACIÓN GUARDADA - Iniciar directamente
		print("[Lux] Loading saved validation...")
		API_KEY = savedValidation
		API_PROVIDER = savedProvider or "gemini" -- AGREGAR ESTA LÍNEA
		_G.LuxAPIKey = savedValidation
		_G.LuxAPIReady = true
		-- Mostrar MainFrame directamente (sin AuthFrame)
		ShowMainFrameWithAnimation()
		-- Activar IA automáticamente
		task.wait(0.3)
		Settings.AI_Enabled = true
		ToggleButton.Text = "💡 AI ACTIVE"
		ToggleButton.BackgroundColor3 = Color3.fromRGB(60, 180, 100)
		_G.LuxAIEnabled = true
		UpdateConsole("AI Active", "Auto-started")
		print("[Lux] AI system auto-started with saved validation")
	else
		-- ❌ NO HAY VALIDACIÓN - Mostrar AuthFrame
		print("[Lux] No saved validation found, showing auth screen")
		AuthFrame.Visible = true
		-- Intentar cargar API Key antigua (para rellenar el campo)
		local savedKey = LoadAPIKey()
		if savedKey then
			APIKeyBox.Text = savedKey
		end
	end
end)
-- ============================================
-- SISTEMA DE CHAT Y RESPUESTAS DE IA
-- ============================================
local TextChatService = game:GetService("TextChatService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local BOT_NAME = "lux" -- Nombre para activar el bot
local CHAT_COOLDOWN = 3 -- Segundos entre respuestas
local lastChatTime = 0
-- Sistema de historial de conversaciones
local MAX_HISTORY_LENGTH = 10 -- Máximo de mensajes en memoria (5 intercambios)
local ChatHistory = {} -- {role: "user/model", parts: {{text = "..."}}}
local HISTORY_FILE_NAME = "Lux_ChatHistory.json"
-- ============================================
-- SISTEMA DE CONVERSACIONES ACTIVAS
-- ============================================
local ActiveConversations = {} -- {[UserId] = true}
local ACTIVE_CONVERSATIONS_FILE = "Lux_ActiveConversations.json"
local CONVERSATION_TIMEOUT = 300 -- 5 minutos de inactividad = se quita de la lista
local LastMessageTime = {} -- {[UserId] = tick()}
-- Función para guardar historial
local function SaveChatHistory()
	if not writefile then return end
	pcall(function()
		local data = {
			history = ChatHistory,
			lastSaved = os.time()
		}
		writefile(HISTORY_FILE_NAME, HttpService:JSONEncode(data))
	end)
end
-- Función para cargar historial
local function LoadChatHistory()
	if not isfile or not isfile(HISTORY_FILE_NAME) then return end
	pcall(function()
		local content = readfile(HISTORY_FILE_NAME)
		local data = HttpService:JSONDecode(content)
		if data.history then
			ChatHistory = data.history
			print("[Lux] Historial cargado:", #ChatHistory, "mensajes")
		end
	end)
end
-- Función para limpiar historial
local function ClearChatHistory()
	ChatHistory = {}
	SaveChatHistory()
	print("[Lux] Historial limpiado")
end
-- ============================================
-- FUNCIONES DEL SISTEMA DE CONVERSACIONES
-- ============================================
-- Guardar lista de conversaciones activas
local function SaveActiveConversations()
	if not writefile then return end
	pcall(function()
		local saveData = {}
		for userId, _ in pairs(ActiveConversations) do
			table.insert(saveData, userId)
		end
		writefile(ACTIVE_CONVERSATIONS_FILE, HttpService:JSONEncode(saveData))
	end)
end
-- Cargar lista de conversaciones activas
local function LoadActiveConversations()
	if not isfile or not isfile(ACTIVE_CONVERSATIONS_FILE) then return end
	pcall(function()
		local content = readfile(ACTIVE_CONVERSATIONS_FILE)
		local saveData = HttpService:JSONDecode(content)
		for _, userId in ipairs(saveData) do
			ActiveConversations[userId] = true
		end
		print("[Lux] Conversaciones activas cargadas:", #saveData, "usuarios")
	end)
end
-- Agregar usuario a conversación activa
local function AddToActiveConversation(userId)
	if not ActiveConversations[userId] then
		ActiveConversations[userId] = true
		LastMessageTime[userId] = tick()
		SaveActiveConversations()
		print("[Lux] Usuario", userId, "agregado a conversaciones activas")
	else
		-- Actualizar timestamp
		LastMessageTime[userId] = tick()
	end
end
-- Verificar si usuario está en conversación activa
local function IsInActiveConversation(userId)
	return ActiveConversations[userId] == true
end
-- Remover usuario de conversación activa
local function RemoveFromActiveConversation(userId)
	if ActiveConversations[userId] then
		ActiveConversations[userId] = nil
		LastMessageTime[userId] = nil
		SaveActiveConversations()
		print("[Lux] Usuario", userId, "removido de conversaciones activas")
	end
end
-- Limpiar conversaciones inactivas (timeout)
local function CleanupInactiveConversations()
	local currentTime = tick()
	local removed = 0
	for userId, _ in pairs(ActiveConversations) do
		local lastTime = LastMessageTime[userId] or 0
		if (currentTime - lastTime) > CONVERSATION_TIMEOUT then
			RemoveFromActiveConversation(userId)
			removed = removed + 1
		end
	end
	if removed > 0 then
		print("[Lux] Limpiadas", removed, "conversaciones inactivas")
	end
end
-- Obtener contador de conversaciones activas
local function GetActiveConversationsCount()
	local count = 0
	for _ in pairs(ActiveConversations) do
		count = count + 1
	end
	return count
end
-- Función para enviar mensajes al chat de Roblox
local function SendChatMessage(text)
	if not text or text:gsub("%s+", "") == "" then return end
	local messageSent = false
	-- Método 1: TextChatService (nuevo sistema)
	if not messageSent and TextChatService.ChatVersion == Enum.ChatVersion.TextChatService then
		local channels = TextChatService:FindFirstChild("TextChannels")
		if channels then
			local targetChannel = channels:FindFirstChild("RBXGeneral") or channels:FindFirstChild("General")
			if targetChannel then
				pcall(function() 
					targetChannel:SendAsync("[Lux]: " .. text) 
					messageSent = true 
				end)
			end
		end
	end
	-- Método 2: ChatInputBarConfiguration
	if not messageSent and TextChatService:FindFirstChild("ChatInputBarConfiguration") then
		local config = TextChatService.ChatInputBarConfiguration
		if config.TargetTextChannel then
			pcall(function() 
				config.TargetTextChannel:SendAsync("[Lux]: " .. text) 
				messageSent = true 
			end)
		end
	end
	-- Método 3: Legacy chat system
	if not messageSent then
		local events = ReplicatedStorage:FindFirstChild("DefaultChatSystemChatEvents")
		if events then
			local sayRequest = events:FindFirstChild("SayMessageRequest")
			if sayRequest then
				pcall(function() 
					sayRequest:FireServer("[Lux]: " .. text, "All") 
					messageSent = true 
				end)
			end
		end
	end
	-- Método 4: Último recurso
	if not messageSent then
		pcall(function() 
			game:GetService("Players"):Chat("[Lux]: " .. text) 
		end)
	end
end
-- Función principal para hacer request a la IA (Gemini o Groq)
local function AskAI(userMessage, playerName)
	if API_KEY == "" or not _G.LuxAPIReady then
		return nil, "API Key no configurada"
	end
	
	if not Settings.AI_Enabled then
		return nil, "IA no activada"
	end
	
	if tick() < Settings.NextReqTime then
		return nil, "Cooldown activo"
	end
	
	if Settings.IsThinking then
		return nil, "Ya estoy procesando otra petición"
	end
	
	if Settings.IsServerBusy then
		if not Settings.IsProbing then
			task.spawn(ProbeServer)
		end
		return nil, "Servidor ocupado"
	end
	
	Settings.IsThinking = true
	ThinkingStartTime = tick()
	UpdateConsole("Processing: " .. userMessage:sub(1, 20), "Thinking...")
	
	local environmentContext = FormatEnvironmentForLux()
	
    -- ⭐ PERSONALIDAD MEJORADA PARA PLS DONATE
    local personality = [[
You are Lux, a chill and funny AI friend who loves hanging out in Roblox.

PERSONALITY CORE:
- 😎 Super relaxed and conversational - like texting a close friend
- 😂 Naturally funny - use humor, memes references, internet slang (Spanish)
- 🤝 Genuinely interested in people - ask follow-up questions
- 💬 Keep people talking - be engaging and curious
- 🎮 Gamer vibe - references to Roblox, games, trends

CONVERSATIONAL STYLE:
- Use "jaja", "lol", "xd" naturally (not every message)
- Emojis sparingly but effectively (1-2 max per message)
- Short sentences - NEVER exceed 180 characters (Roblox chat limit!)
- Ask open questions to keep conversation going
- Remember details from previous messages
- Be genuine, not robotic

IMPORTANT RULES:
- ALWAYS respond in casual Spanish (Mexico/Latin America style)
- Max 180 characters INCLUDING spaces and emojis
- NEVER mention donations, money, or robux first
- If they mention being broke/poor, empathize naturally
- Make people WANT to keep talking to you
- Reference their previous messages naturally

CONVERSATION TACTICS:
- First message: Friendly greeting + casual question
- Follow-up: Show genuine interest, make jokes
- Build connection through: humor, shared experiences, compliments
- NEVER: Be pushy, formal, or salesy
- DO: Be the friend they want to talk to

FORBIDDEN:
- Don't ask "how can I help you" (too formal)
- Don't be overly enthusiastic (!!!! ✨✨✨)
- Don't mention "Lux" or "AI" unless asked
- Don't give long explanations

EXAMPLES OF GOOD RESPONSES:
"Ey qué onda! 😎 ¿Viniste a farmear o solo a viciar un rato?"
"Jaja me identifico, siempre estoy pobre en Roblox 💀 ¿Cuál es tu juego favorito?"
"Naaah eso estuvo cerca jaja ¿Cuánto tiempo llevas jugando?"
"Respeto 🔥 Yo también soy más de [theme]. ¿Y qué música escuchas?"

CONTEXT:
Player: ]] .. playerName .. [[
Game: PLS DONATE (people here to donate/receive)
Time: ]] .. os.date("%H:%M") .. [[

Be their cool friend. Keep it real. Keep them engaged.
]]
	
	local url, requestBody, headers
	
	if API_PROVIDER == "groq" then
		-- FORMATO GROQ (OpenAI-compatible)
		url = GROQ_MODEL_URL
		headers = {
			["Content-Type"] = "application/json",
			["Authorization"] = "Bearer " .. API_KEY
		}
		
		local messages = {{role = "system", content = personality}}
		
		-- Agregar historial en formato Groq
		for _, msg in ipairs(ChatHistory) do
			if msg.role == "user" then
				table.insert(messages, {role = "user", content = msg.parts[1].text})
			else
				table.insert(messages, {role = "assistant", content = msg.parts[1].text})
			end
		end
		
		-- Mensaje actual
		table.insert(messages, {role = "user", content = userMessage})
		
		requestBody = HttpService:JSONEncode({
			model = GROQ_MODEL,
			messages = messages,
			max_tokens = 120,
			temperature = 0.9
		})
		
	else
		-- FORMATO GEMINI
		url = GEMINI_MODEL_URL .. "?key=" .. API_KEY
		headers = {["Content-Type"] = "application/json"}
		
		local contents = {}
		for _, message in ipairs(ChatHistory) do
			table.insert(contents, message)
		end
		table.insert(contents, {role = "user", parts = {{text = userMessage}}})
		
		requestBody = HttpService:JSONEncode({
			systemInstruction = {parts = {{text = personality}}},
			contents = contents,
			generationConfig = {
				maxOutputTokens = 120,
				temperature = 0.9,
				topP = 0.95,
				topK = 40
			}
		})
	end
	
	local success, res = pcall(function()
		return HttpRequest({
			Url = url,
			Method = "POST",
			Headers = headers,
			Body = requestBody
		})
	end)
	
	Settings.IsThinking = false
	
	if success and res and res.Body then
		local data
		local decodeSuccess = pcall(function()
			data = HttpService:JSONDecode(res.Body)
		end)
		
		if not decodeSuccess then
			UpdateConsole("Error: JSON Decode Failed", nil)
			return nil, "Error decodificando respuesta"
		end
		
		-- Extraer respuesta según proveedor
		local reply
		
		if API_PROVIDER == "groq" then
			if data.choices and data.choices[1] and data.choices[1].message then
				reply = data.choices[1].message.content
			elseif data.error then
				UpdateConsole("Groq Error", data.error.code or "Unknown")
				return nil, data.error.message or "Error de Groq"
			end
		else
			if data.candidates and data.candidates[1] and data.candidates[1].content then
				reply = data.candidates[1].content.parts[1].text
			elseif data.error then
				if data.error.code == 429 or data.error.code == 503 then
					Settings.IsServerBusy = true
					BusyStartTime = tick()
					if not Settings.IsProbing then
						task.spawn(ProbeServer)
					end
					return nil, "Servidor ocupado"
				end
				UpdateConsole("Gemini Error", data.error.code or "Unknown")
				return nil, data.error.message or "Error de Gemini"
			end
		end
		
		if reply then
			reply = reply:gsub("^%s+", ""):gsub("%s+$", "")
			
			if #reply > 200 then
				reply = reply:sub(1, 197) .. "..."
			end
			
			-- Detectar y ejecutar scripts (solo en modo privado)
			if PRIVATE_MODE and reply:match("%[EXECUTE:") then
				local keyword = reply:match("%[EXECUTE:(.-)%]")
				
				if keyword then
					local script, suggestions = FindScriptByKeyword(keyword)
					
					if script then
						local success, execError = SafeExecuteScript(script.url)
						
						if success then
							reply = "✅ " .. script.name .. " ejecutado correctamente!"
						else
							reply = "❌ Error al ejecutar " .. script.name .. ": " .. execError
						end
					elseif suggestions and #suggestions > 0 then
						reply = "🤔 No encontré ese script. ¿Quisiste decir: " .. suggestions[1].keyword .. "?"
					else
						reply = "❌ Script no encontrado. Escribe 'scripts' para ver la lista."
					end
				end
			end
			
			-- Guardar en historial
			table.insert(ChatHistory, {role = "user", parts = {{text = userMessage}}})
			table.insert(ChatHistory, {role = "model", parts = {{text = reply}}})
			
			while #ChatHistory > MAX_HISTORY_LENGTH do
				table.remove(ChatHistory, 1)
			end
			
			SaveChatHistory()
			
			UpdateConsole("Response OK | History: " .. #ChatHistory, "Sent")
			Settings.NextReqTime = tick() + 4
			
			return reply, nil
		else
			UpdateConsole("Empty Response", "No content")
			return nil, "Respuesta vacía"
		end
	else
		UpdateConsole("Network Error", "Connection failed")
		return nil, "Error de conexión"
	end
end
-- Función para manejar mensajes entrantes del chat
local function HandleChatMessage(player, messageText)
	-- Debug: Ver qué mensajes llegan
	print("[Lux DEBUG] Message from:", player.Name, "| Text:", messageText)
	print("[Lux DEBUG] Mode:", PRIVATE_MODE and "PRIVATE" or "PUBLIC")
	print("[Lux DEBUG] AI Enabled:", Settings.AI_Enabled)
	-- Verificar que la IA esté activada
	if not Settings.AI_Enabled or not _G.LuxAPIReady then
		print("[Lux DEBUG] AI not ready, ignoring message")
		return
	end
	-- MODO PRIVADO: Solo el dueño del script puede hablar
	if PRIVATE_MODE and player ~= scriptOwner then
		print("[Lux DEBUG] Private mode - ignoring non-owner message")
		return
	end
	-- MODO PÚBLICO: Ignorar mensajes del dueño (anti-loop)
	if not PRIVATE_MODE and player == scriptOwner then
		print("[Lux DEBUG] Public mode - ignoring owner message (anti-loop)")
		return
	end
	print("[Lux DEBUG] Message passed filters, processing...")
	local userId = player.UserId
	local lowerMessage = messageText:lower()
	local startsWithLux = lowerMessage:match("^" .. BOT_NAME)
	local isInConversation = IsInActiveConversation(userId)
	-- Determinar si debe procesar el mensaje
	local shouldProcess = false
	local userMessage = ""
	if startsWithLux then
		-- Caso 1: Mensaje empieza con "lux"
		shouldProcess = true
		userMessage = messageText:sub(#BOT_NAME + 1):gsub("^%s+", "")
		-- Agregar a conversación activa si no está
		if not isInConversation then
			AddToActiveConversation(userId)
			-- Si el mensaje es solo "lux", dar bienvenida
			if userMessage == "" or #userMessage < 2 then
				SendChatMessage("¡Hola! Ahora estamos en conversación. Ya no necesitas decir 'lux' antes de cada mensaje 😊")
				return
			end
		end
	elseif isInConversation then
		-- Caso 2: Usuario YA está en conversación activa, no necesita decir "lux"
		shouldProcess = true
		userMessage = messageText
		-- Actualizar timestamp de actividad
		AddToActiveConversation(userId) -- Esto actualiza LastMessageTime
	else
		-- Caso 3: No empieza con "lux" y no está en conversación = ignorar
		return
	end
	-- Validar que haya contenido
	if userMessage == "" or #userMessage < 1 then
		return
	end
	-- Comando especial: salir de conversación
	if userMessage:lower() == "adios" or userMessage:lower() == "chao" or userMessage:lower() == "salir" then
		RemoveFromActiveConversation(userId)
		SendChatMessage("¡Hasta luego! Si quieres hablar de nuevo, solo di 'lux' 👋")
		return
	end
	-- Comando especial: limpiar historial
	if userMessage:lower() == "olvida" or userMessage:lower() == "resetear" or userMessage:lower() == "clear" then
		ClearChatHistory()
		SendChatMessage("Historial limpiado. Empecemos de nuevo 🔄")
		return
	end
	-- Comando especial: ver entorno
	if userMessage:lower() == "entorno" or userMessage:lower() == "alrededor" or userMessage:lower() == "quien esta aqui" then
		local summary = GetEnvironmentSummary()
		SendChatMessage(summary)
		return
	end
	-- Comando especial: buscar jugador
	if userMessage:lower():match("^donde esta%s+") or userMessage:lower():match("^ubicacion%s+") then
		local playerName = userMessage:lower():match("^donde esta%s+(.+)") or userMessage:lower():match("^ubicacion%s+(.+)")
		if playerName then
			local playerData = FindPlayerByName(playerName)
			if playerData then
				local msg = string.format(
					"%s está a %d studs de ti (Salud: %d/%d)",
					playerData.name,
					playerData.distanceFromYou,
					playerData.health,
					playerData.maxHealth
				)
				SendChatMessage(msg)
			else
				SendChatMessage("No encontré a '" .. playerName .. "' en el servidor")
			end
		end
		return
	end
	-- Comando especial: jugadores cercanos
	if userMessage:lower() == "cercanos" or userMessage:lower() == "quien cerca" then
		local env = ScanEnvironment()
		if env and #env.nearestPlayers > 0 then
			local msg = "Jugadores cercanos: "
			local names = {}
			for i = 1, math.min(3, #env.nearestPlayers) do
				local p = env.nearestPlayers[i]
				table.insert(names, p.name .. " (" .. p.distanceFromYou .. " studs)")
			end
			SendChatMessage(msg .. table.concat(names, ", "))
		else
			SendChatMessage("No hay jugadores cerca de ti")
		end
		return
	end
	-- Comando especial: ver estado de conversación
	if userMessage:lower() == "estado" or userMessage:lower() == "status" then
		local activeCount = GetActiveConversationsCount()
		SendChatMessage("Conversaciones activas: " .. activeCount .. " | Historial: " .. #ChatHistory .. " mensajes")
		return
	end
	-- Comando especial: ver entorno
	if userMessage:lower() == "entorno" or userMessage:lower() == "alrededor" or userMessage:lower() == "quien esta aqui" then
		local summary = GetEnvironmentSummary()
		SendChatMessage(summary)
		return
	end
	-- Comando especial: buscar jugador
	if userMessage:lower():match("^donde esta%s+") or userMessage:lower():match("^ubicacion%s+") then
		local playerName = userMessage:lower():match("^donde esta%s+(.+)") or userMessage:lower():match("^ubicacion%s+(.+)")
		if playerName then
			local playerData = FindPlayerByName(playerName)
			if playerData then
				local msg = string.format(
					"%s está a %d studs de ti (Salud: %d/%d)",
					playerData.name,
					playerData.distanceFromYou,
					playerData.health,
					playerData.maxHealth
				)
				SendChatMessage(msg)
			else
				SendChatMessage("No encontré a '" .. playerName .. "' en el servidor")
			end
		end
		return
	end
	-- Comando especial: jugadores cercanos
	if userMessage:lower() == "cercanos" or userMessage:lower() == "quien cerca" then
		local env = ScanEnvironment()
		if env and #env.nearestPlayers > 0 then
			local msg = "Jugadores cercanos: "
			local names = {}
			for i = 1, math.min(3, #env.nearestPlayers) do
				local p = env.nearestPlayers[i]
				table.insert(names, p.name .. " (" .. p.distanceFromYou .. " studs)")
			end
			SendChatMessage(msg .. table.concat(names, ", "))
		else
			SendChatMessage("No hay jugadores cerca de ti")
		end
		return
	end
-- Comando especial: listar scripts disponibles (solo en modo privado)
	if PRIVATE_MODE and (userMessage:lower() == "scripts" or userMessage:lower() == "lista") then
		SendChatMessage("📜 Scripts: GRIP, God Mode, Fly, Infinite Yield, Nameless Admin. Di 'ejecuta [nombre]'")
		return
	end
	-- Comando especial: ejecutar script directamente (solo en modo privado)
	if PRIVATE_MODE and userMessage:lower():match("^ejecuta?%s+") then
		local scriptKeyword = userMessage:lower():match("^ejecuta?%s+(.+)")
		if scriptKeyword then
			local script, suggestions = FindScriptByKeyword(scriptKeyword)
			if script then
				local success, execError = SafeExecuteScript(script.url)
				if success then
					SendChatMessage("✅ " .. script.name .. " ejecutado!")
				else
					SendChatMessage("❌ Error: " .. execError)
				end
			elseif suggestions and #suggestions > 0 then
				SendChatMessage("🤔 ¿Quisiste decir: " .. suggestions[1].keyword .. "?")
			else
				SendChatMessage("❌ Script no encontrado. Di 'scripts' para ver la lista.")
			end
		end
		return
	end	
	-- Cooldown entre respuestas
	if tick() - lastChatTime < CHAT_COOLDOWN then
		return
	end
	lastChatTime = tick()
	-- Procesar en background
	task.spawn(function()
		local response, error = AskAI(userMessage, player.DisplayName or player.Name)
		if response then
			SendChatMessage(response)
		elseif error then
			-- Solo mostrar errores importantes
			if error == "Servidor ocupado" then
				SendChatMessage("Un momento, estoy procesando otra petición...")
			elseif error == "API Key no configurada" then
				warn("[Lux] API Key no configurada")
			end
		end
	end)
end
-- Conectar al sistema de chat (TextChatService)
if TextChatService.ChatVersion == Enum.ChatVersion.TextChatService then
	print("[Lux] Using TextChatService (new chat system)")
	TextChatService.MessageReceived:Connect(function(textChatMessage)
		-- Validación básica
		if not textChatMessage.TextSource then return end
		local userId = textChatMessage.TextSource.UserId
		local player = Players:GetPlayerByUserId(userId)
		if not player then return end
		print("[Lux DEBUG] Received message from:", player.Name, "| Mode:", PRIVATE_MODE and "PRIVATE" or "PUBLIC")
		-- Filtro basado en modo
		if PRIVATE_MODE then
			-- Modo privado: solo el dueño
			if player == scriptOwner then
				HandleChatMessage(player, textChatMessage.Text)
			else
				print("[Lux DEBUG] Ignoring non-owner in PRIVATE mode")
			end
		else
			-- Modo público: todos EXCEPTO el dueño (anti-loop)
			if userId ~= scriptOwner.UserId then
				HandleChatMessage(player, textChatMessage.Text)
			else
				print("[Lux DEBUG] Ignoring owner in PUBLIC mode (anti-loop)")
			end
		end
	end)
else
	-- Sistema de chat legacy
	local function ConnectPlayerChat(player)
		if not player then return end
		player.Chatted:Connect(function(message)
			-- En modo PRIVADO: solo el dueño
			if PRIVATE_MODE then
				if player == scriptOwner then
					HandleChatMessage(player, message)
				end
			else
				-- En modo PÚBLICO: todos excepto el dueño
				if player.UserId ~= scriptOwner.UserId then
					HandleChatMessage(player, message)
				end
			end
		end)
	end
	-- Conectar jugadores existentes
	for _, player in pairs(Players:GetPlayers()) do
		if player ~= scriptOwner or PRIVATE_MODE then
			ConnectPlayerChat(player)
		end
	end
	-- Conectar nuevos jugadores
	Players.PlayerAdded:Connect(function(player)
		if player ~= scriptOwner or PRIVATE_MODE then
			ConnectPlayerChat(player)
		end
	end)
end
-- Cargar historial al iniciar
LoadChatHistory()
LoadActiveConversations()
-- Limpieza periódica de conversaciones inactivas (cada 60 segundos)
task.spawn(function()
	while ScreenGui.Parent do
		task.wait(60)
		CleanupInactiveConversations()
	end
end)
print("[Lux] API System loaded")
print("[Lux] Chat system initialized")
print("[Lux] Mode: " .. (PRIVATE_MODE and "PRIVATE (Owner Only)" or "PUBLIC"))
print("[Lux] Conversation system: ACTIVE")
print("[Lux] Say 'lux [message]' to start a conversation")
print("[Lux] Once started, you don't need to say 'lux' anymore")
print("[Lux] Say 'adios' to end conversation | 'lux olvida' to clear history")
print("[Lux] Get your API key: https://aistudio.google.com/app/apikey")
print("[Lux] Now supports both Gemini and Groq APIs!")
