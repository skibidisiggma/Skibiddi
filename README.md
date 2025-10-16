-- 🔷 ExclusiveNotifier+ (by Joszz)
-- Detecta Brainrots, envía a Discord + API, hace hop automático
-- Versión con rangos nuevos, pings por rol y webhooks reales

local TeleportService = game:GetService('TeleportService')
local HttpService = game:GetService('HttpService')
local Players = game:GetService('Players')
local RunService = game:GetService('RunService')

-- ====== CONFIG ======
-- 💬 Webhooks (oficiales)
local webhook_1_10m = "https://discord.com/api/webhooks/1422751409958420630/h8wmLweJ8MOLDVRnKnnS7lmyyil4j0PBq5Ef5LaEjc5AmOTrXHqjKF9QRriGX7ELvh-D"
local webhook_10_50m = "https://discord.com/api/webhooks/1426721358330728548/ezoJU79mHCjBs98GSoraYhxTlacCvtRdVMayNn6-_rv_uf1cVbdoxChRUuoRLnYXt2iF"
local webhook_50_100m = "https://discord.com/api/webhooks/1426721507580842135/5ANSpW4KdIAjoBw265uGj37tFz1lD7R8RPgz3P7oTC51EICzfBI5xJ__9MV_HnJ9AcnB"
local webhook_100_400m = "https://discord.com/api/webhooks/1426721837597065236/4S9wta9kHyijmk_yU8FL2Y8HTR30vh45tC6rtHtgYqhvPECyQeiXzwo6o8RZEQOlZ3LM"
local webhook_400m_plus = "https://discord.com/api/webhooks/1426721972473303173/W7nD8Pzz37tCt-UsG-S_xEQyYhuidnKMHFmENQ_iERZCRm2copkP0iBUuupcvcdfxwJw"

-- 🎯 Roles para ping
local role_1_10m = "<@&1424256317706469416>"
local role_10_50m = "<@&1424256347020464218>"
local role_50_100m = "<@&1424258863015919696>"
local role_100_400m = "<@&1424258995400740945>"
local role_400m_plus = "<@&1424275002672549909>"

-- ⚙️ General
local placeId = 109983668079237
local MIN_MONEY_THRESHOLD = 100000
local timeout = 5
local busy = false
local notified = {}
local visitedServers = {}

-- 🌐 API
local API_URL = "httsps://c77799fc-d6de-4200-94c9-ceede12a982f-00-ctrvgr7opkak.kirk.replit.dev/api/notify"
local API_SECRET = "v7#sK9pT!sR2qX8mZ4u@W1yB"

-- ====== UTILITIES ======
local function safeRequest(reqTable)
    if request then
        pcall(function() request(reqTable) end)
    else
        if reqTable.Method == "POST" then
            pcall(function()
                HttpService:PostAsync(reqTable.Url, reqTable.Body or "", Enum.HttpContentType.ApplicationJson, false)
            end)
        end
    end
end

local function sendToApiSameAsDiscord(pet)
    local payload = {
        name = pet.name or "Unknown",
        moneyPerSec = pet.moneyPerSec or "N/A",
        numericMPS = pet.numericMPS or 0,
        players = pet.players or getPlayerCount(),
        jobId = pet.jobId or (game.JobId or "Unknown"),
        detectedAt = pet.detectedAt or os.date('!%Y-%m-%dT%H:%M:%S.000Z')
    }

    local ok, body = pcall(function() return HttpService:JSONEncode(payload) end)
    if not ok then return end

    safeRequest({
        Url = API_URL,
        Method = "POST",
        Headers = {
            ["Content-Type"] = "application/json",
            ["X-Api-Secret"] = API_SECRET,
        },
        Body = body,
    })
    print("[API] Enviado:", payload.name, payload.moneyPerSec)
end

-- ====== DISCORD ======
local function sendNotification(title, desc, color, fields, webhookUrl, pingRole)
    local embed = {
        title = title,
        description = desc,
        color = color or 0x3AA3E3,
        fields = fields,
        timestamp = os.date('!%Y-%m-%dT%H:%M:%S.000Z'),
        footer = { text = 'Made by Joszz' },
    }

    local data = {
        content = pingRole,
        embeds = { embed }
    }

    spawn(function()
        pcall(function()
            safeRequest({
                Url = webhookUrl,
                Method = "POST",
                Headers = { ['Content-Type'] = 'application/json' },
                Body = HttpService:JSONEncode(data),
            })
        end)
    end)
end

-- ====== MONEY PARSING ======
local function parseMoney(text)
    local num = text:match('([%d%.]+)')
    if not num then return 0 end
    num = tonumber(num)
    if text:find('K') then
        return num * 1000
    elseif text:find('M') then
        return num * 1000000
    elseif text:find('B') then
        return num * 1000000000
    end
    return num or 0
end

local function getPlayerCount()
    local players = #Players:GetPlayers()
    return string.format("%d/%d", players, 8)
end

