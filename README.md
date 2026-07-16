-- Brainrot Finder - Desktop Application
-- Main Application Script

-- ============================================
-- MODULE: Configuration
-- ============================================
local Config = {
    api_url = "http://localhost:8080/api/brainrots",  -- Your API endpoint
    update_interval = 1,  -- seconds
    min_value = 100000000,  -- 100M
    alert_sound = "alert.wav",  -- Sound file path
    log_file = "brainrot_log.txt",
    max_history = 1000
}

-- ============================================
-- MODULE: Logger
-- ============================================
local Logger = {}
function Logger:new()
    local obj = {entries = {}}
    setmetatable(obj, self)
    self.__index = self
    return obj
end

function Logger:log(message)
    local timestamp = os.date("%Y-%m-%d %H:%M:%S")
    local log_entry = string.format("[%s] %s\n", timestamp, message)
    table.insert(self.entries, log_entry)
    
    -- Write to file
    local file = io.open(Config.log_file, "a")
    if file then
        file:write(log_entry)
        file:close()
    end
    
    -- Keep history limited
    if #self.entries > Config.max_history then
        table.remove(self.entries, 1)
    end
end

function Logger:getHistory()
    return self.entries
end

-- ============================================
-- MODULE: Audio Alert
-- ============================================
local AudioAlert = {}
function AudioAlert:play()
    -- Using system beep as fallback if sound file not available
    -- In a real implementation, you'd use a library like love.audio or system command
    if love and love.audio then
        -- If using LÖVE framework
        local sound = love.audio.newSource(Config.alert_sound, "static")
        love.audio.play(sound)
    else
        -- System beep
        os.execute('echo -e "\a"')
    end
end

-- ============================================
-- MODULE: Notification
-- ============================================
local Notification = {}
function Notification:show(title, message)
    -- Cross-platform notification
    local os_name = love.system.getOS() or "Windows"
    
    if os_name == "Windows" then
        os.execute(string.format('msg * "%s"', title .. " - " .. message))
    elseif os_name == "Mac OS X" then
        os.execute(string.format('osascript -e \'display notification "%s" with title "%s"\'', message, title))
    elseif os_name == "Linux" then
        os.execute(string.format('notify-send "%s" "%s"', title, message))
    else
        print(string.format("[NOTIFICATION] %s: %s", title, message))
    end
end

-- ============================================
-- MODULE: Brainrot Detector
-- ============================================
local BrainrotDetector = {}
BrainrotDetector.__index = BrainrotDetector

function BrainrotDetector:new()
    local obj = {
        detected_brainrots = {},  -- Key: job_id_brainrot_name
        history = {},
        is_running = false,
        timer = nil
    }
    setmetatable(obj, self)
    return obj
end

function BrainrotDetector:start()
    if self.is_running then return end
    self.is_running = true
    self:runDetection()
end

function BrainrotDetector:stop()
    self.is_running = false
    if self.timer then
        timer.cancel(self.timer)
        self.timer = nil
    end
end

function BrainrotDetector:runDetection()
    if not self.is_running then return end
    
    -- Async API call
    self:fetchBrainrots(function(success, data)
        if success then
            self:processBrainrots(data)
        end
    end)
    
    -- Schedule next run
    self.timer = timer.after(Config.update_interval, function()
        self:runDetection()
    end)
end

function BrainrotDetector:fetchBrainrots(callback)
    -- Simulated API call - Replace with actual HTTP request
    -- Using coroutine for async behavior
    local co = coroutine.create(function()
        -- In real implementation, use http.request or similar
        -- For demo, we'll simulate with mock data
        local mock_data = self:generateMockData()
        callback(true, mock_data)
    end)
    coroutine.resume(co)
end

