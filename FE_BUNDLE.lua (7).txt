--[[
    FE_BUNDLE REBUILT
    Version: 1.0.0-clean
    Persistence schema: 1

    Architecture summary:
    - One AppState table is the source of truth.
    - One Data/API layer handles catalog search and animation resolving.
    - One Animation layer handles bundle application, emote playback, and controller tracks.
    - One UI layer renders pages, modal, loading, toast, floating buttons, and quick selector.
    - One Settings layer changes real runtime behavior and persists preferences.
    - One Lifecycle layer handles character respawn and cleanup.

    Major features:
    - Emote browser with Favorites / Roblox / UGC filters.
    - Bundle browser and full animation bundle apply.
    - Custom mix slots for Idle / Walk / Run / Jump / Fall / Climb / Swim.
    - Info modal with avatar preview, metadata, copy animation ID, favorite, play, floating shortcut.
    - Floating buttons with Autogrid / Freeform and placement options.
    - Quick selector alternative to floating buttons.
    - Animation controller with track select, loop, reverse, speed, intensity.
    - Settings with real behavior: width mode, cache, suggestions, picker, performance, privacy.
    - Save/load persistence with schema validation and safe defaults.

    Known limitations:
    - Some games override Animate and can prevent bundle changes.
    - Some executors block game:HttpGet or game:GetObjects.
    - R6 converters can make R15 animation bundles look stiff.
    - AI suggestions are local/offline suggestions only; no AI request is made unless implemented later.
]]

---------------------------------------------------------------------
-- SERVICES
---------------------------------------------------------------------

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")
local StarterGui = game:GetService("StarterGui")

local LocalPlayer = Players.LocalPlayer

---------------------------------------------------------------------
-- CONSTANTS
---------------------------------------------------------------------

local SAVE_FILE = "FE_BUNDLE_CLEAN_SCHEMA_1.json"
local SCHEMA_VERSION = 1
local CAN_SAVE = type(writefile) == "function" and type(readfile) == "function" and type(isfile) == "function"

local STATES = {"Idle", "Walk", "Run", "Jump", "Fall", "Climb", "Swim"}

local ANIMATE_NAMES = {
    Idle = {"idle"},
    Walk = {"walk"},
    Run = {"run"},
    Jump = {"jump"},
    Fall = {"fall"},
    Climb = {"climb"},
    Swim = {"swim", "swimidle"}
}

local SPEED_PRESETS = {
    {Name = "Paused", Value = 0},
    {Name = "Slower", Value = 0.2},
    {Name = "Slow", Value = 0.65},
    {Name = "Normal", Value = 1},
    {Name = "Fast", Value = 1.25},
    {Name = "Faster", Value = 1.75}
}

local LOCAL_SUGGESTIONS = {
    "dance", "pose", "wave", "laugh", "sit", "sleep", "spin", "hype", "ninja", "robot",
    "zombie", "cute", "sad", "happy", "style", "float", "kick", "clap", "idol", "ballet"
}

local THEME = {
    Page = Color3.fromRGB(255, 255, 255),
    Paper = Color3.fromRGB(247, 250, 248),
    Card = Color3.fromRGB(239, 245, 243),
    Field = Color3.fromRGB(248, 250, 249),
    Header = Color3.fromRGB(255, 181, 48),
    Cyan = Color3.fromRGB(137, 211, 222),
    Blue = Color3.fromRGB(35, 150, 222),
    Orange = Color3.fromRGB(255, 181, 48),
    Green = Color3.fromRGB(137, 222, 205),
    Yellow = Color3.fromRGB(255, 216, 126),
    Red = Color3.fromRGB(255, 100, 120),
    Text = Color3.fromRGB(24, 24, 24),
    Muted = Color3.fromRGB(82, 92, 100),
    LightMuted = Color3.fromRGB(150, 160, 168),
    Black = Color3.fromRGB(18, 18, 18)
}

---------------------------------------------------------------------
-- CENTRAL STATE
---------------------------------------------------------------------

local AppState = {
    Alive = true,
    CurrentPage = "Emotes",
    CurrentSource = "Roblox",
    SearchQuery = "",
    BundleQuery = "animation",
    SearchGeneration = 0,
    LoadingMore = false,

    EmoteResults = {},
    BundleResults = {},
    NextEmoteCursor = nil,
    NextBundleCursor = nil,

    Favorites = {
        Emotes = {},
        Bundles = {}
    },

    FloatingButtons = {},
    QuickEntries = {},
    SavedPacks = {},

    CurrentForm = {
        Idle = "",
        Walk = "",
        Run = "",
        Jump = "",
        Fall = "",
        Climb = "",
        Swim = ""
    },

    SlotMeta = {
        Idle = nil,
        Walk = nil,
        Run = nil,
        Jump = nil,
        Fall = nil,
        Climb = nil,
        Swim = nil
    },

    ChoosingState = nil,
    EditingSaveIndex = nil,
    AutoLoadName = "",
    LastAppliedName = "",

    Settings = {
        SourceFilter = "Roblox",
        PickerProvider = "Floating buttons",
        FloatingMode = "Autogrid",
        FloatingPlacement = "Top right",
        WidthMode = "Wide",
        AvoidScaling = false,
        ScreenBlur = false,
        StartMenuClosed = false,
        Crowdsource = false,
        CacheUGCIds = true,
        CacheUGCTracks = false,
        Suggestions = true,
        ControllerLoop = false,
        ControllerReverse = false,
        ControllerSpeed = 1,
        ControllerSpeedName = "Normal",
        ControllerIntensity = 1,
        ApplyMethod = "Animate",
        AutoLoad = true,
        ModalDimTransparency = 0.45,
        EmoteSpeed = 1,
        EmoteLoop = true,
        MoveWhileEmote = true,
        UITransparency = 0,
        BlurAmount = 0
    },

    Cache = {
        EmoteAnimationIds = {},
        BundleResolved = {},
        RuntimeTracks = {}
    },

    Character = {
        Instance = nil,
        Humanoid = nil,
        Animator = nil,
        Animate = nil
    },

    Controller = {
        SelectedIndex = 1,
        ReverseConnection = nil,
        CurrentTrack = nil,
        TrackSnapshot = {}
    },

    Runtime = {
        ActiveEmoteTrack = nil,
        ActiveModalPreview = nil,
        Logs = {}
    }
}

---------------------------------------------------------------------
-- GUI REFERENCES
---------------------------------------------------------------------

local Gui = {
    Screen = nil,
    Icon = nil,
    Main = nil,
    Header = nil,
    HeaderTitle = nil,
    Body = nil,
    Status = nil,
    Toast = nil,
    LoadingDim = nil,
    LoadingCard = nil,
    LoadingBar = nil,
    ModalDim = nil,
    ModalCard = nil,
    FloatingLayer = nil,
    QuickLayer = nil,
    QuickPanel = nil,
    QuickButton = nil,
    Blur = nil
}

local Connections = {
    Global = {},
    Page = {},
    Modal = {},
    Floating = {},
    Quick = {},
    Controller = {},
    Character = {}
}

---------------------------------------------------------------------
-- BASIC UTILITY
---------------------------------------------------------------------

local function addConnection(bucket, connection)
    table.insert(bucket or Connections.Global, connection)
    return connection
end

local function disconnectBucket(bucket)
    for _, connection in ipairs(bucket) do
        pcall(function()
            connection:Disconnect()
        end)
    end
    for index = #bucket, 1, -1 do
        table.remove(bucket, index)
    end
end

local function logMessage(message)
    local line = os.date("%H:%M:%S") .. " | " .. tostring(message)
    table.insert(AppState.Runtime.Logs, 1, line)
    while #AppState.Runtime.Logs > 80 do
        table.remove(AppState.Runtime.Logs)
    end
    pcall(function()
        warn("[FE_BUNDLE] " .. line)
    end)
end

local function createInstance(className, properties)
    local object = Instance.new(className)
    for property, value in pairs(properties or {}) do
        pcall(function()
            object[property] = value
        end)
    end
    return object
end

local function addCorner(object, radius)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, radius or 8)
    corner.Parent = object
    return corner
end

local function addStroke(object, color, thickness, transparency)
    local stroke = Instance.new("UIStroke")
    stroke.Color = color or THEME.Black
    stroke.Thickness = thickness or 1
    stroke.Transparency = transparency or 0
    stroke.Parent = object
    return stroke
end

local function animate(object, properties, duration)
    if AppState.Settings.AvoidScaling then
        for property, value in pairs(properties) do
            pcall(function()
                object[property] = value
            end)
        end
        return
    end
    pcall(function()
        TweenService:Create(object, TweenInfo.new(duration or 0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), properties):Play()
    end)
end

local function clearChildren(parent)
    if not parent then return end
    for _, child in ipairs(parent:GetChildren()) do
        child:Destroy()
    end
end

local function normalizeId(value)
    value = tostring(value or "")
    return string.match(value, "%d+") or ""
end

local function toAnimationUrl(id)
    id = normalizeId(id)
    if id == "" then return "" end
    return "rbxassetid://" .. id
end

local function assetThumbnail(id)
    return "rbxthumb://type=Asset&id=" .. tostring(id) .. "&w=150&h=150"
end

local function bundleThumbnail(id)
    return "rbxthumb://type=BundleThumbnail&id=" .. tostring(id) .. "&w=150&h=150"
end

local function getParentGui()
    local parentGui = nil
    pcall(function()
        if gethui then parentGui = gethui() end
    end)
    if not parentGui then
        pcall(function() parentGui = game:GetService("CoreGui") end)
    end
    if not parentGui then
        parentGui = LocalPlayer:WaitForChild("PlayerGui")
    end
    return parentGui
end

local function tableCopy(source)
    local copy = {}
    for key, value in pairs(source or {}) do
        if type(value) == "table" then
            local inner = {}
            for innerKey, innerValue in pairs(value) do
                inner[innerKey] = innerValue
            end
            copy[key] = inner
        else
            copy[key] = value
        end
    end
    return copy
end