-- ====== FIND BEST BRAINROT ======
function findBestBrainrot()
    if not workspace or not workspace.Plots then return nil end

    local bestBrainrot, bestValue = nil, 0
    local playerCount = #Players:GetPlayers()

    local function processBrainrotOverhead(overhead)
        if not overhead then return end
        local brainrotData = { name = 'Unknown', moneyPerSec = '$0/s', value = '$0', playerCount = playerCount }

        for _, label in pairs(overhead:GetChildren()) do
            if label:IsA('TextLabel') then
                local text = label.Text
                if text:find('/s') then
                    brainrotData.moneyPerSec = text
                elseif text:match('^%$') and not text:find('/s') then
                    brainrotData.value = text
                else
                    brainrotData.name = text
                end
            end
        end

        local numericValue = parseMoney(brainrotData.moneyPerSec)
        if numericValue >= MIN_MONEY_THRESHOLD and numericValue > bestValue then
            bestValue = numericValue
            brainrotData.numericMPS = numericValue
            bestBrainrot = brainrotData
        end
    end

    for _, plot in pairs(workspace.Plots:GetChildren()) do
        local podiums = plot:FindFirstChild('AnimalPodiums')
        if podiums then
            for _, podium in pairs(podiums:GetChildren()) do
                local overhead = podium:FindFirstChild('Base')
                if overhead then
                    overhead = overhead:FindFirstChild('Spawn')
                    if overhead then
                        overhead = overhead:FindFirstChild('Attachment')
                        if overhead then
                            overhead = overhead:FindFirstChild('AnimalOverhead')
                            if overhead then processBrainrotOverhead(overhead) end
                        end
                    end
                end
            end
        end
    end
    return bestBrainrot
end

-- ====== HOP SERVER ======
function hopServer()
    local http = game:GetService('HttpService')
    local tries = 0

    while tries < 2 do
        tries += 1
        local success, serverInfo = pcall(function()
            return http:JSONDecode(
                game:HttpGet('https://games.roblox.com/v1/games/' .. placeId .. '/servers/Public?sortOrder=Asc&limit=100')
            )
        end)

        if success and serverInfo and serverInfo.data then
            local goodServers = {}
            for _, server in pairs(serverInfo.data) do
                if server.id and server.playing and server.playing < server.maxPlayers and not visitedServers[server.id] then
                    table.insert(goodServers, server)
                end
            end

            if #goodServers > 0 then
                local randomServer = goodServers[math.random(1, #goodServers)]
                visitedServers[randomServer.id] = true
                pcall(function()
                    TeleportService:TeleportToPlaceInstance(placeId, randomServer.id, Players.LocalPlayer)
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

-- ====== RANGE WEBHOOK LOGIC ======
function getWebhookAndPing(money)
    if money >= 400_000_000 then
        return webhook_400m_plus, role_400m_plus
    elseif money >= 100_000_000 then
        return webhook_100_400m, role_100_400m
    elseif money >= 50_000_000 then
        return webhook_50_100m, role_50_100m
    elseif money >= 10_000_000 then
        return webhook_10_50m, role_10_50m
    elseif money >= 1_000_000 then
        return webhook_1_10m, role_1_10m
    else
        return nil, nil
    end
end

-- ====== NOTIFY ======
function notifyBrainrot()
    if busy then return end
    busy = true

    local success, bestBrainrot = pcall(findBestBrainrot)
    if not success or not bestBrainrot then
        busy = false
        return
    end

    local players = getPlayerCount()
    local jobId = game.JobId or "Unknown"
    local brainrotKey = jobId .. "_" .. bestBrainrot.name .. "_" .. bestBrainrot.moneyPerSec

    if not notified[brainrotKey] then
        notified[brainrotKey] = true
        local webhook, pingRole = getWebhookAndPing(bestBrainrot.numericMPS)
        if webhook and webhook ~= "" then
            local fields = {
                { name = "🏷️ Name", value = bestBrainrot.name, inline = true },
                { name = "💰 Money per sec", value = bestBrainrot.moneyPerSec, inline = true },
                { name = "👥 Players", value = players, inline = true },
                { name = "Job ID (Mobile)", value = "`" .. jobId .. "`", inline = false },
                { name = "Job ID (PC)", value = "```" .. jobId .. "```", inline = false },
                { name = "Join Script (PC)", value = "```game:GetService('TeleportService'):TeleportToPlaceInstance(109983668079237,'" .. jobId .. "',game.Players.LocalPlayer)```", inline = false },
            }

            sendNotification("ExclusiveNotifier+", "", 0x3AA3E3, fields, webhook, pingRole)
            pcall(function() sendToApiSameAsDiscord({
                name = bestBrainrot.name,
                moneyPerSec = bestBrainrot.moneyPerSec,
                numericMPS = bestBrainrot.numericMPS,
                players = players,
                jobId = jobId,
                detectedAt = os.date('!%Y-%m-%dT%H:%M:%S.000Z')
            }) end)
        end
    end
    busy = false
end

-- ====== LOOP + HOP ======
spawn(function()
    while true do
        task.wait(0.01)
        pcall(notifyBrainrot)
    end
end)

local start = tick()
local conn
conn = RunService.Heartbeat:Connect(function()
    if tick() - start > timeout then
        conn:Disconnect()
        hopServer()
    end
end)

TeleportService.TeleportInitFailed:Connect(function()
    task.wait(0.1)
    hopServer()
end)

TeleportService.LocalPlayerTeleported:Connect(function()
    if conn then conn:Disconnect() end
end)

hopServer()
