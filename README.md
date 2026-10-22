-- Lucky Trait Server Hopper
-- Busca servidores con el evento/trait "Lucky" activo, avisa por Discord y sigue saltando.
 
local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
 
local unpack = unpack or table.unpack
 
------------------------------------------------------------
-- CONFIG
------------------------------------------------------------
local Config = {
    WebhookUrl = "https://discord.com/api/webhooks/1432387347496042599/HU5TIDm8LlROESAtOpBkpgTEbPJeOtSyNyYZHgQv7naGZru5FxegB00PGnf8-lOUTOgl", -- regenera el tuyo, el anterior quedo expuesto
    PingEveryone = true,
    HopDelay = 5,                 -- segundos maximos en cada servidor antes de saltar
    ScanInterval = 0.5,           -- cada cuanto revisa si hay Lucky
    MaxRememberedServers = 5,     -- servidores recientes que no se repiten
    LuckyKeywords = { "lucky" },  -- palabras a buscar (minusculas)
    -- Nombres de contenedores donde el juego guarda eventos/clima activos.
    -- Ajusta si en tu juego se llaman distinto.
    EventContainers = { "Weather Events", "Luck Event", "Trait Event", "Trait Luck Weather" },
}
 
------------------------------------------------------------
-- ESTADO
------------------------------------------------------------
local State = {
    VisitedServers = {},
    AlreadyNotified = {},
    HeartbeatConn = nil,
    StartTime = tick(),
    Found = false,
}
 
------------------------------------------------------------
-- DETECCION DE LUCKY
------------------------------------------------------------
local function matchesKeyword(text)
    text = string.lower(tostring(text))
    for _, kw in ipairs(Config.LuckyKeywords) do
        if text:find(kw, 1, true) then
            return true
        end
    end
    return false
end
 
-- 1) Atributos activos (true / numero > 0) en workspace o ReplicatedStorage
local function checkAttributes(instance)
    for name, value in pairs(instance:GetAttributes()) do
        if matchesKeyword(name) then
            if value == true or (type(value) == "number" and value > 0) or (type(value) == "string" and value ~= "") then
                return true, "Atributo: " .. name
            end
        end
    end
    return false
end
 
-- 2) Valores (BoolValue/NumberValue/StringValue) con nombre Lucky y valor activo
local function checkValueObjects(root)
    for _, obj in ipairs(root:GetDescendants()) do
        if matchesKeyword(obj.Name) then
            if obj:IsA("BoolValue") and obj.Value then
                return true, "BoolValue: " .. obj:GetFullName()
            elseif obj:IsA("NumberValue") or obj:IsA("IntValue") then
                if obj.Value > 0 then
                    return true, "Valor: " .. obj:GetFullName()
                end
            end
        end
    end
    return false
end
 
-- 3) Textos visibles en la GUI (ej. banner "Lucky Event" o "x2 Luck")
local function checkGui()
    local player = Players.LocalPlayer
    local gui = player and player:FindFirstChild("PlayerGui")
    if not gui then return false end
    for _, obj in ipairs(gui:GetDescendants()) do
        if (obj:IsA("TextLabel") or obj:IsA("TextButton")) and obj.Visible and obj.Text ~= "" then
            if matchesKeyword(obj.Text) then
                return true, "GUI: " .. obj.Text
            end
        end
    end
    return false
end
 
-- 4) Modelos/instancias en workspace con "lucky" en el nombre (clima/evento spawneado)
local function checkWorkspace()
    local found, why = checkAttributes(workspace)
    if found then return true, why end
    for _, container in ipairs(Config.EventContainers) do
        local inst = workspace:FindFirstChild(container, true)
        if inst then
            for _, child in ipairs(inst:GetChildren()) do
                if matchesKeyword(child.Name) then
                    return true, "Workspace: " .. child:GetFullName()
                end
            end
        end
    end
    return false
end
 
local function detectLucky()
    local checks = {
        function() return checkAttributes(ReplicatedStorage) end,
        checkWorkspace,
        function() return checkValueObjects(ReplicatedStorage) end,
        checkGui,
    }
    for _, check in ipairs(checks) do
        local ok, found, why = pcall(check)
        if ok and found then
            return true, why
        end
    end
    return false
end
 