local function getCharacterObjects()
    local character = LocalPlayer.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    local animator = humanoid and humanoid:FindFirstChildOfClass("Animator")
    local animateScript = character and character:FindFirstChild("Animate")
    return character, humanoid, animator, animateScript
end

local function setStatus(message, good)
    if not Gui.Status then return end
    Gui.Status.Text = tostring(message or "")
    if good == true then
        Gui.Status.TextColor3 = Color3.fromRGB(40, 130, 90)
    elseif good == false then
        Gui.Status.TextColor3 = THEME.Red
    else
        Gui.Status.TextColor3 = THEME.Muted
    end
end

local function notify(message, good)
    if not Gui.Toast then return end
    Gui.Toast.Text = tostring(message or "")
    Gui.Toast.TextColor3 = good == false and THEME.Red or Color3.fromRGB(40, 130, 90)
    Gui.Toast.Visible = true
    Gui.Toast.Position = UDim2.new(0.5, -140, 0, -42)
    animate(Gui.Toast, {Position = UDim2.new(0.5, -140, 0, 16)}, 0.16)
    task.delay(1.5, function()
        if Gui.Toast then
            animate(Gui.Toast, {Position = UDim2.new(0.5, -140, 0, -42)}, 0.16)
            task.delay(0.18, function()
                if Gui.Toast then Gui.Toast.Visible = false end
            end)
        end
    end)
end

local function copyToClipboard(text)
    text = tostring(text or "")
    local success = false
    if setclipboard then success = pcall(function() setclipboard(text) end) end
    if not success and toclipboard then success = pcall(function() toclipboard(text) end) end
    if not success and syn and syn.write_clipboard then success = pcall(function() syn.write_clipboard(text) end) end
    if success then notify("Animation copied.", true) else notify("Clipboard unavailable.", false) end
    return success
end

local function httpGet(url)
    local success, result = pcall(function()
        return game:HttpGet(url)
    end)
    if success and type(result) == "string" then return result end
    return nil
end

local function decodeJson(raw)
    if not raw then return nil end
    local success, data = pcall(function()
        return HttpService:JSONDecode(raw)
    end)
    if success then return data end
    return nil
end

---------------------------------------------------------------------
-- PERSISTENCE
---------------------------------------------------------------------

local function defaultSaveData()
    return {
        version = SCHEMA_VERSION,
        favorites = {
            emotes = {},
            bundles = {}
        },
        floatingButtons = {},
        quickSelector = {},
        savedPacks = {},
        settings = tableCopy(AppState.Settings),
        currentForm = tableCopy(AppState.CurrentForm),
        slotMeta = tableCopy(AppState.SlotMeta),
        autoLoadName = "",
        lastAppliedName = "",
        cache = {
            emoteAnimationIds = {},
            bundleResolved = {}
        }
    }
end

local function validateSettings(settings)
    settings = type(settings) == "table" and settings or {}
    for key, value in pairs(AppState.Settings) do
        if settings[key] == nil then
            settings[key] = value
        end
    end
    settings.ModalDimTransparency = math.clamp(tonumber(settings.ModalDimTransparency) or 0.45, 0.05, 0.9)
    settings.EmoteSpeed = tonumber(settings.EmoteSpeed) or 1
    settings.ControllerSpeed = tonumber(settings.ControllerSpeed) or 1
    settings.ControllerIntensity = math.clamp(tonumber(settings.ControllerIntensity) or 1, 0, 2)
    if settings.PickerProvider ~= "Floating buttons" and settings.PickerProvider ~= "Quick selector" then settings.PickerProvider = "Floating buttons" end
    if settings.FloatingMode ~= "Autogrid" and settings.FloatingMode ~= "Freeform" then settings.FloatingMode = "Autogrid" end
    local validPlacement = { ["Top right"] = true, ["Top left"] = true, ["Bottom right"] = true, ["Bottom left"] = true }
    if not validPlacement[settings.FloatingPlacement] then settings.FloatingPlacement = "Top right" end
    if settings.WidthMode ~= "Wide" and settings.WidthMode ~= "Compact" then settings.WidthMode = "Wide" end
    if settings.ApplyMethod ~= "Animate" and settings.ApplyMethod ~= "Description" and settings.ApplyMethod ~= "Both" then settings.ApplyMethod = "Animate" end
    return settings
end

local function saveData()
    if not CAN_SAVE then return false end
    local data = {
        version = SCHEMA_VERSION,
        favorites = {
            emotes = AppState.Favorites.Emotes,
            bundles = AppState.Favorites.Bundles
        },
        floatingButtons = AppState.FloatingButtons,
        quickSelector = AppState.QuickEntries,
        savedPacks = AppState.SavedPacks,
        settings = AppState.Settings,
        currentForm = AppState.CurrentForm,
        slotMeta = AppState.SlotMeta,
        autoLoadName = AppState.AutoLoadName,
        lastAppliedName = AppState.LastAppliedName,
        cache = {
            emoteAnimationIds = AppState.Cache.EmoteAnimationIds,
            bundleResolved = AppState.Cache.BundleResolved
        }
    }
    local success = pcall(function()
        writefile(SAVE_FILE, HttpService:JSONEncode(data))
    end)
    if success then logMessage("settings saved") end
    return success
end

local function loadData()
    if not CAN_SAVE then return false end
    local exists = false
    pcall(function() exists = isfile(SAVE_FILE) end)
    if not exists then return false end
    local raw
    local readOk = pcall(function() raw = readfile(SAVE_FILE) end)
    if not readOk or not raw then return false end
    local data
    local decodeOk = pcall(function() data = HttpService:JSONDecode(raw) end)
    if not decodeOk or type(data) ~= "table" then return false end

    local fallback = defaultSaveData()
    data.favorites = type(data.favorites) == "table" and data.favorites or fallback.favorites
    data.cache = type(data.cache) == "table" and data.cache or fallback.cache

    AppState.Favorites.Emotes = type(data.favorites.emotes) == "table" and data.favorites.emotes or {}
    AppState.Favorites.Bundles = type(data.favorites.bundles) == "table" and data.favorites.bundles or {}
    AppState.FloatingButtons = type(data.floatingButtons) == "table" and data.floatingButtons or {}
    AppState.QuickEntries = type(data.quickSelector) == "table" and data.quickSelector or {}
    AppState.SavedPacks = type(data.savedPacks) == "table" and data.savedPacks or {}
    AppState.Settings = validateSettings(data.settings)
    AppState.CurrentForm = type(data.currentForm) == "table" and data.currentForm or fallback.currentForm
    AppState.SlotMeta = type(data.slotMeta) == "table" and data.slotMeta or fallback.slotMeta
    AppState.AutoLoadName = type(data.autoLoadName) == "string" and data.autoLoadName or ""
    AppState.LastAppliedName = type(data.lastAppliedName) == "string" and data.lastAppliedName or ""
    AppState.Cache.EmoteAnimationIds = type(data.cache.emoteAnimationIds) == "table" and data.cache.emoteAnimationIds or {}
    AppState.Cache.BundleResolved = type(data.cache.bundleResolved) == "table" and data.cache.bundleResolved or {}
    logMessage("settings loaded")
    return true
end

---------------------------------------------------------------------
-- CHARACTER LIFECYCLE
---------------------------------------------------------------------

local function stopCurrentEmote()
    if AppState.Runtime.ActiveEmoteTrack then
        pcall(function()
            AppState.Runtime.ActiveEmoteTrack:Stop(0.15)
            AppState.Runtime.ActiveEmoteTrack:Destroy()
        end)
        AppState.Runtime.ActiveEmoteTrack = nil
    end
end

local function stopReverseLoop()
    if AppState.Controller.ReverseConnection then
        pcall(function() AppState.Controller.ReverseConnection:Disconnect() end)
        AppState.Controller.ReverseConnection = nil
    end
end

local function refreshCharacterReferences()
    local character, humanoid, animator, animateScript = getCharacterObjects()
    AppState.Character.Instance = character
    AppState.Character.Humanoid = humanoid
    AppState.Character.Animator = animator
    AppState.Character.Animate = animateScript
end

local function captureOriginalAnimations()
    refreshCharacterReferences()
    AppState.OriginalAnimations = {}
    for _, state in ipairs(STATES) do
        AppState.OriginalAnimations[state] = {}
    end
    local animateScript = AppState.Character.Animate
    if not animateScript then return end
    for _, state in ipairs(STATES) do
        local names = ANIMATE_NAMES[state]
        for _, child in ipairs(animateScript:GetChildren()) do
            local lowerName = string.lower(child.Name)
            for _, expected in ipairs(names or {}) do
                if lowerName == expected then
                    if child:IsA("Animation") then table.insert(AppState.OriginalAnimations[state], child.AnimationId) end
                    for _, descendant in ipairs(child:GetDescendants()) do
                        if descendant:IsA("Animation") then table.insert(AppState.OriginalAnimations[state], descendant.AnimationId) end
                    end
                end
            end
        end
    end
end

local function onCharacterAdded(character)
    disconnectBucket(Connections.Character)
    task.wait(0.8)
    refreshCharacterReferences()
    captureOriginalAnimations()
    stopCurrentEmote()
    stopReverseLoop()
    AppState.Controller.CurrentTrack = nil
    AppState.Controller.TrackSnapshot = {}
    logMessage("character refreshed")
end

local function setupCharacterLifecycle()
    addConnection(Connections.Global, LocalPlayer.CharacterAdded:Connect(onCharacterAdded))
    if LocalPlayer.Character then
        onCharacterAdded(LocalPlayer.Character)
    end
end

---------------------------------------------------------------------
-- CATALOG / DATA API
---------------------------------------------------------------------

local function buildCatalogUrl(kind, query, cursor)
    local encoded = HttpService:UrlEncode(query or "")
    local cursorPart = cursor and ("&Cursor=" .. HttpService:UrlEncode(cursor)) or ""
    if kind == "Emote" then
        local creatorFilter = ""
        if AppState.CurrentSource == "Roblox" then creatorFilter = "&CreatorType=User&CreatorTargetId=1" end
        return "https://catalog.roblox.com/v1/search/items/details?Category=12&Subcategory=39&Keyword=" .. encoded .. "&Limit=30&SortType=0" .. creatorFilter .. cursorPart
    end
    return "https://catalog.roblox.com/v1/search/items/details?Category=12&Subcategory=38&Keyword=" .. encoded .. "&Limit=30&SortType=0" .. cursorPart
