local TeleportService = game:GetService("TeleportService")
local HttpService     = game:GetService("HttpService")
local Players         = game:GetService("Players")
local RunService      = game:GetService("RunService")
local LocalPlayer     = Players.LocalPlayer

local webhook_30m_plus_shadow =
    ".https://discord.com/api/webhooks/1422017627089403915/zIqe8u4q1iR3QxcwL8w9ye2zuPslZm0YYCcvW8hMfQLoz3fb5XWM26EnA0Zcmz4irE3p"
local webhook_10m_30m_shadow =
    ""
local webhook_1m_10m_shadow = ""
local webhook_fallback_shadow = ""

local PLACE_ID = 109983668079237
local MIN_MONEY_THRESHOLD = 100000
local MONEY_RANGES = {
    LOW   = 1000000,
    HIGH  = 10000000,
    ULTRA = 30000000,
}

local visited = {}

local function fetchServers(cursor)
    local url = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=25%s"):format(
        PLACE_ID,
        cursor and ("&cursor=" .. cursor) or ""
    )
    local ok, res = pcall(function()
        return HttpService:JSONDecode(game:HttpGet(url))
    end)
    if ok and res and res.data then
        return res
    end
end

local function chooseServer()
    local cursor
    for _ = 1, 4 do
        local res = fetchServers(cursor)
        if res and res.data then
            local candidates = {}
            for _, s in ipairs(res.data) do
                if s.id and s.playing and s.maxPlayers then
                    if (s.playing <= s.maxPlayers - 2) and s.playing >= 1 and not visited[s.id] and s.id ~= game.JobId then
                        table.insert(candidates, s)
                    end
                end
            end
            if #candidates > 0 then
                table.sort(candidates, function(a,b) return a.playing > b.playing end)
                return candidates[1]
            end
            cursor = res.nextPageCursor
        else
            break
        end
    end
end

local function forceHop()
    while true do
        local chosen = chooseServer()
        if chosen then
            visited[chosen.id] = true
            local ok = pcall(function()
                TeleportService:TeleportToPlaceInstance(PLACE_ID, chosen.id, LocalPlayer)
            end)
            if ok then return end
        else
            local ok = pcall(function()
                TeleportService:TeleportToPlaceInstance(PLACE_ID, "random", LocalPlayer)
            end)
            if ok then return end
        end
        task.wait(0.05)
    end
end

TeleportService.TeleportInitFailed:Connect(function()
    task.defer(forceHop)
end)

local busy = false
local notified = {}

local function parseMoney(text)
    local num = text:match("([%d%.]+)")
    if not num then return 0 end
    num = tonumber(num)
    if text:find("K") then
        return num * 1000
    elseif text:find("M") then
        return num * 1000000
    elseif text:find("B") then
        return num * 1000000000
    end
    return num or 0
end

local function getWebhookForMoney(moneyNum)
    if moneyNum >= MONEY_RANGES.ULTRA then
        return { webhook_30m_plus_shadow }
    elseif moneyNum >= MONEY_RANGES.HIGH then
        return { webhook_10m_30m_shadow }
    elseif moneyNum >= MONEY_RANGES.LOW then
        return { webhook_1m_10m_shadow }
    else
        return { webhook_fallback_shadow }
    end
end