------------------------------------------------------------
-- DISCORD
------------------------------------------------------------
local function sendLuckyEmbed(reason)
    local jobId = game.JobId
    local placeId = game.PlaceId
    local playerCount = string.format("%d/%d", #Players:GetPlayers(), Players.MaxPlayers)
 
    local joinLink = "[Click to Join](https://testing5312.github.io/joiner/?placeId="
        .. placeId .. "&gameInstanceId=" .. jobId .. ")"
    local joinScript = "```game:GetService(\"TeleportService\"):TeleportToPlaceInstance("
        .. placeId .. ",\"" .. jobId .. "\",game.Players.LocalPlayer)```"
 
    local payload = {
        embeds = {{
            title = "🍀 Lucky Trait Encontrado",
            description = "",
            color = 5763719,
            fields = {
                { name = "🍀 Detectado", value = tostring(reason), inline = false },
                { name = "👥 Players", value = playerCount, inline = true },
                { name = "🔗 Join Link", value = joinLink, inline = false },
                { name = "Job ID (Mobile)", value = "`" .. jobId .. "`", inline = false },
                { name = "Job ID (PC)", value = "```" .. jobId .. "```", inline = false },
                { name = "Join Script (PC)", value = joinScript, inline = false },
            },
            timestamp = os.date("!%Y-%m-%dT%H:%M:%S.000Z"),
        }},
    }
    if Config.PingEveryone then
        payload.content = "@everyone"
    end
 
    task.spawn(function()
        pcall(function()
            request({
                Url = Config.WebhookUrl,
                Method = "POST",
                Headers = { ["Content-Type"] = "application/json" },
                Body = HttpService:JSONEncode(payload),
            })
        end)
    end)
end
 
------------------------------------------------------------
-- SERVER HOP
------------------------------------------------------------
local function rememberServer(id)
    State.VisitedServers[id] = true
    local count = 0
    for _ in pairs(State.VisitedServers) do count += 1 end
    if count > Config.MaxRememberedServers then
        for key in pairs(State.VisitedServers) do
            State.VisitedServers[key] = nil
            break
        end
    end
end
 
local function hopServer()
    local placeId = game.PlaceId
 
    for _ = 1, 2 do
        local ok, result = pcall(function()
            local body = game:HttpGet("https://games.roblox.com/v1/games/" .. placeId
                .. "/servers/Public?sortOrder=Asc&limit=100")
            return HttpService:JSONDecode(body)
        end)
 
        if ok and result and result.data then
            local candidates = {}
            for _, server in ipairs(result.data) do
                if server.id
                    and server.id ~= game.JobId
                    and server.playing
                    and server.playing < server.maxPlayers
                    and server.playing >= 1
                    and not State.VisitedServers[server.id] then
                    table.insert(candidates, server)
                end
            end
 
            if #candidates > 0 then
                local pick = candidates[math.random(1, #candidates)]
                rememberServer(pick.id)
                pcall(function()
                    TeleportService:TeleportToPlaceInstance(placeId, pick.id, Players.LocalPlayer)
                end)
                return
            end
        end
        task.wait(0)
    end
 
    pcall(function()
        TeleportService:TeleportToPlaceInstance(placeId, "random", Players.LocalPlayer)
    end)
end
 
------------------------------------------------------------
-- LOOP PRINCIPAL
------------------------------------------------------------
local function scanLoop()
    while true do
        task.wait(Config.ScanInterval)
        if not State.Found then
            local found, reason = detectLucky()
            if found and not State.AlreadyNotified[game.JobId] then
                State.Found = true
                State.AlreadyNotified[game.JobId] = true
                sendLuckyEmbed(reason)
                -- Quita el timer de salto para quedarte en el server con Lucky
                if State.HeartbeatConn then
                    State.HeartbeatConn:Disconnect()
                    State.HeartbeatConn = nil
                end
                -- Si prefieres que siga saltando despues de avisar, descomenta:
                -- task.wait(2); hopServer()
            end
        end
    end
end
 
task.spawn(scanLoop)
 
State.HeartbeatConn = RunService.Heartbeat:Connect(function()
    if tick() - State.StartTime > Config.HopDelay and not State.Found then
        State.HeartbeatConn:Disconnect()
        hopServer()
    end
end)
 
TeleportService.TeleportInitFailed:Connect(function()
    task.wait(0.1)
    hopServer()
end)
 
TeleportService.LocalPlayerTeleported:Connect(function()
    if State.HeartbeatConn then
        State.HeartbeatConn:Disconnect()
    end
end)
 
hopServer()
 