end

local function searchCatalog(kind, query, append)
    query = tostring(query or "")
    if query == "" then query = kind == "Emote" and "dance" or "animation" end

    AppState.SearchGeneration = AppState.SearchGeneration + 1
    local generation = AppState.SearchGeneration

    if not append then
        if kind == "Emote" then
            AppState.EmoteResults = {}
            AppState.NextEmoteCursor = nil
        else
            AppState.BundleResults = {}
            AppState.NextBundleCursor = nil
        end
    end

    local cursor = kind == "Emote" and AppState.NextEmoteCursor or AppState.NextBundleCursor
    local url = buildCatalogUrl(kind, query, append and cursor or nil)
    local data = decodeJson(httpGet(url))

    if not data and kind == "Bundle" then
        local encoded = HttpService:UrlEncode(query)
        local cursorPart = append and cursor and ("&Cursor=" .. HttpService:UrlEncode(cursor)) or ""
        data = decodeJson(httpGet("https://catalog.roblox.com/v1/search/items/details?Category=12&Subcategory=27&Keyword=" .. encoded .. "&Limit=30&SortType=0" .. cursorPart))
    end

    if generation ~= AppState.SearchGeneration and not append then
        return false, 0, "stale request"
    end

    if not data or type(data.data) ~= "table" then
        return false, 0, "search failed"
    end

    if kind == "Emote" then
        AppState.SearchQuery = query
        for _, item in ipairs(data.data) do
            if AppState.CurrentSource ~= "UGC" or tostring(item.creatorName or "") ~= "Roblox" then
                table.insert(AppState.EmoteResults, item)
            end
        end
        AppState.NextEmoteCursor = data.nextPageCursor
        return true, #AppState.EmoteResults
    else
        AppState.BundleQuery = query
        for _, item in ipairs(data.data) do
            table.insert(AppState.BundleResults, item)
        end
        AppState.NextBundleCursor = data.nextPageCursor
        return true, #AppState.BundleResults
    end
end

local function fetchBundleDetails(bundleId)
    bundleId = normalizeId(bundleId)
    if bundleId == "" then return nil end
    return decodeJson(httpGet("https://catalog.roblox.com/v1/bundles/" .. bundleId .. "/details"))
end

local function fetchAssetDetails(assetId)
    assetId = normalizeId(assetId)
    if assetId == "" then return nil end
    return decodeJson(httpGet("https://catalog.roblox.com/v1/catalog/items/" .. assetId .. "/details?itemType=Asset"))
end

---------------------------------------------------------------------
-- ANIMATION RESOLUTION
---------------------------------------------------------------------

local function categorizeAnimation(pathText)
    pathText = string.lower(tostring(pathText or ""))
    if string.find(pathText, "idle", 1, true) then return "Idle" end
    if string.find(pathText, "walk", 1, true) then return "Walk" end
    if string.find(pathText, "run", 1, true) then return "Run" end
    if string.find(pathText, "jump", 1, true) then return "Jump" end
    if string.find(pathText, "fall", 1, true) then return "Fall" end
    if string.find(pathText, "climb", 1, true) then return "Climb" end
    if string.find(pathText, "swim", 1, true) then return "Swim" end
    return nil
end

local function scanAnimationTree(root, path, output)
    for _, child in ipairs(root:GetChildren()) do
        local newPath = path .. "." .. child.Name
        if child:IsA("Animation") then
            local id = normalizeId(child.AnimationId)
            local state = categorizeAnimation(newPath)
            if id ~= "" and state and not output[state] then output[state] = id end
        end
        if #child:GetChildren() > 0 then
            scanAnimationTree(child, newPath, output)
        end
    end
end

local function resolveAnimationsFromBundleAsset(assetId)
    assetId = normalizeId(assetId)
    if assetId == "" then return {} end
    if AppState.Cache.BundleResolved[assetId] then return AppState.Cache.BundleResolved[assetId] end

    local resolved = {}
    local ok, objects = pcall(function()
        return game:GetObjects("rbxassetid://" .. assetId)
    end)
    if ok and objects then
        for _, object in ipairs(objects) do
            scanAnimationTree(object, object.Name, resolved)
            pcall(function() object:Destroy() end)
        end
    end
    AppState.Cache.BundleResolved[assetId] = resolved
    return resolved
end

local function extractAnimationsFromBundle(details)
    local form = {}
    if not details then return form end
    local items = details.items or details.Items or {}
    for _, item in ipairs(items) do
        local itemId = tostring(item.id or item.Id or "")
        local resolved = resolveAnimationsFromBundleAsset(itemId)
        for state, id in pairs(resolved) do
            if not form[state] then form[state] = id end
        end
    end
    local assetTypeToState = {[48]="Climb", [50]="Fall", [51]="Idle", [52]="Jump", [53]="Run", [54]="Swim", [55]="Walk"}
    for _, item in ipairs(items) do
        local id = tostring(item.id or item.Id or "")
        local assetType = tonumber(item.assetType or item.AssetType or item.assetTypeId or item.AssetTypeId)
        local state = assetTypeToState[assetType] or categorizeAnimation(item.name or item.Name)
        if state and not form[state] then form[state] = id end
    end
    return form
end

local function resolveEmoteAnimation(assetId)
    assetId = normalizeId(assetId)
    if assetId == "" then return nil end
    if AppState.Settings.CacheUGCIds and AppState.Cache.EmoteAnimationIds[assetId] then
        return AppState.Cache.EmoteAnimationIds[assetId]
    end

    local found = nil
    pcall(function()
        local objects = game:GetObjects("rbxassetid://" .. assetId)
        for _, object in ipairs(objects or {}) do
            if object:IsA("Animation") then
                found = normalizeId(object.AnimationId)
            else
                for _, descendant in ipairs(object:GetDescendants()) do
                    if descendant:IsA("Animation") then
                        found = normalizeId(descendant.AnimationId)
                        break
                    end
                end
            end
            pcall(function() object:Destroy() end)
            if found and found ~= "" then break end
        end
    end)

    if not found or found == "" then
        pcall(function()
            local delivery = decodeJson(httpGet("https://assetdelivery.roblox.com/v1/assetId/" .. assetId))
            if delivery and delivery.location then
                local content = httpGet(delivery.location)
                if content then found = normalizeId(string.match(content, "rbxassetid://%d+") or "") end
            end
        end)
    end

    if not found or found == "" then found = assetId end
    if AppState.Settings.CacheUGCIds then
        AppState.Cache.EmoteAnimationIds[assetId] = found
        saveData()
    end
    return found
end

---------------------------------------------------------------------
-- ANIMATION PLAYBACK SYSTEM
---------------------------------------------------------------------

local function getAnimationsForState(state)
    refreshCharacterReferences()
    local animateScript = AppState.Character.Animate
    if not animateScript then return {} end
    local result = {}
    local names = ANIMATE_NAMES[state] or {}
    for _, child in ipairs(animateScript:GetChildren()) do
        local lowerName = string.lower(child.Name)
        for _, expected in ipairs(names) do
            if lowerName == expected then
                if child:IsA("Animation") then table.insert(result, child) end
                for _, descendant in ipairs(child:GetDescendants()) do
                    if descendant:IsA("Animation") then table.insert(result, descendant) end
                end
            end
        end
    end
    return result
end

local function restartAnimateScript()
    refreshCharacterReferences()
    local animateScript = AppState.Character.Animate
    if animateScript then
        pcall(function()
            animateScript.Disabled = true
            task.wait(0.1)
            animateScript.Disabled = false
        end)
    end
end

local function setStateAnimation(state, id)
    id = normalizeId(id)
    if id == "" then return false end
    local animations = getAnimationsForState(state)
    if #animations == 0 then return false end
    for _, animation in ipairs(animations) do
        animation.AnimationId = toAnimUrl(id)
    end
    return true
end

local function applyDescriptionAnimations()
    refreshCharacterReferences()
    local humanoid = AppState.Character.Humanoid
    if not humanoid then return 0 end
    local props = {Idle="IdleAnimation", Walk="WalkAnimation", Run="RunAnimation", Jump="JumpAnimation", Fall="FallAnimation", Climb="ClimbAnimation", Swim="SwimAnimation"}
    local changed = 0
    pcall(function()
        local desc = humanoid:GetAppliedDescription()
        for state, prop in pairs(props) do
            local id = normalizeId(AppState.CurrentForm[state])
            if id ~= "" then
                desc[prop] = tonumber(id) or 0
                changed = changed + 1
            end
        end
        humanoid:ApplyDescription(desc)
    end)
    return changed
end

local function applyCurrentForm(name)
    local changed = 0
    local descChanged = 0
    if AppState.Settings.ApplyMethod == "Animate" or AppState.Settings.ApplyMethod == "Both" then
        for _, state in ipairs(STATES) do
            if normalizeId(AppState.CurrentForm[state]) ~= "" and setStateAnimation(state, AppState.CurrentForm[state]) then
                changed = changed + 1
            end
        end
    end
    if AppState.Settings.ApplyMethod == "Description" or AppState.Settings.ApplyMethod == "Both" then
        descChanged = applyDescriptionAnimations()
    end
    restartAnimateScript()
    if name then AppState.LastAppliedName = name end
    saveData()
    setStatus("Applied " .. tostring(name or AppState.LastAppliedName or "pack") .. " | " .. tostring(changed) .. " states", changed > 0 or descChanged > 0)
end