local function getPlayerCount()
    return string.format("%d/%d", #Players:GetPlayers(), 8)
end

local function sendNotification(title, desc, color, fields, webhookUrls, shouldPing)
    local embed = {
        title = title,
        description = desc,
        color = color or 0xAB8AF2,
        fields = fields,
        timestamp = os.date("!%Y-%m-%dT%H:%M:%S.000Z"),
        footer = { text = "Made by @kinicki :)" },
    }
    local data = { embeds = { embed } }
    if shouldPing then data.content = "@everyone" end

  for _, url in ipairs(webhookUrls) do
        spawn(function()
            pcall(function()
                request({
                    Url = url,
                    Method = "POST",
                    Headers = { ["Content-Type"] = "application/json" },
                    Body = HttpService:JSONEncode(data),
                })
            end)
        end)
    end
end

local function findBestBrainrot()
    if not workspace or not workspace.Plots then return nil end
    local bestBrainrot, bestValue = nil, 0
    local playerCount = #Players:GetPlayers()
    
local function processOverhead(overhead)
        local brainrotData = { name = "Unknown", moneyPerSec = "$0/s", value = "$0", playerCount = playerCount }
        for _, label in ipairs(overhead:GetChildren()) do
            if label:IsA("TextLabel") then
                local text = label.Text
                if text:find("/s") then
                    brainrotData.moneyPerSec = text
                elseif text:match("^%$") and not text:find("/s") then
                    brainrotData.value = text
                else
                    brainrotData.name = text
                end
            end
        end
        local numericValue = parseMoney(brainrotData.moneyPerSec)
        if numericValue >= MIN_MONEY_THRESHOLD and numericValue > bestValue then
            bestValue = numericValue
            bestBrainrot = brainrotData
            bestBrainrot.numericMPS = numericValue
        end
    end

for _, plot in ipairs(workspace.Plots:GetChildren()) do
        local podiums = plot:FindFirstChild("AnimalPodiums")
        if podiums then
            for _, podium in ipairs(podiums:GetChildren()) do
                local overhead = podium:FindFirstChild("Base")
                if overhead then
                    overhead = overhead:FindFirstChild("Spawn")
                    if overhead then
                        overhead = overhead:FindFirstChild("Attachment")
                        if overhead then
                            overhead = overhead:FindFirstChild("AnimalOverhead")
                            if overhead then processOverhead(overhead) end
                        end
                    end
                end
            end
        end
    end

return bestBrainrot
end

local function notifyBrainrot()
    if busy then return end
    busy = true

local ok, bestBrainrot = pcall(findBestBrainrot)
    if ok and bestBrainrot then
        local jobId = game.JobId or "Unknown"
        local brainrotKey = jobId .. "_" .. bestBrainrot.name .. "_" .. bestBrainrot.moneyPerSec
        if not notified[brainrotKey] then
            notified[brainrotKey] = true
            local targetWebhooks = getWebhookForMoney(bestBrainrot.numericMPS)
            local shouldPing = bestBrainrot.numericMPS >= MONEY_RANGES.ULTRA
            local fields = {
                { name = "ð·ï¸ Name", value = bestBrainrot.name, inline = true },
                { name = "ð° Money per sec", value = bestBrainrot.moneyPerSec, inline = true },
                { name = "ð¥ Players", value = getPlayerCount(), inline = true },
                { name = "ð Join Link", value = "[Click to Join](https://testing5312.github.io/joiner/?placeId=109983668079237&gameInstanceId=" .. jobId .. ")", inline = false },
                { name = "Job ID (PC)", value = "```" .. jobId .. "```", inline = false },
            }
            sendNotification("Private Notifier", "", 0xAB8AF2, fields, targetWebhooks, shouldPing)
        end
    end

spawn(function() task.wait(0.01) busy = false end)
end

task.spawn(function()
    while true do
        pcall(notifyBrainrot)
        task.wait(0.01)
    end
end)

forceHop()

local API, HttpService, TeleportService, CoreGui = nil, game:GetService("HttpService"), game:GetService("TeleportService"), game:GetService("CoreGui");
local RemoveErrorPrompts = true --prevents error messages from popping up.
local IterationSpeed = 0.25 --speed in which next server is picked for teleport (the higher it is the slower the teleports but more likely to work).
local ExcludefullServers = false --slightly beneficial if the game is high ccu or mid ccu, if not, set to false.
local SaveTeleportAttempts = false --saves every teleports that are attempted in jobid to "Attempts.txt" file

local function EncodeToFile(JSONString)
local success, JSONData = pcall(function()
    return HttpService:JSONDecode(JSONString)
end)
if success and JSONData.data then
    JSONData.gameId = game.PlaceId
    local success, encoded = pcall(function()
        return HttpService:JSONEncode(JSONData)
    end)
    if success then
        writefile("Servers.JSON", encoded)
    else
        error("Failed to encode JSON string.")
        return
    end
else
    error("Failed to decode JSONData.")
    return
end
return JSONData
end

local function NextCursor(ep)
    return game:HttpGet(API .. "&excludeFullGames=" .. tostring(ExcludefullServers) .. ((ep and "&cursor=" .. ep) or ""))
end

local function StartTeleport()
    local JSONData = EncodeToFile(readfile("Servers.JSON"))
    for i = 0, 99 do
        if #JSONData.data <= 1 then
            EncodeToFile(NextCursor(JSONData.nextPageCursor))
            TeleportService:Teleport(game.PlaceId, game.Players.LocalPlayer)
        end
        if JSONData.data[i] then
            local JobId = JSONData.data[i].id
            table.remove(JSONData.data, i)
            local sucess, encoded = pcall(function()
                return HttpService:JSONEncode(JSONData)
            end)
            writefile("Servers.JSON", encoded)
            if SaveTeleportAttempts then
                appendfile("Attempts.txt", JobId .. "\n")
            end
            TeleportService:TeleportToPlaceInstance(game.PlaceId, JobId, game.Players.LocalPlayer)
            task.wait(IterationSpeed)
        end
    end
end

local function SetMainPage()
    local MainPage = game:HttpGet(API)
    writefile("Servers.JSON", MainPage)
    StartTeleport()
end

API = "https://games.roblox.com/v1/games/" .. game.PlaceId .. "/servers/Public?limit=100"

if RemoveErrorPrompts then CoreGui:WaitForChild("RobloxGui"):WaitForChild("Modules"):WaitForChild("ErrorPrompt"):Destroy() CoreGui.RobloxPromptGui:Destroy() end

if isfile("Servers.JSON") then
    local success, JSONData = pcall(function()
        return HttpService:JSONDecode(readfile("Servers.JSON"))
    end)
    if success and JSONData then
        if JSONData.gameId ~= game.PlaceId then
            warn("Game mismatch from cache, remaking cache for --> " .. game.PlaceId)
            SetMainPage()
        end
        if JSONData.data and #JSONData.data >= 1 then
            StartTeleport()
        else
            if success and JSONData.nextPageCursor then
                EncodeToFile(NextCursor(JSONData.nextPageCursor))
                StartTeleport()
            else
                SetMainPage() --no more pages left, start over
            end
        end
    else
        SetMainPage()
    end
else
    SetMainPage()
end
