local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local RS = game:GetService("ReplicatedStorage")

pcall(function()
    local vim = game:GetService("VirtualInputManager")
    vim:SendKeyEvent(true, Enum.KeyCode.F9, false, game)
    task.wait()
    vim:SendKeyEvent(false, Enum.KeyCode.F9, false, game)
end)

-- ==================== MULTI-INSTÂNCIA COM JOB ID DIFERENTE ====================
local ACCOUNT_SLOT = 1        -- <<< MUDE PARA 2, 3, 4, 5... EM CADA ROBLOX !!!
local DELAY_OFFSET = (ACCOUNT_SLOT - 1) * 9

print("==============================================")
print("Players: " .. tostring(#Players:GetPlayers()))
print("[MULTI] Slot: " .. ACCOUNT_SLOT .. " | Delay: " .. DELAY_OFFSET .. "s")
print("[JOB ID] Atual: " .. tostring(game.JobId))
print("==============================================")

-- WEBHOOKS
local WEBHOOK_BRAINROTS = "https://discord.com/api/webhooks/1493358563048034412/kk_wyTs7WdBDLF1xiHyPwuQo9Vv15Ss9cSSaP6TjHygW-ffiASHnZg7ZfxvjFsB9o4NY"
local WEBHOOK_MID = "https://discord.com/api/webhooks/1493387992323063828/O3QODFcpg9mpLujlcJJj9kAUrfBRfgSISA0N0fivDJgC0PIFzNWliErkWQPGsnctDm7x"
local WEBHOOK_HIGH = "https://discord.com/api/webhooks/1493358240795332730/2BSz6yuGDV38-PwEhTRacVDAUAmc4RUraRXA6pWROKM82Qk06YIudn3zZcsnsf4E4Umu"
local WEBHOOK_ULTRA = "https://discord.com/api/webhooks/1493388375955083355/SlAQGYkYb7vIV6wX7gS_FWmr9-CbjnWK9SCy93jFgkZM-nU4yE6iEfX-HUGGFS4mZun6"

-- FIREBASE
local FIREBASE_URL = "https://notify-f6091-default-rtdb.firebaseio.com/"

local TARGET_NAMES = {""}
local BLACK_NAMES = {"Fishboard", ""}
local MIN_VALUE = 1e6
local NOTIFIER_NAME = "canelloni notify"

local CURRENT_JOB = tostring(game.JobId)
local PLACE_ID = game.PlaceId
local MAX_ENTRIES = 15

local http_request = (syn and syn.request) or (http and http.request) or http_request or request

local Synchronizer = nil
local AnimalsData = nil
local AnimalsShared = nil

do
    local function tryLoad(parent, name)
        local ok, result = pcall(function() return require(parent:WaitForChild(name, 5)) end)
        return ok and result or nil
    end
    local ok_pkg, Packages = pcall(function() return RS:WaitForChild("Packages", 5) end)
    local ok_dat, Datas = pcall(function() return RS:WaitForChild("Datas", 5) end)
    local ok_sh, Shared = pcall(function() return RS:WaitForChild("Shared", 5) end)
    if ok_pkg and Packages then Synchronizer = tryLoad(Packages, "Synchronizer") end
    if ok_dat and Datas then AnimalsData = tryLoad(Datas, "Animals") end
    if ok_sh and Shared then AnimalsShared = tryLoad(Shared, "Animals") end
end

-- Funções básicas
local function parseValue(text)
    if not text then return 0 end
    local clean = tostring(text):gsub("[%$%s%,%/s%/m%/h]", "")
    local numStr, suf = clean:match("([%d%.]+)(%a?)")
    if not numStr then return 0 end
    local n = tonumber(numStr) or 0
    local mult = ({K=1e3, M=1e6, B=1e9, T=1e12})[(suf or ""):upper()] or 1
    return n * mult
end

local function formatValue(n)
    if n >= 1e12 then return string.format("%.1fT", n/1e12) end
    if n >= 1e9 then return string.format("%.1fB", n/1e9) end
    if n >= 1e6 then return string.format("%.1fM", n/1e6) end
    if n >= 1e3 then return string.format("%.1fK", n/1e3) end
    return tostring(math.floor(n))
end

-- Função para enviar dados ao Firebase
local function sendToFirebase(data)
    if not FIREBASE_URL or FIREBASE_URL == "" then return end
    
    local timestamp = os.time()
    local uniqueId = tostring(timestamp) .. "_" .. tostring(math.random(10000, 99999))
    
    local firebaseData = {
        jobId = data.jobId or CURRENT_JOB,
        name = data.name or "Unknown",
        value = data.value or 0,
        formattedValue = data.formattedValue or "0",
        owner = data.owner or "?",
        timestamp = timestamp,
        date = os.date("%Y-%m-%d %H:%M:%S"),
        slot = ACCOUNT_SLOT,
        placeId = PLACE_ID
    }
    
    local url = FIREBASE_URL .. "scans/" .. uniqueId .. ".json"
    
    pcall(function()
        local response = http_request({
            Url = url,
            Method = "PUT",
            Headers = {["Content-Type"] = "application/json"},
            Body = HttpService:JSONEncode(firebaseData)
        })
        
        if response and response.StatusCode == 200 then
            print("[Firebase] ✅ Dados enviados com sucesso! ID: " .. uniqueId)
        else
            print("[Firebase] ❌ Erro ao enviar dados. Status: " .. tostring(response and response.StatusCode or "unknown"))
        end
    end)
end

local IGNORE_PATTERNS = {"fuse","fusing","craft","upgrade","evolve","fundir","duel","machine","duelos","duels"}

local function isIgnoredText(text)
    if not text then return true end
    if text:match("%d+:%d+") or text:match("%d+s") or text:match("%d+m") then return true end
    local low = text:lower()
    for _, p in ipairs(IGNORE_PATTERNS) do if low:find(p) then return true end end
    return false
end

local function isBlacklisted(itemName)
    local low = itemName:lower()
    for _, b in ipairs(BLACK_NAMES) do
        if b ~= "" and low:find(b:lower()) then return true end
    end
    return false
end

local function isTargeted(itemName)
    local low = itemName:lower()
    for _, t in ipairs(TARGET_NAMES) do
        if t ~= "" and low:find(t:lower()) then return true end
    end
    return false
end

local function resolveOwnerName(owner)
    if not owner then return nil end
    if typeof(owner) == "Instance" and owner:IsA("Player") then return owner.Name end
    if typeof(owner) == "table" then
        if owner.Name and owner.Name ~= "" then return owner.Name end
        if owner.UserId then
            for _, p in ipairs(Players:GetPlayers()) do
                if p.UserId == owner.UserId then return p.Name end
            end
            return tostring(owner.UserId)
        end
    end
    if typeof(owner) == "Instance" then return owner.Name end
    return nil
end

local function scan()
    local items = {}
    local bestRejectedName = nil
    local bestRejectedValue = -1
    local function tryInsert(itemName, val, ownerName)
        if isBlacklisted(itemName) then return end
        if val >= MIN_VALUE or isTargeted(itemName) then
            table.insert(items, {name = itemName, value = val, owner = ownerName or "?"})
        elseif val > bestRejectedValue then
            bestRejectedValue = val
            bestRejectedName = itemName
        end
    end
    if Synchronizer then
        local plots = Workspace:FindFirstChild("Plots")
        if plots then
            for _, plot in ipairs(plots:GetChildren()) do
                pcall(function()
                    local channel = Synchronizer:Get(plot.Name)
                    if not channel then return end
                    local animalList = channel:Get("AnimalList")
                    if not animalList then return end
                    local ownerName = resolveOwnerName(channel:Get("Owner")) or "?"
                    for _, animalData in pairs(animalList) do
                        if type(animalData) ~= "table" then continue end
                        local animalIndex = animalData.Index
                        if not animalIndex then continue end
                        local itemName = animalIndex
                        if AnimalsData then
                            local info = AnimalsData[animalIndex]
                            if info and info.DisplayName then itemName = info.DisplayName end
                        end
                        local val = 0
                        if AnimalsShared then
                            local okG, gen = pcall(function() return AnimalsShared:GetGeneration(animalIndex, animalData.Mutation, animalData.Traits, nil) end)
                            if okG and type(gen) == "number" then val = gen end
                        end
                        tryInsert(itemName, val, ownerName)
                    end
                end)
            end
        end
    end
    if #items == 0 then return nil end
    table.sort(items, function(a, b) return a.value > b.value end)
    local best = items[1]
    local bestOwner = best.owner
    local listText = ""
    local listCount = 0
    for _, it in ipairs(items) do
        if it.owner == bestOwner and listCount < 5 then
            listText = listText .. "1x " .. it.name .. " ($" .. formatValue(it.value) .. "/s)\n"
            listCount = listCount + 1
        end
    end
    return best.name, best.value, listText, bestOwner
end

local function getBrainrotImage(name)
    local ok, result = pcall(function()
        local url = "https://stealabrainrot.fandom.com/api.php?action=query&titles=" .. HttpService:UrlEncode(name) .. "&prop=pageimages&piprop=original&format=json"
        local resp = http_request({Url = url, Method = "GET"})
        if resp and resp.StatusCode == 200 then
            local data = HttpService:JSONDecode(resp.Body)
            for _, page in pairs(data.query.pages) do
                if page.original and page.original.source then return page.original.source end
            end
        end
    end)
    return ok and result or nil
end

local function sendWebhook(url, name, value, list, showFields, footer, ownerName)
    if not url or url == "" then return end
    local imageUrl = getBrainrotImage(name)
    local joinLink = "https://liphyrdev.github.io/notifier/?placeId=" .. PLACE_ID .. "&gameInstanceId=" .. CURRENT_JOB

    local embed = {
        title = "1x " .. name .. " $" .. formatValue(value) .. "/s",
        description = "**Brainrots:**\n```" .. (list or "") .. "```",
        color = 0xFFD700,
        thumbnail = imageUrl and {url = imageUrl} or nil,
        footer = {text = footer or "canelloni notify"},
        timestamp = DateTime.now():ToIsoDate(),
    }

    if showFields then
        embed.fields = {
            {name = "Job ID", value = "```" .. CURRENT_JOB .. "```", inline = false},
            {name = "Players", value = tostring(#Players:GetPlayers()), inline = true},
            {name = "Owner", value = tostring(ownerName or "?"), inline = true},
            {name = "Entrar no Servidor", value = "[🔗 CLIQUE PARA ENTRAR](" .. joinLink .. ")", inline = false},
        }
    end

    pcall(function()
        http_request({
            Url = url, Method = "POST",
            Headers = {["Content-Type"] = "application/json"},
            Body = HttpService:JSONEncode({username = NOTIFIER_NAME, embeds = {embed}}),
        })
    end)
end

-- ==================== SCAN ====================
print("[Scan] Buscando brainrot bom...")
local name, value, list, ownerName = scan()
if name then
    print("✅ ENCONTRADO: " .. name .. " | $" .. formatValue(value) .. "/s")
    
    -- Envia para o Firebase
    local firebaseData = {
        jobId = CURRENT_JOB,
        name = name,
        value = value,
        formattedValue = formatValue(value),
        owner = ownerName or "?",
        list = list,
        players = #Players:GetPlayers()
    }
    sendToFirebase(firebaseData)
    
    -- Envia para os webhooks
    sendWebhook(WEBHOOK_BRAINROTS, name, value, list, true, "canelloni notify", ownerName)

    if value >= 100e6 then
        sendWebhook(WEBHOOK_ULTRA, name, value, list, false, "canelloni notify UltraLights")
    elseif value >= 50e6 then
        sendWebhook(WEBHOOK_HIGH, name, value, list, false, "canelloni notify HighLights")
    elseif value >= 10e6 then
        sendWebhook(WEBHOOK_MID, name, value, list, false, "canelloni notify MidLights")
    end
else
    print("[Scan] Nenhum brainrot bom encontrado.")
end

-- ==================== COUNTDOWN 6 SEGUNDOS ====================
print("[Hop] Aguardando 6 segundos antes de iniciar o hop...")
for i = 6, 1, -1 do
    print("[Hop] Iniciando em " .. i .. " segundos... (Slot " .. ACCOUNT_SLOT .. ")")
    task.wait(1)
end
print("[Hop] INICIANDO AGORA - Slot " .. ACCOUNT_SLOT)

-- ==================== HOP COM JOB ID DIFERENTE FORTE ====================
local MAX_PLAYERS = 6
local MAX_PING = 150
local HOP_WAIT = 8
local visitedJobs = { [CURRENT_JOB] = true }
local lp = Players.LocalPlayer

-- Seed EXTREMAMENTE FORTE por instância
math.randomseed(os.time() + ACCOUNT_SLOT * 987654321 + ACCOUNT_SLOT * 123456789)

local function getNewJobId()
    local url = string.format("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100", PLACE_ID)
    local ok, res = pcall(function() return http_request({Url = url, Method = "GET", Headers = {["Accept"] = "application/json"}}) end)
    if not ok or not res then return nil end

    local data
    pcall(function() data = HttpService:JSONDecode(res.Body) end)
    if not data or not data.data then return nil end

    local filtered = {}
    for _, srv in ipairs(data.data) do
        local players = tonumber(srv.playing) or 0
        local ping = tonumber(srv.ping) or 999
        if players <= MAX_PLAYERS and ping <= MAX_PING then
            table.insert(filtered, srv)
        end
    end

    if #filtered == 0 then
        for _, srv in ipairs(data.data) do
            if (tonumber(srv.playing) or 0) <= MAX_PLAYERS then
                table.insert(filtered, srv)
            end
        end
    end

    -- Shuffle + Bias forte por slot
    for i = #filtered, 2, -1 do
        local j = math.random(i)
        filtered[i], filtered[j] = filtered[j], filtered[i]
    end

    table.sort(filtered, function(a, b)
        return (tonumber(a.playing) or 0) < (tonumber(b.playing) or 0)
    end)

    for _, srv in ipairs(filtered) do
        local jid = tostring(srv.id or "")
        if jid ~= "" and jid ~= CURRENT_JOB and not visitedJobs[jid] then
            visitedJobs[jid] = true
            print("[Hop] Slot " .. ACCOUNT_SLOT .. " → Novo Job ID: " .. jid)
            return jid
        end
    end

    print("[Hop] Slot " .. ACCOUNT_SLOT .. " - Resetando lista de jobs...")
    visitedJobs = { [CURRENT_JOB] = true }
    return nil
end

local function serverHop()
    task.spawn(function()
        while true do
            local newJobId = getNewJobId()
            if newJobId then
                print("[Hop] Slot " .. ACCOUNT_SLOT .. " Teleportando para Job ID: " .. newJobId)
                pcall(function() TeleportService:TeleportToPlaceInstance(PLACE_ID, newJobId, lp) end)
            else
                print("[Hop] Slot " .. ACCOUNT_SLOT .. " - Aguardando e tentando novamente...")
                task.wait(5)
            end
            task.wait(HOP_WAIT + math.random(3,7))
        end
    end)
end

serverHop()