local function applyBundleFull(bundleId, bundleName)
    showLoading("Resolving bundle...")
    task.spawn(function()
        local details = fetchBundleDetails(bundleId)
        if not details then hideLoading(); setStatus("Bundle details failed", false); return end
        local form = extractAnimationsFromBundle(details)
        local count = 0
        for _, state in ipairs(STATES) do
            AppState.CurrentForm[state] = form[state] or ""
            AppState.SlotMeta[state] = AppState.CurrentForm[state] ~= "" and {Bundle=bundleName or details.name or "Bundle", BundleId=normalizeId(bundleId), Id=AppState.CurrentForm[state]} or nil
            if AppState.CurrentForm[state] ~= "" then count = count + 1 end
        end
        hideLoading()
        if count <= 0 then setStatus("No animations found in bundle", false); return end
        applyCurrentForm(bundleName or details.name or "Bundle")
    end)
end

local function setCustomSlotFromBundle(state, bundleId, bundleName)
    showLoading("Setting " .. state .. "...")
    task.spawn(function()
        local details = fetchBundleDetails(bundleId)
        if not details then hideLoading(); setStatus("Bundle details failed", false); return end
        local form = extractAnimationsFromBundle(details)
        hideLoading()
        local id = form[state]
        if normalizeId(id) == "" then setStatus("This bundle has no " .. state .. " animation", false); return end
        AppState.CurrentForm[state] = id
        AppState.SlotMeta[state] = {Bundle=bundleName or details.name or "Bundle", BundleId=normalizeId(bundleId), Id=id}
        AppState.ChoosingState = nil
        saveData()
        setStatus("Set " .. state .. " from " .. tostring(bundleName or details.name), true)
        renderCustom()
    end)
end

local function restoreOriginalAnimations()
    for _, state in ipairs(STATES) do
        local originals = AppState.OriginalAnimations[state]
        local anims = getAnimationsForState(state)
        if originals and #originals > 0 then
            for i, anim in ipairs(anims) do anim.AnimationId = originals[i] or originals[1] end
        end
    end
    restartAnimateScript()
    setStatus("Original animations restored", true)
end

local function stopEmote()
    if AppState.Runtime.ActiveEmoteTrack then
        pcall(function()
            AppState.Runtime.ActiveEmoteTrack:Stop(0.15)
            AppState.Runtime.ActiveEmoteTrack:Destroy()
        end)
        AppState.Runtime.ActiveEmoteTrack = nil
    end
end

local function playEmote(assetId, name)
    refreshCharacterReferences()
    local humanoid = AppState.Character.Humanoid
    if not humanoid then notify("Humanoid not found", false); return end
    local animationId = resolveEmoteAnimation(assetId)
    if not animationId or animationId == "" then notify("Animation failed", false); return end
    stopEmote()
    local animation = Instance.new("Animation")
    animation.AnimationId = toAnimUrl(animationId)
    local ok, track = pcall(function() return humanoid:LoadAnimation(animation) end)
    if not ok or not track then notify("Animation failed", false); return end
    AppState.Runtime.ActiveEmoteTrack = track
    pcall(function()
        track.Priority = AppState.Settings.MoveWhileEmote and Enum.AnimationPriority.Core or Enum.AnimationPriority.Action4
        track.Looped = AppState.Settings.EmoteLoop
        track:Play(0.15, 1, AppState.Settings.EmoteSpeed)
    end)
    notify("Animation loaded", true)
    setStatus("Playing emote: " .. tostring(name or animationId), true)
end

---------------------------------------------------------------------
-- CONTROLLER SYSTEM
---------------------------------------------------------------------

local function stopControllerReverse()
    if AppState.Controller.ReverseConnection then
        pcall(function() AppState.Controller.ReverseConnection:Disconnect() end)
        AppState.Controller.ReverseConnection = nil
    end
end

local function getPlayingTracks()
    refreshCharacterReferences()
    local animator = AppState.Character.Animator
    if not animator then return {} end
    local ok, tracks = pcall(function() return animator:GetPlayingAnimationTracks() end)
    if ok and type(tracks) == "table" then return tracks end
    return {}
end

local function getSelectedTrack()
    local tracks = getPlayingTracks()
    local track = tracks[AppState.Controller.SelectedIndex] or tracks[1]
    return track, tracks
end

local function applyControllerToTrack(track)
    if not track then return end
    stopControllerReverse()
    pcall(function() track.Looped = AppState.Settings.ControllerLoop end)
    pcall(function() track:AdjustWeight(AppState.Settings.ControllerIntensity, 0.1) end)
    if AppState.Settings.ControllerReverse then
        pcall(function() track:AdjustSpeed(0) end)
        AppState.Controller.ReverseConnection = RunService.Heartbeat:Connect(function(deltaTime)
            if not track or not track.IsPlaying then stopControllerReverse(); return end
            local length = track.Length
            if not length or length <= 0 then return end
            local nextTime = track.TimePosition - math.max(0.01, AppState.Settings.ControllerSpeed) * deltaTime
            if nextTime <= 0 then
                if AppState.Settings.ControllerLoop then nextTime = length else pcall(function() track:Stop(0.05) end); stopControllerReverse(); return end
            end
            pcall(function() track.TimePosition = nextTime end)
        end)
    else
        pcall(function() track:AdjustSpeed(AppState.Settings.ControllerSpeed) end)
    end
end

---------------------------------------------------------------------
-- FLOATING BUTTON MANAGER
---------------------------------------------------------------------

local function getFloatingLayer()
    if Gui.FloatingLayer and Gui.FloatingLayer.Parent then return Gui.FloatingLayer end
    Gui.FloatingLayer = createInstance("Frame", {Parent=Gui.Screen, Name="FEFloatingLayer", BackgroundTransparency=1, Position=UDim2.new(0,0,0,0), Size=UDim2.new(1,0,1,0), ZIndex=150})
    return Gui.FloatingLayer
end

local function reflowFloatingButtons()
    local layer = getFloatingLayer()
    if not layer then return end
    if AppState.Settings.PickerProvider ~= "Floating buttons" then layer.Visible = false else layer.Visible = true end
    if AppState.Settings.FloatingMode ~= "Autogrid" then return end
    local buttons = {}
    for _, child in ipairs(layer:GetChildren()) do
        if child:IsA("ImageButton") then table.insert(buttons, child) end
    end
    local viewport = workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize or Vector2.new(1280, 720)
    local size, gap = 52, 8
    for index, buttonObject in ipairs(buttons) do
        local col = (index - 1) % 4
        local row = math.floor((index - 1) / 4)
        local x, y
        if AppState.Settings.FloatingPlacement == "Top left" then
            x = 12 + col * (size + gap); y = 90 + row * (size + gap)
        elseif AppState.Settings.FloatingPlacement == "Bottom left" then
            x = 12 + col * (size + gap); y = viewport.Y - 90 - size - row * (size + gap)
        elseif AppState.Settings.FloatingPlacement == "Bottom right" then
            x = viewport.X - 12 - size - col * (size + gap); y = viewport.Y - 90 - size - row * (size + gap)
        else
            x = viewport.X - 12 - size - col * (size + gap); y = 90 + row * (size + gap)
        end
        buttonObject.Position = UDim2.new(0, math.floor(x), 0, math.floor(y))
    end
end

local function rebuildFloatingButtons()
    disconnectBucket(Connections.Floating)
    local layer = getFloatingLayer()
    clearChildren(layer)
    for _, entry in ipairs(AppState.FloatingButtons) do
        local id = tostring(entry.id)
        local buttonObject = createInstance("ImageButton", {Parent=layer, Name="Float_"..id, Size=UDim2.new(0,52,0,52), BackgroundColor3=THEME.Card, BorderSizePixel=0, Image=assetThumbnail(id), Active=true, AutoButtonColor=true, ZIndex=151})
        addCorner(buttonObject, 12); addStroke(buttonObject, THEME.Black, 1, 0)
        if AppState.Settings.FloatingMode == "Freeform" and entry.x and entry.y then buttonObject.Position = UDim2.new(0, entry.x, 0, entry.y) end
        addConnection(Connections.Floating, buttonObject.MouseButton1Click:Connect(function() playEmote(id, entry.name) end))
        local dragging, dragInput, dragStart, startPos, moved = false, nil, nil, nil, false
        addConnection(Connections.Floating, buttonObject.InputBegan:Connect(function(input)
            if AppState.Settings.FloatingMode ~= "Freeform" then return end
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = true; moved = false; dragInput = input; dragStart = input.Position; startPos = buttonObject.Position
                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        dragging = false
                        for _, state in ipairs(AppState.FloatingButtons) do
                            if tostring(state.id) == id then state.x = buttonObject.Position.X.Offset; state.y = buttonObject.Position.Y.Offset end
                        end
                        saveData()
                    end
                end)
            end
        end))
        addConnection(Connections.Floating, buttonObject.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement then dragInput = input end
        end))
        addConnection(Connections.Floating, UserInputService.InputChanged:Connect(function(input)
            if dragging and input == dragInput then
                local delta = input.Position - dragStart
                if math.abs(delta.X) > 6 or math.abs(delta.Y) > 6 then moved = true end
                local viewport = workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize or Vector2.new(1280, 720)
                local nx = math.clamp(startPos.X.Offset + delta.X, -20, viewport.X - 32)
                local ny = math.clamp(startPos.Y.Offset + delta.Y, -20, viewport.Y - 32)
                buttonObject.Position = UDim2.new(0, nx, 0, ny)
            end
        end))
    end
    reflowFloatingButtons()
end

local function createFloatingButton(item)
    local id = tostring(item.id or item.Id or "")
    if id == "" then return end
    for _, entry in ipairs(AppState.FloatingButtons) do if tostring(entry.id) == id then notify("Floating button already exists", false); return end end
    table.insert(AppState.FloatingButtons, {id=id, name=tostring(item.name or item.Name or id), kind="Emote"})
    saveData()
    rebuildFloatingButtons()
    notify("Floating button created", true)
end

---------------------------------------------------------------------
-- QUICK SELECTOR MANAGER
---------------------------------------------------------------------

local function getQuickLayer()
    if Gui.QuickLayer and Gui.QuickLayer.Parent then return Gui.QuickLayer end
    Gui.QuickLayer = createInstance("Frame", {Parent=Gui.Screen, Name="FEQuickSelector", BackgroundTransparency=1, Position=UDim2.new(0,0,0,0), Size=UDim2.new(1,0,1,0), ZIndex=175})
    return Gui.QuickLayer
