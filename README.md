--[[ ⚔️ Guild: Nyxara Exploits Hub ]]
-- Full Pet Sniper + ESP + Webhook A + Webhook B + Auto Server Hop + Auto Restart + 👥 Players

--// 🎯 Targeted Pet Configuration
getgenv().WebhookATargets = {
    "Chicleteira Bicicleteira",
    "Dragon Cannelloni",
    "La Grande Combinasion",
    "Garama and Madundung",
    "Nuclearo Dinossauro",
    "Los Combinasionas",
    "Esok Sekolah",
    "Los Hotspotitos",
    "Pot Hostpot"
}

getgenv().WebhookBTargets = {
    "La Vacca Saturno Saturnita",
    "Chimpanzini Spiderini",
    "Los Tralaleritos",
    "Tortuginni Dragonfrutini",
    "Las Vaquitas Saturnitas",
    "Graipuss Medussi",
    "Las Tralaleritas"
}

--// 🌐 Webhooks
local webhookA = "https://discord.com/api/webhooks/1402480794643075213/_GbDViIjHFUXCcSQiojz324x9l0DIY2ubeMzZDRTcI9g2p1M1F_yoE7vJX_XOv4Xh4K1"
local webhookB = "https://discord.com/api/webhooks/1404225086940250203/VJzWDW_vkq6iid9CyrPfZeUYOHt2u9d12YaOuE_cJ6VieoHMd7kb5N4PdZqvxgz85VuU"

--// 🔧 Services
local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local LocalPlayer = Players.LocalPlayer

--// 📦 State
local visitedJobIds = {[game.JobId] = true}
local hops = 0
local maxHopsBeforeReset = 1000
local maxTeleportRetries = 10

--// AnimalOverHead Module (integrado)
local AnimalOverHead = {}

AnimalOverHead.pets = {
    ["Los Combinasionas"] = {
        brainrot_base = 10,
        steal_multiplier = 2,
        value_per_brainrot = 5
    },
    ["OtherPet"] = {
        brainrot_base = 8,
        steal_multiplier = 1.5,
        value_per_brainrot = 4
    }
}

local function calc_brainrot(petData, luck)
    local brainrot = petData.brainrot_base * petData.steal_multiplier
    brainrot = brainrot * (1 + (luck or 0) * 0.01)
    return brainrot
end

function AnimalOverHead.moneyPerSecond(petName, luck)
    local pet = AnimalOverHead.pets[petName]
    if not pet then return 0 end
    local brainrot = calc_brainrot(pet, luck)
    return brainrot * pet.value_per_brainrot
end

function AnimalOverHead.totalMoneyPerSecond(petList, luck)
    local total = 0
    for _, name in ipairs(petList) do
        total = total + AnimalOverHead.moneyPerSecond(name, luck)
    end
    return total
end

--// 🔍 Función para leer dinero por segundo usando AnimalOverHead
local function getMoneyPerSecond(petModel)
    -- Ya no usamos dinero, así que simplemente devolvemos "N/A"
    return "N/A"
end

--// Función para obtener traits del pet
local function getTraits(petModel)
    local traits = {}

    for _, child in ipairs(petModel:GetDescendants()) do
        if (child:IsA("StringValue") or child:IsA("IntValue") or child:IsA("NumberValue")) then
            local nameLower = string.lower(child.Name)
            if string.find(nameLower, "trait") or string.find(nameLower, "mutation") or string.find(nameLower, "effect") then
                table.insert(traits, tostring(child.Value))
            end
        end
    end

    for attrName, attrValue in pairs(petModel:GetAttributes()) do
        local nameLower = string.lower(attrName)
        if string.find(nameLower, "trait") or string.find(nameLower, "mutation") or string.find(nameLower, "effect") then
            table.insert(traits, tostring(attrValue))
        end
    end

    if #traits == 0 then
        return "None"
    else
        return table.concat(traits, ", ")
    end
end

--// 👁️ ESP
local function addESP(targetModel)
    if targetModel:FindFirstChild("PetESP") then return end
    local Billboard = Instance.new("BillboardGui")
    Billboard.Name = "PetESP"
    Billboard.Adornee = targetModel
    Billboard.Size = UDim2.new(0, 100, 0, 30)
    Billboard.StudsOffset = Vector3.new(0, 3, 0)
    Billboard.AlwaysOnTop = true
    Billboard.Parent = targetModel

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(1, 0, 1, 0)
    Label.BackgroundTransparency = 1
    Label.Text = "🎯 Target Pet"
    Label.TextColor3 = Color3.fromRGB(255, 0, 0)
    Label.TextStrokeTransparency = 0.5
    Label.Font = Enum.Font.SourceSansBold
    Label.TextScaled = true
    Label.Parent = Billboard