function BrainrotDetector:generateMockData()
    -- Mock data generator for testing
    local brainrots = {}
    local names = {"AlphaBrain", "BetaRot", "GammaSpike", "DeltaWave", "EpsilonSurge"}
    local locations = {"US-East-1", "EU-West-2", "AP-South-1", "SA-East-1", "AF-South-1"}
    local job_ids = {"job-12345", "job-67890", "job-54321", "job-09876", "job-13579"}
    
    for i = 1, 5 do
        local value = math.random(50000000, 150000000) -- 50M to 150M
        if value >= Config.min_value then
            table.insert(brainrots, {
                name = names[math.random(#names)],
                value = value,
                location = locations[math.random(#locations)],
                job_id = job_ids[math.random(#job_ids)],
                timestamp = os.time()
            })
        end
    end
    return brainrots
end

function BrainrotDetector:processBrainrots(data)
    if not data or #data == 0 then return end
    
    for _, brainrot in ipairs(data) do
        if brainrot.value >= Config.min_value then
            self:handleNewBrainrot(brainrot)
        end
    end
end

function BrainrotDetector:handleNewBrainrot(brainrot)
    local key = brainrot.job_id .. "_" .. brainrot.name
    
    -- Check for duplicates
    if self.detected_brainrots[key] then
        -- Update existing entry
        self.detected_brainrots[key].last_updated = os.time()
        return
    end
    
    -- New brainrot detected
    local detection = {
        name = brainrot.name,
        value = brainrot.value,
        location = brainrot.location or "N/A",
        job_id = brainrot.job_id,
        timestamp = os.time(),
        last_updated = os.time(),
        status = "Active"
    }
    
    self.detected_brainrots[key] = detection
    table.insert(self.history, detection)
    
    -- Trigger alerts
    self:triggerAlerts(detection)
    
    -- Update UI
    if self.onBrainrotDetected then
        self.onBrainrotDetected(detection)
    end
end

function BrainrotDetector:triggerAlerts(detection)
    -- Log the detection
    local log_msg = string.format("BRAINROT DETECTED! Name: %s, Value: %d, Job ID: %s, Location: %s",
        detection.name, detection.value, detection.job_id, detection.location)
    Logger:log(log_msg)
    
    -- Play sound
    AudioAlert:play()
    
    -- Show notification
    local notification_msg = string.format(
        "Name: %s\nValue: %d\nLocation: %s\nJob ID: %s\nTime: %s",
        detection.name,
        detection.value,
        detection.location,
        detection.job_id,
        os.date("%Y-%m-%d %H:%M:%S", detection.timestamp)
    )
    Notification:show("🚨 New Brainrot Detected!", notification_msg)
end

function BrainrotDetector:getDetectedBrainrots()
    local results = {}
    for _, v in pairs(self.detected_brainrots) do
        table.insert(results, v)
    end
    return results
end

function BrainrotDetector:filterByValue(min_value)
    local results = {}
    for _, v in pairs(self.detected_brainrots) do
        if v.value >= min_value then
            table.insert(results, v)
        end
    end
    return results
end

function BrainrotDetector:searchByName(query)
    local results = {}
    for _, v in pairs(self.detected_brainrots) do
        if string.find(string.lower(v.name), string.lower(query)) then
            table.insert(results, v)
        end
    end
    return results
end

-- ============================================
-- MODULE: UI (LÖVE2D Implementation)
-- ============================================
local UI = {}
UI.__index = UI

function UI:new(detector)
    local obj = {
        detector = detector,
        window_width = 1024,
        window_height = 768,
        font = nil,
        scroll_offset = 0,
        selected_job_id = nil,
        filter_value = Config.min_value,
        search_query = "",
        show_history = false
    }
    setmetatable(obj, self)
    return obj
end

function UI:init()
    love.window.setTitle("Brainrot Finder")
    love.window.setMode(self.window_width, self.window_height, {
        resizable = true,
        minwidth = 800,
        minheight = 600
    })
    
    self.font = love.graphics.newFont(14)
    love.graphics.setFont(self.font)
    
    -- Set dark theme
    love.graphics.setBackgroundColor(0.12, 0.12, 0.15)
    
    -- Callback for new detections
    self.detector.onBrainrotDetected = function(detection)
        self:updateUI()
    end
end

function UI:updateUI()
    -- This will be called when new data arrives
end

function UI:draw()
    love.graphics.setColor(1, 1, 1)
    
    -- Title
    love.graphics.setColor(0.3, 0.8, 1)
    love.graphics.print("🧠 Brainrot Finder", 20, 10)
    love.graphics.setColor(0.6, 0.6, 0.6)
    love.graphics.print(string.format("Monitoring: %s | Threshold: %d", 
        Config.api_url, Config.min_value), 20, 40)
    
    -- Filter controls
    self:drawFilters()
    
    -- Table header
    self:drawTableHeader()
    
    -- Table content
    self:drawTableContent()
    
    -- Status bar
    self:drawStatusBar()
end

function UI:drawFilters()
    local y = 70
    love.graphics.setColor(0.2, 0.2, 0.25)
    love.graphics.rectangle("fill", 20, y, self.window_width - 40, 50)
    
    love.graphics.setColor(0.8, 0.8, 0.8)
    love.graphics.print("Min Value:", 30, y + 15)
    love.graphics.print(tostring(self.filter_value), 120, y + 15)
    
    love.graphics.print("Search:", 250, y + 15)
    love.graphics.print(self.search_query, 310, y + 15)
    
    -- Copy button
    if self.selected_job_id then
        love.graphics.setColor(0.2, 0.6, 0.2)
        love.graphics.rectangle("fill", 500, y + 10, 100, 30)
        love.graphics.setColor(1, 1, 1)
        love.graphics.print("Copy Job ID", 520, y + 18)
    end
end

function UI:drawTableHeader()
    local y = 130
    love.graphics.setColor(0.2, 0.2, 0.25)
    love.graphics.rectangle("fill", 20, y, self.window_width - 40, 30)
    
    love.graphics.setColor(0.8, 0.8, 0.8)
    local headers = {"Name", "Value", "Job ID", "Location", "Status", "Last Update"}
    local x_positions = {30, 200, 350, 550, 700, 850}
    
    for i, header in ipairs(headers) do
        love.graphics.print(header, x_positions[i], y + 8)
    end
end

function UI:drawTableContent()
    local y = 165
    local row_height = 25
    local brainrots = self.detector:getDetectedBrainrots()
    
    -- Sort by timestamp (newest first)
    table.sort(brainrots, function(a, b) 
        return a.timestamp > b.timestamp 
    end)
    
    -- Apply filters
    if self.search_query ~= "" then
        brainrots = self.detector:searchByName(self.search_query)
    end
    if self.filter_value > Config.min_value then
        brainrots = self.detector:filterByValue(self.filter_value)
    end
    
    -- Draw rows
    for i, brainrot in ipairs(brainrots) do
        local row_y = y + (i - 1) * row_height
        
        -- Alternate row colors
        if i % 2 == 0 then
            love.graphics.setColor(0.15, 0.15, 0.18)
        else
            love.graphics.setColor(0.18, 0.18, 0.22)
        end
        love.graphics.rectangle("fill", 20, row_y, self.window_width - 40, row_height)
        
        -- Highlight if selected
        if brainrot.job_id == self.selected_job_id then
            love.graphics.setColor(0.3, 0.3, 0.5, 0.5)
            love.graphics.rectangle("fill", 20, row_y, self.window_width - 40, row_height)
        end
        
        -- Text color based on value
        if brainrot.value >= 200000000 then
            love.graphics.setColor(1, 0.2, 0.2)  -- Critical
        elseif brainrot.value >= 150000000 then
            love.graphics.setColor(1, 0.6, 0)    -- Warning
        else
            love.graphics.setColor(0.8, 0.8, 0.8) -- Normal
        end
        
        -- Draw data
        local data = {
            brainrot.name,
            string.format("%d", brainrot.value),
            brainrot.job_id,
            brainrot.location,
            brainrot.status or "Active",
            os.date("%H:%M:%S", brainrot.last_updated or brainrot.timestamp)
        }
        
        local x_positions = {30, 200, 350, 550, 700, 850}
        for j, value in ipairs(data) do
            love.graphics.print(value, x_positions[j], row_y + 5)
        end
    end
end

function UI:drawStatusBar()
    local y = self.window_height - 40
    love.graphics.setColor(0.15, 0.15, 0.18)
    love.graphics.rectangle("fill", 0, y, self.window_width, 40)
    
    love.graphics.setColor(0.6, 0.6, 0.6)
    local brainrots = self.detector:getDetectedBrainrots()
    love.graphics.print(string.format("Total Detections: %d | History: %d", 
        #brainrots, #self.detector.history), 20, y + 12)
    
    love.graphics.print(string.format("Next update in: %.1fs", Config.update_interval), 
        self.window_width - 200, y + 12)
end

function UI:mousepressed(x, y, button)
    -- Handle clicking on table rows for Job ID selection
    if button == 1 then
        local brainrots = self.detector:getDetectedBrainrots()
        local y_start = 165
        local row_height = 25
        
        for i, brainrot in ipairs(brainrots) do
            local row_y = y_start + (i - 1) * row_height
            if x >= 20 and x <= self.window_width - 20 and 
               y >= row_y and y <= row_y + row_height then
                self.selected_job_id = brainrot.job_id
                -- Copy to clipboard
                love.system.setClipboardText(brainrot.job_id)
                Logger:log(string.format("Copied Job ID: %s", brainrot.job_id))
                break
            end
        end
    end
end

function UI:keypressed(key)
    if key == "escape" then
        self.selected_job_id = nil
    elseif key == "h" then
        self.show_history = not self.show_history
    end
end

-- ============================================
-- MAIN APPLICATION
-- ============================================
local detector = BrainrotDetector:new()
local logger = Logger:new()
local ui = UI:new(detector)

function love.load()
    ui:init()
    detector:start()
    Logger:log("Brainrot Finder started")
end

function love.update(dt)
    -- Update logic here
end

function love.draw()
    ui:draw()
end

function love.mousepressed(x, y, button)
    ui:mousepressed(x, y, button)
end

function love.keypressed(key)
    ui:keypressed(key)
end

function love.quit()
    detector:stop()
    Logger:log("Brainrot Finder stopped")
end

-- ============================================
-- CONFIGURATION MANAGEMENT
-- ============================================
function updateConfig(new_config)
    for key, value in pairs(new_config) do
        if Config[key] ~= nil then
            Config[key] = value
        end
    end
    Logger:log("Configuration updated")
end

-- Example usage:
-- updateConfig({min_value = 150000000, update_interval = 2})

return {
    Config = Config,
    Detector = BrainrotDetector,
    Logger = Logger,
    UI = UI
}