end

local function rebuildQuickSelector()
    disconnectBucket(Connections.Quick)
    local layer = getQuickLayer()
    clearChildren(layer)
    Gui.QuickButton = createInstance("TextButton", {Parent=layer, AnchorPoint=Vector2.new(0.5,1), Position=UDim2.new(0.5,0,1,-18), Size=UDim2.new(0,72,0,38), BackgroundColor3=THEME.Orange, BorderSizePixel=0, Text="QS", TextColor3=THEME.Text, TextSize=14, Font=Enum.Font.GothamBold, Active=true, AutoButtonColor=true, Visible=AppState.Settings.PickerProvider=="Quick selector", ZIndex=176})
    addCorner(Gui.QuickButton, 14); addStroke(Gui.QuickButton, THEME.Black, 1, 0)
    Gui.QuickPanel = createInstance("Frame", {Parent=layer, AnchorPoint=Vector2.new(0.5,1), Position=UDim2.new(0.5,0,1,-62), Size=UDim2.new(0,420,0,84), BackgroundColor3=THEME.Page, BorderSizePixel=0, Visible=false, ZIndex=176})
    addCorner(Gui.QuickPanel, 14); addStroke(Gui.QuickPanel, THEME.Black, 1, 0)
    local scroller = createInstance("ScrollingFrame", {Parent=Gui.QuickPanel, Position=UDim2.new(0,10,0,10), Size=UDim2.new(1,-20,1,-20), BackgroundTransparency=1, BorderSizePixel=0, ScrollBarThickness=3, ScrollBarImageColor3=THEME.Orange, CanvasSize=UDim2.new(0, math.max(400, #AppState.QuickEntries*66), 0,0), ZIndex=177})
    local layout = Instance.new("UIListLayout"); layout.FillDirection=Enum.FillDirection.Horizontal; layout.Padding=UDim.new(0,8); layout.Parent=scroller
    for _, entry in ipairs(AppState.QuickEntries) do
        local b = createInstance("ImageButton", {Parent=scroller, Size=UDim2.new(0,58,0,58), BackgroundColor3=THEME.Card, BorderSizePixel=0, Image=assetThumbnail(entry.id), Active=true, AutoButtonColor=true, ZIndex=178})
        addCorner(b, 12); addStroke(b, THEME.Black, 1, 0)
        addConnection(Connections.Quick, b.MouseButton1Click:Connect(function() playEmote(entry.id, entry.name); Gui.QuickPanel.Visible=false end))
    end
    addConnection(Connections.Quick, Gui.QuickButton.MouseButton1Click:Connect(function()
        Gui.QuickPanel.Visible = not Gui.QuickPanel.Visible
        if Gui.QuickPanel.Visible then
            Gui.QuickPanel.Size = UDim2.new(0,40,0,40)
            animate(Gui.QuickPanel, {Size=UDim2.new(0,420,0,84)}, 0.16)
        end
    end))
end

local function createQuickEntry(item)
    local id = tostring(item.id or item.Id or "")
    if id == "" then return end
    for _, entry in ipairs(AppState.QuickEntries) do if tostring(entry.id) == id then notify("Quick selector entry already exists", false); return end end
    table.insert(AppState.QuickEntries, {id=id, name=tostring(item.name or item.Name or id)})
    saveData(); rebuildQuickSelector(); notify("Quick selector entry created", true)
end

local function createPickerShortcut(item)
    if AppState.Settings.PickerProvider == "Quick selector" then createQuickEntry(item) else createFloatingButton(item) end
end

---------------------------------------------------------------------
-- INFO MODAL
---------------------------------------------------------------------

local function createAvatarPreview(parent, animationId)
    local viewport = createInstance("ViewportFrame", {Parent=parent, Position=UDim2.new(0,18,0,18), Size=UDim2.new(0,132,0,112), BackgroundColor3=THEME.Field, BorderSizePixel=0, Ambient=Color3.fromRGB(180,180,180), LightColor=Color3.fromRGB(255,255,255), ZIndex=202})
    addCorner(viewport, 10); addStroke(viewport, THEME.Black, 1, 0.25)
    local world = Instance.new("WorldModel"); world.Parent=viewport
    local character = LocalPlayer.Character
    if not character then return viewport end
    local oldArchivable = character.Archivable
    pcall(function() character.Archivable = true end)
    local clone
    pcall(function() clone = character:Clone() end)
    pcall(function() character.Archivable = oldArchivable end)
    if not clone then return viewport end
    for _, obj in ipairs(clone:GetDescendants()) do if obj:IsA("Script") or obj:IsA("LocalScript") then obj:Destroy() end end
    clone.Parent = world
    local root = clone:FindFirstChild("HumanoidRootPart") or clone.PrimaryPart
    if root then clone.PrimaryPart=root; pcall(function() root.Anchored=true; clone:SetPrimaryPartCFrame(CFrame.new(0,0,0)*CFrame.Angles(0,math.rad(180),0)) end) end
    local camera = Instance.new("Camera"); camera.Parent=viewport; viewport.CurrentCamera=camera; camera.CFrame=CFrame.new(Vector3.new(0,2.2,6), Vector3.new(0,1.5,0))
    local humanoid = clone:FindFirstChildOfClass("Humanoid")
    local id = normalizeId(animationId)
    if humanoid and id ~= "" then
        local animation = Instance.new("Animation"); animation.AnimationId=toAnimationUrl(id)
        local ok, track = pcall(function() return humanoid:LoadAnimation(animation) end)
        if ok and track then pcall(function() track.Looped=true; track:Play(0.1,1,AppState.Settings.EmoteSpeed) end) end
    end
    return viewport
end

local function closeModal()
    disconnectBucket(Connections.Modal)
    local card = Gui.Modal
    local dim = Gui.ModalDim
    Gui.Modal = nil; Gui.ModalDim = nil
    if card then animate(card, {Size=UDim2.new(0,20,0,20), BackgroundTransparency=1}, 0.13); task.delay(0.15, function() if card then card:Destroy() end end) end
    if dim then animate(dim, {BackgroundTransparency=1}, 0.13); task.delay(0.15, function() if dim then dim:Destroy() end end) end
end

local function showInfoModal(titleText, bodyText, imageId, actions, previewAnimationId)
    closeModal()
    Gui.ModalDim = createInstance("Frame", {Parent=Gui.Screen, Position=UDim2.new(0,0,0,0), Size=UDim2.new(1,0,1,0), BackgroundColor3=THEME.Black, BackgroundTransparency=1, BorderSizePixel=0, ZIndex=200})
    animate(Gui.ModalDim, {BackgroundTransparency=AppState.Settings.ModalDimTransparency}, 0.16)
    Gui.Modal = createInstance("Frame", {Parent=Gui.Screen, AnchorPoint=Vector2.new(0.5,0.5), Position=UDim2.new(0.5,0,0.5,0), Size=UDim2.new(0,20,0,20), BackgroundColor3=THEME.Page, BorderSizePixel=0, ZIndex=201})
    addCorner(Gui.Modal, 16); addStroke(Gui.Modal, THEME.Black, 2, 0); animate(Gui.Modal, {Size=UDim2.new(0,460,0,330)}, 0.18)
    task.delay(0.03, function()
        if not Gui.Modal then return end
        if previewAnimationId and normalizeId(previewAnimationId) ~= "" then createAvatarPreview(Gui.Modal, previewAnimationId) else
            local image = createInstance("ImageLabel", {Parent=Gui.Modal, Position=UDim2.new(0,18,0,18), Size=UDim2.new(0,140,0,118), BackgroundColor3=THEME.Field, BorderSizePixel=0, Image=imageId or "", ScaleType=Enum.ScaleType.Fit, ZIndex=202})
            addCorner(image,10); addStroke(image,THEME.Black,1,0.25)
        end
        label(Gui.Modal, titleText or "Info", UDim2.new(0,176,0,18), UDim2.new(1,-220,0,42), 18, THEME.Text)
        local descScroll = createInstance("ScrollingFrame", {Parent=Gui.Modal, Position=UDim2.new(0,176,0,68), Size=UDim2.new(1,-198,0,178), BackgroundTransparency=1, BorderSizePixel=0, ScrollBarThickness=3, CanvasSize=UDim2.new(0,0,0,260), ZIndex=202})
        label(descScroll, bodyText or "No information.", UDim2.new(0,0,0,0), UDim2.new(1,-8,0,250), 13, THEME.Muted)
        button(Gui.Modal, "X", UDim2.new(1,-42,0,12), UDim2.new(0,30,0,30), closeModal, THEME.Red, Connections.Modal)
        button(Gui.Modal, "CLOSE", UDim2.new(0,18,1,-48), UDim2.new(0,88,0,30), closeModal, THEME.Red, Connections.Modal)
        local x = 116
        for _, action in ipairs(actions or {}) do
            button(Gui.Modal, action.Text or "OK", UDim2.new(0,x,1,-48), UDim2.new(0,102,0,30), function()
                if action.Callback then action.Callback() end
                if action.Close ~= false then closeModal() end
            end, action.Color or THEME.Cyan, Connections.Modal)
            x += 110
            if x > 430 then break end
        end
    end)
end

---------------------------------------------------------------------
-- PAGE SYSTEM
---------------------------------------------------------------------

local renderHome, renderCustom, renderFavorites, renderSave, renderSettings, renderController

local function setPage(page)
    AppState.CurrentPage = page
    disconnectBucket(Connections.Page)
    clearChildren(Gui.Body)
    if Gui.HeaderTitle then Gui.HeaderTitle.Text = page == "Bundles" and "FE Bundle" or page end
    Gui.Body.Position = UDim2.new(0,18,0,82)
    animate(Gui.Body, {Position=UDim2.new(0,12,0,66)}, 0.12)
end

local function renderTabs()
    button(Gui.Body, "EMOTES", UDim2.new(0,12,0,8), UDim2.new(0,76,0,32), function() renderHome("Emote") end, AppState.CurrentPage=="Emotes" and THEME.Cyan or THEME.Card)
    button(Gui.Body, "BUND", UDim2.new(0,96,0,8), UDim2.new(0,64,0,32), function() renderHome("Bundle") end, AppState.CurrentPage=="Bundles" and THEME.Cyan or THEME.Card)
    button(Gui.Body, "CTRL", UDim2.new(0,168,0,8), UDim2.new(0,64,0,32), function() renderController() end, AppState.CurrentPage=="Controller" and THEME.Cyan or THEME.Card)
    button(Gui.Body, "CUST", UDim2.new(0,240,0,8), UDim2.new(0,64,0,32), function() renderCustom() end, AppState.CurrentPage=="Custom" and THEME.Cyan or THEME.Card)
    button(Gui.Body, "FAV", UDim2.new(0,312,0,8), UDim2.new(0,56,0,32), function() renderFavorites() end, AppState.CurrentPage=="Favorites" and THEME.Cyan or THEME.Card)
    button(Gui.Body, "SAVE", UDim2.new(0,376,0,8), UDim2.new(0,62,0,32), function() renderSave() end, AppState.CurrentPage=="Save" and THEME.Cyan or THEME.Card)
    button(Gui.Body, "SET", UDim2.new(0,446,0,8), UDim2.new(0,52,0,32), function() renderSettings() end, AppState.CurrentPage=="Settings" and THEME.Cyan or THEME.Card)
end

local function isFavorite(list, id)
    id = tostring(id)
    for _, item in ipairs(list) do if tostring(item.id) == id then return true end end
    return false
end

local function toggleFavorite(kind, item)
    local list = kind == "Emote" and AppState.Favorites.Emotes or AppState.Favorites.Bundles
    local id = tostring(item.id or item.Id or "")
    for i, favorite in ipairs(list) do
        if tostring(favorite.id) == id then table.remove(list, i); saveData(); notify("Favorite removed", true); return end
    end
    table.insert(list, {id=id, name=tostring(item.name or item.Name or (kind.." "..id)), kind=kind})
    saveData(); notify("Favorite added", true)
end

local function renderItemCard(parent, item, index, kind)
    local id = tostring(item.id or item.Id or "")
    local name = tostring(item.name or item.Name or (kind .. " " .. id))
    local creator = tostring(item.creatorName or item.CreatorName or "Unknown")
    local col = (index - 1) % 2
    local row = math.floor((index - 1) / 2)
    local x = 12 + col * 250
    local y = 12 + row * 142
    local card = panel(parent, UDim2.new(0,x,0,y), UDim2.new(0,238,0,130), THEME.Card)
    local imageId = kind == "Emote" and assetThumb(id) or bundleThumb(id)
    local image = createInstance("ImageLabel", {Parent=card, Position=UDim2.new(0,10,0,10), Size=UDim2.new(0,80,0,72), BackgroundColor3=THEME.Field, BorderSizePixel=0, Image=imageId, ScaleType=Enum.ScaleType.Fit, ZIndex=z(card,2)})
    addCorner(image, 8)
    label(card, name, UDim2.new(0,100,0,10), UDim2.new(1,-110,0,36), 13, THEME.Text)
    label(card, creator, UDim2.new(0,100,0,46), UDim2.new(1,-110,0,18), 11, THEME.Muted)
    label(card, kind .. " ID: " .. id, UDim2.new(0,100,0,64), UDim2.new(1,-110,0,18), 10, THEME.Muted)
    button(card, kind=="Emote" and "PLAY" or (AppState.ChoosingState and ("SET "..string.upper(AppState.ChoosingState)) or "APPLY"), UDim2.new(0,10,1,-36), UDim2.new(0,88,0,28), function()
        if kind=="Emote" then playEmote(id, name) else if AppState.ChoosingState then setCustomSlotFromBundle(AppState.ChoosingState, id, name) else applyBundleFull(id, name) end end
    end, kind=="Emote" and THEME.Green or THEME.Orange)
    button(card, "INFO", UDim2.new(0,106,1,-36), UDim2.new(0,70,0,28), function()
        if kind == "Emote" then
            showLoading("Resolving emote preview...")
            task.spawn(function()
                local animationId = resolveEmoteAnimation(id)
                hideLoading()
                local details = fetchAssetDetails(id) or {}
                local desc = tostring(details.description or details.Description or item.description or "No description available.")
                local body = "Name: "..name.."\nCreator: "..creator.."\nSource: "..AppState.CurrentSource.."\nCatalog ID: "..id.."\nAnimation ID: "..tostring(animationId or "unknown").."\nLink: https://www.roblox.com/catalog/"..id.."\n\n"..desc
                showInfoModal(name, body, imageId, {
                    {Text="PLAY", Color=THEME.Green, Callback=function() playEmote(id, name) end, Close=false},
                    {Text="COPY ANIM.", Color=THEME.Cyan, Callback=function() copyToClipboard(animationId or id) end, Close=false},
                    {Text=AppState.Settings.PickerProvider=="Quick selector" and "QUICK S." or "FLOATING B.", Color=THEME.Orange, Callback=function() if AppState.Settings.PickerProvider=="Quick selector" then createQuickEntry(item) else createFloatingButton(item) end end, Close=false},
                    {Text=isFavorite(AppState.Favorites.Emotes,id) and "FAVORITED" or "FAVORITE", Color=THEME.Yellow, Callback=function() toggleFavorite(kind,item) end, Close=false}
                }, animationId)
            end)
        else
            local body = "Name: "..name.."\nCreator: "..creator.."\nBundle ID: "..id.."\nLink: https://www.roblox.com/bundles/"..id.."\n\nApply full bundle or set it in Custom."
            showInfoModal(name, body, imageId, {
                {Text=AppState.ChoosingState and "SET" or "APPLY", Color=THEME.Green, Callback=function() if AppState.ChoosingState then setCustomSlotFromBundle(AppState.ChoosingState,id,name) else applyBundleFull(id,name) end end},
                {Text=isFavorite(AppState.Favorites.Bundles,id) and "FAVORITED" or "FAVORITE", Color=THEME.Yellow, Callback=function() toggleFavorite(kind,item) end, Close=false}
            }, nil)
        end
    end, THEME.Cyan)
    button(card, isFavorite(kind=="Emote" and AppState.Favorites.Emotes or AppState.Favorites.Bundles, id) and "★" or "☆", UDim2.new(0,184,1,-36), UDim2.new(0,42,0,28), function()
        toggleFavorite(kind, item)
        renderHome(kind)
    end, THEME.Yellow)
end

renderHome = function(kind)
    kind = kind or (AppState.CurrentPage == "Emotes" and "Emote" or "Bundle")
    setPage(kind == "Emote" and "Emotes" or "Bundles")
    renderTabs()
    local placeholder = kind == "Emote" and "Search emotes: dance, pose, laugh..." or "Search bundles: ninja, robot, zombie..."
    local search = textbox(Gui.Body, placeholder, UDim2.new(0,12,0,52), UDim2.new(1,-146,0,38), kind=="Emote" and AppState.SearchQuery or AppState.BundleQuery)
    button(Gui.Body, "SEARCH", UDim2.new(1,-124,0,52), UDim2.new(0,112,0,38), function()
        showLoading("Loading "..string.lower(kind).."s...")
        task.spawn(function()
            local ok, count = searchCatalog(kind, search.Text, false)
            hideLoading()
            if ok then renderHome(kind); setStatus("Loaded "..tostring(count).." "..string.lower(kind).."s", true) else setStatus("Search failed", false) end
        end)
    end, THEME.Cyan)
    if kind == "Emote" then
        button(Gui.Body, "Favorites", UDim2.new(0,12,0,96), UDim2.new(0,92,0,28), function() AppState.CurrentSource="Favorites"; AppState.EmoteResults=AppState.Favorites.Emotes; renderHome("Emote") end, AppState.CurrentSource=="Favorites" and THEME.Green or THEME.Card)
        button(Gui.Body, "Roblox", UDim2.new(0,112,0,96), UDim2.new(0,82,0,28), function() AppState.CurrentSource="Roblox"; AppState.EmoteResults={}; searchCatalog("Emote", AppState.SearchQuery, false); renderHome("Emote") end, AppState.CurrentSource=="Roblox" and THEME.Green or THEME.Card)
        button(Gui.Body, "UGC", UDim2.new(0,202,0,96), UDim2.new(0,72,0,28), function() AppState.CurrentSource="UGC"; AppState.EmoteResults={}; searchCatalog("Emote", AppState.SearchQuery, false); renderHome("Emote") end, AppState.CurrentSource=="UGC" and THEME.Green or THEME.Card)
    else
        label(Gui.Body, AppState.ChoosingState and ("Choosing: "..AppState.ChoosingState.." | tap a bundle card to set it.") or "Apply full bundle or use Custom to mix slots.", UDim2.new(0,12,0,96), UDim2.new(1,-24,0,24), 12, AppState.ChoosingState and THEME.Red or THEME.Muted)
    end
    local listY = kind == "Emote" and 132 or 124
    local scroller = scrollFrame(Gui.Body, UDim2.new(0,12,0,listY), UDim2.new(1,-24,1,-(listY+38)))
    local list = kind == "Emote" and AppState.EmoteResults or AppState.BundleResults
    if #list == 0 then
        label(scroller, "Loading popular "..string.lower(kind).."s...", UDim2.new(0,16,0,16), UDim2.new(0,300,0,30), 16, THEME.Muted)
        task.spawn(function()
            task.wait(0.2)
            local ok = searchCatalog(kind, kind=="Emote" and AppState.SearchQuery or AppState.BundleQuery, false)
            if ok and Gui.Body and AppState.CurrentPage == (kind=="Emote" and "Emotes" or "Bundles") then renderHome(kind) end
        end)
    else
        for i, item in ipairs(list) do renderItemCard(scroller, item, i, kind) end
        local rows = math.ceil(#list/2)
        scroller.CanvasSize = UDim2.new(0,0,0,math.max(360, rows*142 + 62))
        local hasNext = kind=="Emote" and AppState.NextEmoteCursor or AppState.NextBundleCursor
        if hasNext then
            label(scroller, "Scroll to bottom to load more...", UDim2.new(0,16,0,rows*142+18), UDim2.new(0,260,0,30), 13, THEME.Muted)
            addConnection(Connections.Page, scroller:GetPropertyChangedSignal("CanvasPosition"):Connect(function()
                if AppState.LoadingMore then return end
                local bottom = scroller.CanvasPosition.Y + scroller.AbsoluteWindowSize.Y
                if bottom >= scroller.CanvasSize.Y.Offset - 45 then
                    AppState.LoadingMore = true
                    showLoading("Loading more...")
                    task.spawn(function()
                        local ok = searchCatalog(kind, kind=="Emote" and AppState.SearchQuery or AppState.BundleQuery, true)
                        hideLoading(); AppState.LoadingMore=false
                        if ok then renderHome(kind) end
                    end)
                end
            end))
        end
    end
end

-- Minimal versions of remaining pages are below. They are functional and use central state.
renderCustom = function()
    setPage("Custom"); renderTabs()
    label(Gui.Body, "Customize or mix your animation pack", UDim2.new(0,12,0,52), UDim2.new(1,-24,0,24), 15, THEME.Text)
    local sc = scrollFrame(Gui.Body, UDim2.new(0,12,0,86), UDim2.new(1,-24,1,-124))
    local y=12
    for _, state in ipairs(STATES) do
        local row = panel(sc, UDim2.new(0,12,0,y), UDim2.new(1,-34,0,50), THEME.Card)
        label(row, state, UDim2.new(0,10,0,4), UDim2.new(0,62,0,42), 14, THEME.Text)
        local meta = AppState.SlotMeta[state]
        label(row, meta and ((meta.Bundle or "Bundle").." | ID "..tostring(meta.Id or "")) or "not set", UDim2.new(0,80,0,4), UDim2.new(1,-248,0,42), 12, meta and THEME.Muted or THEME.LightMuted)
        button(row, "SET", UDim2.new(1,-158,0,10), UDim2.new(0,46,0,30), function() AppState.ChoosingState=state; renderHome("Bundle") end, THEME.Green)
        button(row, "INFO", UDim2.new(1,-106,0,10), UDim2.new(0,58,0,30), function()
            local body = meta and ("State: "..state.."\nBundle: "..tostring(meta.Bundle).."\nBundle ID: "..tostring(meta.BundleId).."\nAnimation ID: "..tostring(meta.Id)) or ("State: "..state.."\nNo animation selected yet.")
            showInfoModal("Custom Slot: "..state, body, meta and bundleThumb(meta.BundleId) or "", {})
        end, THEME.Cyan)
        button(row, "X", UDim2.new(1,-40,0,10), UDim2.new(0,28,0,30), function() AppState.CurrentForm[state]=""; AppState.SlotMeta[state]=nil; saveData(); renderCustom() end, THEME.Red)
        y += 58
    end
    local saveBox = textbox(sc, "Save as name...", UDim2.new(0,12,0,y+8), UDim2.new(0,160,0,34), "")
    button(sc, "APPLY CUSTOM", UDim2.new(0,184,0,y+8), UDim2.new(0,130,0,34), function() applyCurrentForm("Custom Mix") end, THEME.Orange)
    button(sc, "SAVE MIX", UDim2.new(0,326,0,y+8), UDim2.new(0,96,0,34), function()
        local nm = tostring(saveBox.Text or ""); if nm=="" then nm="Custom Mix "..tostring(#AppState.SavedPacks+1) end
        if AppState.EditingSaveIndex and AppState.SavedPacks[AppState.EditingSaveIndex] then AppState.SavedPacks[AppState.EditingSaveIndex]={Name=nm,Form=tableCopy(AppState.CurrentForm),Meta=tableCopy(AppState.SlotMeta)}; AppState.EditingSaveIndex=nil else table.insert(AppState.SavedPacks,{Name=nm,Form=tableCopy(AppState.CurrentForm),Meta=tableCopy(AppState.SlotMeta)}) end
        saveData(); renderCustom(); notify("Saved mix", true)
    end, THEME.Green)
    sc.CanvasSize = UDim2.new(0,0,0,y+70)
end

renderFavorites = function()
    setPage("Favorites"); renderTabs()
    local sc = scrollFrame(Gui.Body, UDim2.new(0,12,0,52), UDim2.new(1,-24,1,-90))
    local index=1
    for _, fav in ipairs(AppState.Favorites.Bundles) do renderItemCard(sc, fav, index, "Bundle"); index+=1 end
    for _, fav in ipairs(AppState.Favorites.Emotes) do renderItemCard(sc, fav, index, "Emote"); index+=1 end
    sc.CanvasSize = UDim2.new(0,0,0,math.max(360, math.ceil((index-1)/2)*142+62))
end

renderSave = function()
    setPage("Save"); renderTabs()
    local nameBox = textbox(Gui.Body, "Name save as...", UDim2.new(0,12,0,52), UDim2.new(0,220,0,36), "")
    button(Gui.Body, "SAVE CURRENT", UDim2.new(0,244,0,52), UDim2.new(0,140,0,36), function()
        local nm=tostring(nameBox.Text or ""); if nm=="" then nm="Saved Pack "..tostring(#AppState.SavedPacks+1) end
        table.insert(AppState.SavedPacks,{Name=nm,Form=tableCopy(AppState.CurrentForm),Meta=tableCopy(AppState.SlotMeta)}); saveData(); renderSave(); notify("Saved pack", true)
    end, THEME.Green)
    local sc=scrollFrame(Gui.Body, UDim2.new(0,12,0,100), UDim2.new(1,-24,1,-138)); local y=12
    for i, pack in ipairs(AppState.SavedPacks) do
        local row=panel(sc, UDim2.new(0,12,0,y), UDim2.new(1,-34,0,58), THEME.Card)
        local nm=tostring(pack.Name or ("Pack "..i)); label(row,nm..(AppState.AutoLoadName==nm and " [AUTO]" or ""),UDim2.new(0,10,0,5),UDim2.new(1,-250,0,22),14,THEME.Text)
        button(row,"AUTO",UDim2.new(1,-228,0,14),UDim2.new(0,52,0,30),function() AppState.AutoLoadName=nm; AppState.Settings.AutoLoad=true; saveData(); renderSave() end,AppState.AutoLoadName==nm and THEME.Green or THEME.Cyan)
        button(row,"USE",UDim2.new(1,-168,0,14),UDim2.new(0,48,0,30),function() AppState.CurrentForm=tableCopy(pack.Form); AppState.SlotMeta=tableCopy(pack.Meta); applyCurrentForm(nm) end,THEME.Orange)
        button(row,"EDIT",UDim2.new(1,-112,0,14),UDim2.new(0,52,0,30),function() AppState.CurrentForm=tableCopy(pack.Form); AppState.SlotMeta=tableCopy(pack.Meta); AppState.EditingSaveIndex=i; renderCustom() end,THEME.Cyan)
        button(row,"DEL",UDim2.new(1,-52,0,14),UDim2.new(0,40,0,30),function() table.remove(AppState.SavedPacks,i); saveData(); renderSave() end,THEME.Red)
        y+=66
    end
    sc.CanvasSize=UDim2.new(0,0,0,math.max(360,y+20))
end

renderSettings = function()
    setPage("Settings"); renderTabs()
    local sc = scrollFrame(Gui.Body, UDim2.new(0,12,0,52), UDim2.new(1,-24,1,-90))
    local y=12
    label(sc,"Picker",UDim2.new(0,12,0,y),UDim2.new(1,-24,0,24),15,THEME.Text); y+=32
    button(sc,"Provider: "..AppState.Settings.PickerProvider,UDim2.new(0,12,0,y),UDim2.new(0,180,0,32),function() AppState.Settings.PickerProvider=AppState.Settings.PickerProvider=="Floating buttons" and "Quick selector" or "Floating buttons"; saveData(); rebuildFloatingButtons(); rebuildQuickSelector(); renderSettings() end,THEME.Cyan)
    button(sc,"Float: "..AppState.Settings.FloatingMode,UDim2.new(0,204,0,y),UDim2.new(0,160,0,32),function() AppState.Settings.FloatingMode=AppState.Settings.FloatingMode=="Autogrid" and "Freeform" or "Autogrid"; saveData(); rebuildFloatingButtons(); renderSettings() end,THEME.Cyan); y+=44
    label(sc,"Emote Speed",UDim2.new(0,12,0,y),UDim2.new(1,-24,0,24),15,THEME.Text); y+=32
    local speedBox=textbox(sc,"Any speed",UDim2.new(0,12,0,y),UDim2.new(0,160,0,32),tostring(AppState.Settings.EmoteSpeed))
    button(sc,"APPLY",UDim2.new(0,184,0,y),UDim2.new(0,90,0,32),function() local n=tonumber(speedBox.Text); if n then AppState.Settings.EmoteSpeed=n; if AppState.Runtime.ActiveEmoteTrack then pcall(function() AppState.Runtime.ActiveEmoteTrack:AdjustSpeed(n) end) end; saveData(); renderSettings() end end,THEME.Green); y+=44
    button(sc,AppState.Settings.EmoteLoop and "Loop: ON" or "Loop: OFF",UDim2.new(0,12,0,y),UDim2.new(0,130,0,32),function() AppState.Settings.EmoteLoop=not AppState.Settings.EmoteLoop; saveData(); renderSettings() end,AppState.Settings.EmoteLoop and THEME.Green or THEME.Card)
    button(sc,AppState.Settings.MoveWhileEmote and "Move: ON" or "Move: OFF",UDim2.new(0,154,0,y),UDim2.new(0,130,0,32),function() AppState.Settings.MoveWhileEmote=not AppState.Settings.MoveWhileEmote; saveData(); renderSettings() end,AppState.Settings.MoveWhileEmote and THEME.Green or THEME.Card); y+=44
    label(sc,"Apply Method",UDim2.new(0,12,0,y),UDim2.new(1,-24,0,24),15,THEME.Text); y+=32
    for _, method in ipairs({"Animate","Description","Both"}) do
        button(sc,method,UDim2.new(0,12+(y%3)*120,0,y),UDim2.new(0,110,0,32),function() AppState.Settings.ApplyMethod=method; saveData(); renderSettings() end,AppState.Settings.ApplyMethod==method and THEME.Green or THEME.Card)
    end
    y+=44
    button(sc,AppState.Settings.ScreenBlur and "Blur: ON" or "Blur: OFF",UDim2.new(0,12,0,y),UDim2.new(0,130,0,32),function() AppState.Settings.ScreenBlur=not AppState.Settings.ScreenBlur; saveData(); renderSettings() end,AppState.Settings.ScreenBlur and THEME.Green or THEME.Card)
    button(sc,AppState.Settings.AvoidScaling and "No Scale: ON" or "No Scale: OFF",UDim2.new(0,154,0,y),UDim2.new(0,150,0,32),function() AppState.Settings.AvoidScaling=not AppState.Settings.AvoidScaling; saveData(); renderSettings() end,AppState.Settings.AvoidScaling and THEME.Green or THEME.Card); y+=44
    button(sc,"STOP EMOTE",UDim2.new(0,12,0,y),UDim2.new(0,130,0,32),stopCurrentEmote,THEME.Red)
    button(sc,"RESET ORIGINAL",UDim2.new(0,154,0,y),UDim2.new(0,150,0,32),restoreOriginalAnimations,THEME.Yellow)
    sc.CanvasSize=UDim2.new(0,0,0,y+60)
end

renderController = function()
    setPage("Controller"); renderTabs()
    label(Gui.Body,"Animation controller",UDim2.new(0,12,0,52),UDim2.new(1,-24,0,24),15,THEME.Text)
    label(Gui.Body,"Select track to control",UDim2.new(0,12,0,82),UDim2.new(1,-24,0,22),13,THEME.Muted)
    local tracks=getPlayingTracks()
    if #tracks==0 then label(Gui.Body,"No active animation tracks. Play an emote first.",UDim2.new(0,12,0,116),UDim2.new(1,-24,0,40),14,THEME.Muted); return end
    local y=116
    for i,track in ipairs(tracks) do
        local animId=track.Animation and track.Animation.AnimationId or "unknown"
        button(Gui.Body,(i==AppState.Controller.SelectedIndex and "● " or "○ ").."Track "..i,UDim2.new(0,12,0,y),UDim2.new(0,120,0,30),function() AppState.Controller.SelectedIndex=i; renderController() end,i==AppState.Controller.SelectedIndex and THEME.Green or THEME.Card)
        label(Gui.Body,animId,UDim2.new(0,142,0,y),UDim2.new(1,-160,0,30),11,THEME.Muted); y+=36
        if y>220 then break end
    end
    local track=select(1,getSelectedTrack())
    button(Gui.Body,AppState.Settings.ControllerLoop and "Looping: ON" or "Looping: OFF",UDim2.new(0,12,0,250),UDim2.new(0,140,0,32),function() AppState.Settings.ControllerLoop=not AppState.Settings.ControllerLoop; applyControllerToTrack(track); renderController() end,AppState.Settings.ControllerLoop and THEME.Green or THEME.Card)
    button(Gui.Body,AppState.Settings.ControllerReverse and "Reverse: ON" or "Reverse: OFF",UDim2.new(0,164,0,250),UDim2.new(0,140,0,32),function() AppState.Settings.ControllerReverse=not AppState.Settings.ControllerReverse; applyControllerToTrack(track); renderController() end,AppState.Settings.ControllerReverse and THEME.Green or THEME.Card)
    local x=12
    for _,sp in ipairs(SPEED_PRESETS) do
        button(Gui.Body,sp.Name,UDim2.new(0,x,0,300),UDim2.new(0,82,0,30),function() AppState.Settings.ControllerSpeedName=sp.Name; AppState.Settings.ControllerSpeed=sp.Value; applyControllerToTrack(track); renderController() end,AppState.Settings.ControllerSpeedName==sp.Name and THEME.Green or THEME.Card)
        x+=88
        if x>470 then x=12 end
    end
end

---------------------------------------------------------------------
-- GUI CREATE / BOOT
---------------------------------------------------------------------

local function createGui()
    local parent=getParentGui()
    pcall(function() local old=parent:FindFirstChild("FE_BUNDLE_REBUILT_CLEAN"); if old then old:Destroy() end end)
    Gui.Screen=createInstance("ScreenGui",{Name="FE_BUNDLE_REBUILT_CLEAN",ResetOnSpawn=false,IgnoreGuiInset=true,DisplayOrder=999999,ZIndexBehavior=Enum.ZIndexBehavior.Global})
    Gui.Screen.Parent=parent
    Gui.Icon=createInstance("TextButton",{Parent=Gui.Screen,Position=UDim2.new(0,18,0.5,-30),Size=UDim2.new(0,82,0,60),BackgroundColor3=THEME.Orange,BorderSizePixel=0,Text="OPEN",TextColor3=THEME.Text,TextSize=14,Font=Enum.Font.GothamBold,ZIndex=1000})
    addCorner(Gui.Icon,14); addStroke(Gui.Icon,THEME.Black,2,0)
    Gui.Main=createInstance("Frame",{Parent=Gui.Screen,AnchorPoint=Vector2.new(0.5,0.5),Position=UDim2.new(0.5,0,0.5,0),Size=UDim2.new(0,570,0,535),BackgroundColor3=THEME.Page,BorderSizePixel=0,Visible=not AppState.Settings.StartMenuClosed,Active=true,ZIndex=10})
    addCorner(Gui.Main,14); addStroke(Gui.Main,THEME.Black,2,0)
    local header=createInstance("Frame",{Parent=Gui.Main,Position=UDim2.new(0,0,0,0),Size=UDim2.new(1,0,0,58),BackgroundColor3=THEME.Header,BorderSizePixel=0,ZIndex=11})
    addCorner(header,14); createInstance("Frame",{Parent=header,Position=UDim2.new(0,0,1,-14),Size=UDim2.new(1,0,0,14),BackgroundColor3=THEME.Header,BorderSizePixel=0,ZIndex=11})
    Gui.HeaderTitle=label(Gui.Main,"FE Bundle",UDim2.new(0,18,0,8),UDim2.new(1,-88,0,28),21,THEME.Text)
    label(Gui.Main,"emotes, bundles, controller, shortcuts",UDim2.new(0,18,0,34),UDim2.new(1,-100,0,18),12,THEME.Muted)
    local close=createInstance("TextButton",{Parent=Gui.Main,Position=UDim2.new(1,-48,0,12),Size=UDim2.new(0,34,0,32),BackgroundColor3=THEME.Red,BorderSizePixel=0,Text="X",TextColor3=THEME.Text,Font=Enum.Font.GothamBold,TextSize=14,ZIndex=120})
    addCorner(close,8); addStroke(close,THEME.Black,1,0); addConnection(Connections.Global,close.MouseButton1Click:Connect(function() AppState.Alive=false; stopCurrentEmote(); disconnectBucket(Connections.Global); disconnectBucket(Connections.Page); if Gui.Screen then Gui.Screen:Destroy() end end))
    Gui.Body=createInstance("Frame",{Parent=Gui.Main,Position=UDim2.new(0,12,0,66),Size=UDim2.new(1,-24,1,-104),BackgroundTransparency=1,ZIndex=18})
    Gui.Status=label(Gui.Main,"Ready",UDim2.new(0,16,1,-34),UDim2.new(1,-32,0,24),12,THEME.Muted)
    Gui.Toast=createInstance("TextLabel",{Parent=Gui.Screen,Position=UDim2.new(0.5,-140,0,-42),Size=UDim2.new(0,280,0,30),BackgroundColor3=THEME.Page,BorderSizePixel=0,Text="",TextColor3=THEME.Text,TextSize=12,Font=Enum.Font.Gotham,Visible=false,ZIndex=300})
    addCorner(Gui.Toast,10); addStroke(Gui.Toast,THEME.Black,1,0.2)
    local dragHit=createInstance("TextButton",{Parent=Gui.Main,Position=UDim2.new(0,0,0,0),Size=UDim2.new(1,-58,0,58),BackgroundTransparency=1,Text="",BorderSizePixel=0,Active=true,AutoButtonColor=false,ZIndex=115})
    local dragging=false; local dragInput,dragStart,startPos
    addConnection(Connections.Global,dragHit.InputBegan:Connect(function(input) if input.UserInputType==Enum.UserInputType.Touch or input.UserInputType==Enum.UserInputType.MouseButton1 then dragging=true; dragInput=input; dragStart=input.Position; startPos=Gui.Main.Position; input.Changed:Connect(function() if input.UserInputState==Enum.UserInputState.End then dragging=false end end) end end))
    addConnection(Connections.Global,dragHit.InputChanged:Connect(function(input) if input.UserInputType==Enum.UserInputType.Touch or input.UserInputType==Enum.UserInputType.MouseMovement then dragInput=input end end))
    addConnection(Connections.Global,UserInputService.InputChanged:Connect(function(input) if dragging and input==dragInput then local d=input.Position-dragStart; Gui.Main.Position=UDim2.new(startPos.X.Scale,startPos.X.Offset+d.X,startPos.Y.Scale,startPos.Y.Offset+d.Y) end end))
    addConnection(Connections.Global,Gui.Icon.MouseButton1Click:Connect(function() Gui.Main.Visible=not Gui.Main.Visible end))
    renderHome("Emote")
end

loadData()
createGui()
setupCharacterLifecycle()
rebuildFloatingButtons()
rebuildQuickSelector()

task.spawn(function()
    task.wait(1)
    local hasAny=false
    for _,state in ipairs(STATES) do if normalizeId(AppState.CurrentForm[state])~="" then hasAny=true break end end
    if AppState.Settings.AutoLoad and hasAny then
        applyCurrentForm(AppState.LastAppliedName~="" and AppState.LastAppliedName or "Saved Pack")
        setStatus("Auto-loaded saved pack",true)
    else
        searchCatalog("Emote","dance",false)
        if Gui.Main and Gui.Main.Visible then renderHome("Emote") end
    end
end)