end

--// 📩 Webhook Sender
local function sendWebhook(foundPets, jobId, url)
    local formattedPets = {}
    for _, pet in ipairs(foundPets) do
        local traits = getTraits(pet)
        table.insert(formattedPets, pet.Name .. "\n⭐️ Traits: " .. traits)
    end

    local joinerUrl = "https://testing5312.github.io/joiner/?placeId=" .. tostring(game.PlaceId) .. "&gameInstanceId=" .. jobId
    local jsonData = HttpService:JSONEncode({
        ["content"] = url == webhookA and "@everyone" or "",
        ["embeds"] = { {
            ["title"] = "Shadow Notifier⭐️",
            ["description"] = "Sniped Brainrot in server",
            ["fields"] = {
                {["name"] = "User", ["value"] = LocalPlayer.Name},
                {["name"] = "Found Pet(s)", ["value"] = table.concat(formattedPets, "\n\n")},
                {["name"] = "👥 Players", ["value"] = tostring(#Players:GetPlayers()) .. "/" .. tostring(Players.MaxPlayers)},
                {["name"] = "Server JobId", ["value"] = jobId},
                {["name"] = "🌐 Join Server", ["value"] = "[Click here](" .. joinerUrl .. ")"},
                {["name"] = "Time", ["value"] = os.date("%Y-%m-%d %H:%M:%S")}
            },
            ["color"] = 0x800080
        } }
    })

    local req = http_request or request or (syn and syn.request)
    if req then
        pcall(function()
            req({
                Url = url,
                Method = "POST",
                Headers = {["Content-Type"] = "application/json"},
                Body = jsonData
            })
        end)
    end
end

--// 🔍 Pet Detection
local function checkForPets()
    local foundA, foundB = {}, {}

    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("Model") and not obj:FindFirstChild("PetESP") then
            local nameLower = string.lower(obj.Name)

            for _, target in pairs(getgenv().WebhookATargets) do
                if string.find(nameLower, string.lower(target)) then
                    addESP(obj)
                    table.insert(foundA, obj)
                end
            end

            for _, target in pairs(getgenv().WebhookBTargets) do
                if string.find(nameLower, string.lower(target)) then
                    addESP(obj)
                    table.insert(foundB, obj)
                end
            end
        end
    end

    return foundA, foundB
end

--// 🌍 Server Hop (rápido) con filtro para evitar servidores concurridos y delay aleatorio
local function serverHop()
    hops += 1
    if hops >= maxHopsBeforeReset then
        visitedJobIds = {[game.JobId] = true}
        hops = 0
    end

    local tries, cursor = 0, nil
    while tries < maxTeleportRetries do
        local url = "https://games.roblox.com/v1/games/" .. game.PlaceId .. "/servers/Public?sortOrder=Asc&limit=100"
        if cursor then url = url .. "&cursor=" .. cursor end

        local success, response = pcall(function()
            return HttpService:JSONDecode(game:HttpGet(url))
        end)

        if success and response and response.data then
            local servers = {}
            for _, server in ipairs(response.data) do
                -- Solo servidores con menos de 2 jugadores (0 o 1)
                if server.playing < 2 and not visitedJobIds[server.id] and server.id ~= game.JobId then
                    table.insert(servers, server.id)
                end
            end

            if #servers > 0 then
                local picked = servers[math.random(1, #servers)]
                visitedJobIds[picked] = true

                task.wait(math.random(1, 3)) -- Espera random de 1 a 3 segundos para evitar concurrencia

                TeleportService:TeleportToPlaceInstance(game.PlaceId, picked)
                return
            end

            cursor = response.nextPageCursor
            if not cursor then
                tries += 1
                task.wait(0.05)
            end
        else
            tries += 1
            task.wait(0.05)
        end
    end

    TeleportService:Teleport(game.PlaceId)
end

--// ♻️ Sniping Loop (más rápido)
local function startSniper()
    local foundA, foundB = checkForPets()

    if #foundA > 0 then
        sendWebhook(foundA, game.JobId, webhookA)
        task.wait(2)
        serverHop()
    elseif #foundB > 0 then
        sendWebhook(foundB, game.JobId, webhookB)
        task.wait(2)
        serverHop()
    else
        serverHop()
    end

    task.delay(0, startSniper)
end

--// 🚀 Start
task.delay(0.1, startSniper)
