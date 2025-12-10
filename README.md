loadstring([[
local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local CoreGui = game:GetService("CoreGui")
local RunService = game:GetService("RunService")

local LOCAL_PLAYER = Players.LocalPlayer
local PLACE_ID = game.PlaceId

local cfg = {
    minPlayers = 0,
    maxPlayers = 20,
    autoHop = false,
    hopDelay = 2,
    keywords = {"premium","chili","20m","20M","20 M","premium chili"},
    scanInterval = 1.5,
    avoidSameServer = true,
}

local state = {
    foundServers = {},
    markedGood = {},
    lastJobId = tostring(game.JobId or ""),
    scanning = false,
    stopAutoHop = false,
}

local function createGui()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "HopeChiliFinderGui"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = (LOCAL_PLAYER and LOCAL_PLAYER:FindFirstChildOfClass("PlayerGui")) or CoreGui

    local frame = Instance.new("Frame", screenGui)
    frame.Size = UDim2.new(0, 360, 0, 280)
    frame.Position = UDim2.new(0, 10, 0.5, -140)
    frame.BackgroundTransparency = 0.15
    frame.BackgroundColor3 = Color3.fromRGB(20,20,20)
    frame.BorderSizePixel = 0
    
    local title = Instance.new("TextLabel", frame)
    title.Text = "Hope Chili Finder"
    title.Size = UDim2.new(1, 0, 0, 30)
    title.Position = UDim2.new(0, 0, 0, 0)
    title.BackgroundColor3 = Color3.fromRGB(40,40,40)
    title.TextColor3 = Color3.fromRGB(255,255,255)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 18
    
    local closeBtn = Instance.new("TextButton", frame)
    closeBtn.Text = "X"
    closeBtn.Size = UDim2.new(0, 30, 0, 30)
    closeBtn.Position = UDim2.new(1, -30, 0, 0)
    closeBtn.BackgroundColor3 = Color3.fromRGB(200,50,50)
    closeBtn.TextColor3 = Color3.fromRGB(255,255,255)
    closeBtn.Font = Enum.Font.GothamBold
    
    local configFrame = Instance.new("Frame", frame)
    configFrame.Size = UDim2.new(1, -20, 0, 140)
    configFrame.Position = UDim2.new(0, 10, 0, 40)
    configFrame.BackgroundTransparency = 1
    
    local minLabel = Instance.new("TextLabel", configFrame)
    minLabel.Text = "Min Players:"
    minLabel.Size = UDim2.new(0, 80, 0, 20)
    minLabel.Position = UDim2.new(0, 0, 0, 0)
    minLabel.TextColor3 = Color3.fromRGB(220,220,220)
    minLabel.Font = Enum.Font.Gotham
    minLabel.TextSize = 14
    minLabel.BackgroundTransparency = 1
    
    local minBox = Instance.new("TextBox", configFrame)
    minBox.Text = tostring(cfg.minPlayers)
    minBox.Size = UDim2.new(0, 50, 0, 20)
    minBox.Position = UDim2.new(0, 85, 0, 0)
    minBox.BackgroundColor3 = Color3.fromRGB(40,40,40)
    minBox.TextColor3 = Color3.fromRGB(255,255,255)
    minBox.Font = Enum.Font.Gotham
    
    local maxLabel = Instance.new("TextLabel", configFrame)
    maxLabel.Text = "Max Players:"
    maxLabel.Size = UDim2.new(0, 80, 0, 20)
    maxLabel.Position = UDim2.new(0, 150, 0, 0)
    maxLabel.TextColor3 = Color3.fromRGB(220,220,220)
    maxLabel.Font = Enum.Font.Gotham
    maxLabel.TextSize = 14
    maxLabel.BackgroundTransparency = 1
    
    local maxBox = Instance.new("TextBox", configFrame)
    maxBox.Text = tostring(cfg.maxPlayers)
    maxBox.Size = UDim2.new(0, 50, 0, 20)
    maxBox.Position = UDim2.new(0, 235, 0, 0)
    maxBox.BackgroundColor3 = Color3.fromRGB(40,40,40)
    maxBox.TextColor3 = Color3.fromRGB(255,255,255)
    maxBox.Font = Enum.Font.Gotham
    
    local autoHopBtn = Instance.new("TextButton", configFrame)
    autoHopBtn.Text = "Auto Hop: OFF"
    autoHopBtn.Size = UDim2.new(0, 120, 0, 25)
    autoHopBtn.Position = UDim2.new(0, 0, 0, 30)
    autoHopBtn.BackgroundColor3 = Color3.fromRGB(60,60,60)
    autoHopBtn.TextColor3 = Color3.fromRGB(255,255,255)
    autoHopBtn.Font = Enum.Font.Gotham
    
    local scanBtn = Instance.new("TextButton", configFrame)
    scanBtn.Text = "Start Scan"
    scanBtn.Size = UDim2.new(0, 120, 0, 25)
    scanBtn.Position = UDim2.new(0, 130, 0, 30)
    scanBtn.BackgroundColor3 = Color3.fromRGB(60,60,60)
    scanBtn.TextColor3 = Color3.fromRGB(255,255,255)
    scanBtn.Font = Enum.Font.Gotham
    
    local keywordLabel = Instance.new("TextLabel", configFrame)
    keywordLabel.Text = "Keywords (comma):"
    keywordLabel.Size = UDim2.new(1, 0, 0, 20)
    keywordLabel.Position = UDim2.new(0, 0, 0, 65)
    keywordLabel.TextColor3 = Color3.fromRGB(220,220,220)
    keywordLabel.Font = Enum.Font.Gotham
    keywordLabel.TextSize = 14
    keywordLabel.BackgroundTransparency = 1
    
    local keywordBox = Instance.new("TextBox", configFrame)
    keywordBox.Text = table.concat(cfg.keywords, ",")
    keywordBox.Size = UDim2.new(1, 0, 0, 25)
    keywordBox.Position = UDim2.new(0, 0, 0, 85)
    keywordBox.BackgroundColor3 = Color3.fromRGB(40,40,40)
    keywordBox.TextColor3 = Color3.fromRGB(255,255,255)
    keywordBox.Font = Enum.Font.Gotham
    
    local serversFrame = Instance.new("ScrollingFrame", frame)
    serversFrame.Size = UDim2.new(1, -20, 0, 80)
    serversFrame.Position = UDim2.new(0, 10, 1, -90)
    serversFrame.BackgroundColor3 = Color3.fromRGB(30,30,30)
    serversFrame.BorderSizePixel = 0
    serversFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    
    local statusLabel = Instance.new("TextLabel", frame)
    statusLabel.Text = "Status: Ready"
    statusLabel.Size = UDim2.new(1, -20, 0, 20)
    statusLabel.Position = UDim2.new(0, 10, 1, -100)
    statusLabel.TextColor3 = Color3.fromRGB(220,220,220)
    statusLabel.Font = Enum.Font.Gotham
    statusLabel.TextSize = 14
    statusLabel.BackgroundTransparency = 1
    
    closeBtn.MouseButton1Click:Connect(function()
        screenGui:Destroy()
        state.scanning = false
        state.stopAutoHop = true
    end)
    
    minBox.FocusLost:Connect(function()
        local num = tonumber(minBox.Text)
        if num and num >= 0 then
            cfg.minPlayers = num
        else
            minBox.Text = tostring(cfg.minPlayers)
        end
    end)
    
    maxBox.FocusLost:Connect(function()
        local num = tonumber(maxBox.Text)
        if num and num > 0 then
            cfg.maxPlayers = num
        else
            maxBox.Text = tostring(cfg.maxPlayers)
        end
    end)
    
    autoHopBtn.MouseButton1Click:Connect(function()
        cfg.autoHop = not cfg.autoHop
        if cfg.autoHop then
            autoHopBtn.Text = "Auto Hop: ON"
            autoHopBtn.BackgroundColor3 = Color3.fromRGB(50, 150, 50)
            state.stopAutoHop = false
        else
            autoHopBtn.Text = "Auto Hop: OFF"
            autoHopBtn.BackgroundColor3 = Color3.fromRGB(60,60,60)
            state.stopAutoHop = true
        end
    end)
    
    keywordBox.FocusLost:Connect(function()
        local text = keywordBox.Text
        local keywords = {}
        for word in string.gmatch(text, "[^,]+") do
            table.insert(keywords, string.lower(string.trim(word)))
        end
        if #keywords > 0 then
            cfg.keywords = keywords
        end
    end)
    
    local function updateStatus(text, color)
        statusLabel.Text = "Status: " .. text
        statusLabel.TextColor3 = color or Color3.fromRGB(220,220,220)
    end
    
    local function addServerToList(serverId, players, foundText)
        local serverItem = Instance.new("Frame", serversFrame)
        serverItem.Size = UDim2.new(1, -10, 0, 25)
        serverItem.Position = UDim2.new(0, 5, 0, (#state.foundServers * 30))
        serverItem.BackgroundColor3 = Color3.fromRGB(40,40,40)
        serverItem.BorderSizePixel = 0
        
        local serverText = Instance.new("TextLabel", serverItem)
        serverText.Text = "Server: " .. string.sub(serverId, 1, 8) .. " | Players: " .. players
        serverText.Size = UDim2.new(0.7, 0, 1, 0)
        serverText.Position = UDim2.new(0, 5, 0, 0)
        serverText.TextColor3 = Color3.fromRGB(220,220,220)
        serverText.Font = Enum.Font.Gotham
        serverText.TextSize = 12
        serverText.BackgroundTransparency = 1
        serverText.TextXAlignment = Enum.TextXAlignment.Left
        
        local tpBtn = Instance.new("TextButton", serverItem)
        tpBtn.Text = "TP"
        tpBtn.Size = UDim2.new(0.2, 0, 0.7, 0)
        tpBtn.Position = UDim2.new(0.75, 0, 0.15, 0)
        tpBtn.BackgroundColor3 = Color3.fromRGB(50, 100, 200)
        tpBtn.TextColor3 = Color3.fromRGB(255,255,255)
        tpBtn.Font = Enum.Font.Gotham
        tpBtn.TextSize = 12
        
        tpBtn.MouseButton1Click:Connect(function()
            TeleportService:TeleportToPlaceInstance(PLACE_ID, serverId)
        end)
        
        serversFrame.CanvasSize = UDim2.new(0, 0, 0, (#state.foundServers * 30 + 30))
    end
    
    local function scanVisibleText()
        local found = false
        local lowercaseKeywords = {}
        for _, kw in ipairs(cfg.keywords) do
            table.insert(lowercaseKeywords, string.lower(kw))
        end
        
        local function scanInstance(instance)
            if instance:IsA("TextLabel") or instance:IsA("TextButton") or 
               instance:IsA("TextBox") or instance:IsA("TextLabel") then
                local text = string.lower(tostring(instance.Text))
                for _, keyword in ipairs(lowercaseKeywords) do
                    if string.find(text, keyword, 1, true) then
                        return true, instance.Text
                    end
                end
            end
            return false, nil
        end
        
        local function recursiveScan(parent)
            for _, child in ipairs(parent:GetChildren()) do
                local isFound, foundText = scanInstance(child)
                if isFound then
                    return true, foundText
                end
                local childFound, childText = recursiveScan(child)
                if childFound then
                    return true, childText
                end
            end
            return false, nil
        end
        
        local gui = LOCAL_PLAYER:FindFirstChildOfClass("PlayerGui")
        if gui then
            found, foundText = recursiveScan(gui)
        end
        
        return found, foundText
    end
    
    local function getServers()
        local servers = {}
        local success, result = pcall(function()
            return HttpService:JSONDecode(game:HttpGet(
                "https://games.roblox.com/v1/games/" .. PLACE_ID .. "/servers/Public?limit=100&cursor=",
                true
            ))
        end)
        
        if success and result and result.data then
            for _, server in ipairs(result.data) do
                if server.playing >= cfg.minPlayers and server.playing <= cfg.maxPlayers then
                    if not cfg.avoidSameServer or server.id ~= state.lastJobId then
                        table.insert(servers, {
                            id = server.id,
                            playing = server.playing
                        })
                    end
                end
            end
        end
        
        return servers
    end
    
    local function hopToServer(serverId)
        if not serverId then return end
        updateStatus("Teleporting...", Color3.fromRGB(255, 200, 50))
        TeleportService:TeleportToPlaceInstance(PLACE_ID, serverId)
    end
    
    scanBtn.MouseButton1Click:Connect(function()
        state.scanning = not state.scanning
        if state.scanning then
            scanBtn.Text = "Stop Scan"
            scanBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
            updateStatus("Scanning...", Color3.fromRGB(50, 200, 50))
            
            spawn(function()
                while state.scanning do
                    local found, foundText = scanVisibleText()
                    if found then
                        local jobId = tostring(game.JobId)
                        if not state.markedGood[jobId] then
                            state.markedGood[jobId] = true
                            table.insert(state.foundServers, {
                                id = jobId,
                                players = #Players:GetPlayers(),
                                text = foundText
                            })
                            
                            addServerToList(jobId, #Players:GetPlayers(), foundText)
                            updateStatus("Found: " .. foundText, Color3.fromRGB(50, 255, 50))
                        end
                    end
                    wait(cfg.scanInterval)
                end
            end)
        else
            scanBtn.Text = "Start Scan"
            scanBtn.BackgroundColor3 = Color3.fromRGB(60,60,60)
            updateStatus("Stopped", Color3.fromRGB(255, 100, 100))
        end
    end)
    
    spawn(function()
        while wait(1) do
            if cfg.autoHop and not state.stopAutoHop then
                local servers = getServers()
                if #servers > 0 then
                    local randomServer = servers[math.random(1, #servers)]
                    updateStatus("Hopping to new server...", Color3.fromRGB(255, 200, 50))
                    wait(cfg.hopDelay)
                    hopToServer(randomServer.id)
                else
                    updateStatus("No servers found", Color3.fromRGB(255, 100, 100))
                end
                wait(5)
            end
        end
    end)
    
    return screenGui
end

createGui()
]])()
