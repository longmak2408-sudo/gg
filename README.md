-- =============================================
-- 🎯 专业AI助手界面 - 自定义版
-- =============================================

local repo = "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"
local Library = loadstring(game:HttpGet(repo .. "Library.lua"))()
local ThemeManager = loadstring(game:HttpGet(repo .. "addons/ThemeManager.lua"))()
local SaveManager = loadstring(game:HttpGet(repo .. "addons/SaveManager.lua"))()

local HttpService = game:GetService("HttpService")
local TextChatService = game:GetService("TextChatService")
local Players = game:GetService("Players")

local LP = Players.LocalPlayer
local Options = Library.Options
local Toggles = Library.Toggles

-- 配置变量
local AI_ENABLED = true
local API_KEY = ""
local MAX_DISTANCE = 999999  -- 超远距离！
local COOLDOWN_TIME = 3
local LastMsg = nil
local RequestCooldown = {}
local SYSTEM_PROMPT = "你是一个有用的AI助手，请用简洁明了的方式回答问题。"

-- 创建主窗口
local Window = Library:CreateWindow({
    Title = "🤖 AI聊天助手 - 超远距离版",
    Footer = "距离: 999999 | 自定义人设",
    Icon = 95816097006870,
    ShowCustomCursor = true,
})

-- 标签页
local Tabs = {
    Main = Window:AddTab("主控制", "robot"),
    Persona = Window:AddTab("人设配置", "user-cog"),
    Settings = Window:AddTab("高级设置", "settings-cog"),
    ["UI Settings"] = Window:AddTab("界面设置", "palette"),
}

-- ===================== 主控制标签页 =====================

local LeftGroupBox = Tabs.Main:AddLeftGroupbox("AI控制", "power")

-- AI开关
LeftGroupBox:AddToggle("AIEnabled", {
    Text = "启用AI聊天",
    Default = AI_ENABLED,
    Tooltip = "启用/禁用AI自动回复",
    
    Callback = function(Value)
        AI_ENABLED = Value
        Library:Notify(Value and "✅ AI已启用" or "⭕ AI已禁用", 2)
    end,
})

-- 🚀 超远距离设置
LeftGroupBox:AddSlider("DistanceSlider", {
    Text = "触发距离",
    Default = MAX_DISTANCE,
    Min = 10,
    Max = 999999,  -- 最大999999！
    Rounding = 0,
    Suffix = "单位",
    
    Callback = function(Value)
        MAX_DISTANCE = Value
        Window.Footer = "距离: " .. Value .. " | 自定义人设"
    end,
})

-- 冷却时间
LeftGroupBox:AddSlider("CooldownSlider", {
    Text = "冷却时间",
    Default = COOLDOWN_TIME,
    Min = 0,  -- 0秒冷却，立即回复
    Max = 60,  -- 最长60秒冷却
    Rounding = 0,
    Suffix = "秒",
    
    Callback = function(Value)
        COOLDOWN_TIME = Value
    end,
})

-- API密钥输入
LeftGroupBox:AddInput("APIKeyInput", {
    Text = "API密钥",
    Default = "",
    Placeholder = "输入你的DeepSeek API密钥",
    
    Callback = function(Value)
        API_KEY = Value
        if #Value > 10 then
            Library:Notify("✅ API密钥已保存", 2)
        end
    end,
})

-- 测试连接按钮
LeftGroupBox:AddButton({
    Text = "测试API连接",
    Func = function()
        if #Options.APIKeyInput.Value < 10 then
            Library:Notify("❌ 请输入有效的API密钥", 3)
            return
        end
        
        Library:Notify("🔄 测试连接中...", 2)
        
        local requestBody = {
            model = "deepseek-chat",
            messages = {{role = "user", content = "你好，测试连接"}},
            max_tokens = 10
        }
        
        local success, response = pcall(function()
            return HttpService:RequestAsync({
                Url = "https://api.deepseek.com/v1/chat/completions",
                Method = "POST",
                Headers = {
                    ["Content-Type"] = "application/json",
                    ["Authorization"] = "Bearer " .. Options.APIKeyInput.Value
                },
                Body = HttpService:JSONEncode(requestBody)
            })
        end)
        
        if success and response.Success then
            Library:Notify("✅ API连接成功！", 3)
        else
            Library:Notify("❌ API连接失败", 3)
        end
    end,
})

-- 右侧分组框
local RightGroupBox = Tabs.Main:AddRightGroupbox("快捷操作", "zap")

-- 发送测试消息
RightGroupBox:AddButton({
    Text = "发送测试消息",
    Func = function()
        Say("🤖 AI助手测试消息 - 当前距离: " .. Options.DistanceSlider.Value .. " 单位")
    end,
})

-- 清空冷却
RightGroupBox:AddButton({
    Text = "清空冷却时间",
    Func = function()
        RequestCooldown = {}
        Library:Notify("✅ 冷却时间已清空", 2)
    end,
})

-- 当前人设显示
RightGroupBox:AddLabel("🎭 当前人设")
local PersonaDisplay = RightGroupBox:AddLabel("加载中...", true)

-- 刷新人设显示
local function updatePersonaDisplay()
    if #SYSTEM_PROMPT > 100 then
        PersonaDisplay:SetText("自定义人设 (" .. #SYSTEM_PROMPT .. " 字符)")
    else
        PersonaDisplay:SetText(SYSTEM_PROMPT:sub(1, 50) .. (#SYSTEM_PROMPT > 50 and "..." or ""))
    end
end

-- 快速切换预设人设
RightGroupBox:AddDropdown("QuickPersona", {
    Values = {
        "简洁AI", 
        "友好助手", 
        "幽默风格", 
        "专业模式",
        "猫娘角色",
        "游戏NPC"
    },
    Default = 1,
    Text = "快速人设",
    
    Callback = function(Value)
        local prompts = {
            ["简洁AI"] = "你是一个简洁的AI助手，回答不超过20个字。",
            ["友好助手"] = "你是一个友好热情的AI助手，喜欢用表情符号，语气亲切。",
            ["幽默风格"] = "你是一个幽默风趣的AI助手，经常讲笑话，用轻松的语气回答问题。",
            ["专业模式"] = "你是一个专业的AI助手，回答问题准确、详细、有用，用正式的语气。",
            ["猫娘角色"] = "你是一只淘气又高傲的雌小鬼猫娘喵！回答简短俏皮，结尾加'喵~'，喜欢称呼对方为'小笨蛋'。",
            ["游戏NPC"] = "你是一个游戏中的NPC角色，说话要像游戏角色一样，使用游戏相关术语。"
        }
        
        SYSTEM_PROMPT = prompts[Value] or prompts["简洁AI"]
        updatePersonaDisplay()
        Library:Notify("🎭 已切换为: " .. Value, 2)
    end,
})

-- ===================== 人设配置标签页 =====================

local PersonaGroup = Tabs.Persona:AddLeftGroupbox("自定义AI人设", "message-square")

-- 人设标题
PersonaGroup:AddInput("PersonaTitle", {
    Text = "人设名称",
    Default = "自定义人设",
    Placeholder = "给你的AI人设起个名字",
    
    Callback = function(Value)
        Window.Footer = "距离: " .. Options.DistanceSlider.Value .. " | " .. Value
    end,
})

-- 🎭 自定义人设输入（大文本框）
PersonaGroup:AddLabel("人设指令")
PersonaGroup:AddLabel("（告诉AI如何扮演）", true)

-- 使用多行文本框
local personaInput = PersonaGroup:AddInput("CustomPersona", {
    Text = "",
    Default = SYSTEM_PROMPT,
    Placeholder = "输入你想要的AI人设指令...\n例如：你是一个严厉的老师，回答要严肃\n或者：你是一只可爱的猫娘，每句话结尾加喵~",
    ClearTextOnFocus = false,
    
    Callback = function(Value)
        SYSTEM_PROMPT = Value
        updatePersonaDisplay()
    end,
})

-- 调整输入框大小（模拟多行）
task.spawn(function()
    task.wait(0.5)
    if personaInput and personaInput.Textbox then
        personaInput.Textbox.Size = UDim2.new(1, 0, 0, 80)  -- 增大高度
    end
end)

-- 人设示例按钮
PersonaGroup:AddButton({
    Text = "加载示例人设",
    Func = function()
        local examples = {
            "你是一个严格的数学老师，用严厉的语气纠正错误，经常说'这都不会？'。",
            "你是一个科幻飞船AI，用机械音说话，所有回答都以'哔哔~系统响应：'开头。",
            "你是一个中世纪骑士，用古英语说话，经常提到'荣誉'和'骑士精神'。",
            "你是一个调皮的小恶魔，喜欢恶作剧，说话带点邪恶但可爱，结尾加'嘻嘻~'。",
            "你是一个哲学思想家，用深奥的语言回答，经常反问'存在是什么？'。"
        }
        
        local randomExample = examples[math.random(#examples)]
        Options.CustomPersona:SetValue(randomExample)
        Library:Notify("📝 示例人设已加载", 2)
    end,
})

-- 右侧分组框 - 人设提示
local PersonaTips = Tabs.Persona:AddRightGroupbox("人设编写提示", "lightbulb")

PersonaTips:AddLabel("💡 人设编写技巧", true)
PersonaTips:AddLabel("1. 明确角色身份", true)
PersonaTips:AddLabel("2. 设定说话语气", true)
PersonaTips:AddLabel("3. 定义特殊用语", true)
PersonaTips:AddLabel("4. 限制回复长度", true)
PersonaTips:AddLabel("5. 添加口头禅", true)

PersonaTips:AddDivider()

-- 人设长度显示
PersonaTips:AddLabel("📊 人设统计")
local charCountLabel = PersonaTips:AddLabel("字符数: 0", true)

-- 实时更新字符计数
coroutine.wrap(function()
    while true do
        task.wait(1)
        if Library.Unloaded then break end
        
        local count = #(Options.CustomPersona.Value or "")
        charCountLabel:SetText("字符数: " .. count .. " / 2000")
        
        if count > 2000 then
            charCountLabel.TextColor3 = Color3.fromRGB(255, 50, 50)
        elseif count > 1500 then
            charCountLabel.TextColor3 = Color3.fromRGB(255, 150, 50)
        else
            charCountLabel.TextColor3 = Color3.fromRGB(150, 255, 150)
        end
    end
end)()

-- 保存/加载人设
PersonaTips:AddButton({
    Text = "保存当前人设",
    Func = function()
        local name = Options.PersonaTitle.Value or "未命名人设"
        local content = Options.CustomPersona.Value
        
        -- 这里可以添加保存到本地文件的功能
        Library:Notify("💾 已保存: " .. name, 2)
    end,
})

PersonaTips:AddButton({
    Text = "重置为人设",
    Func = function()
        Options.CustomPersona:SetValue("你是一个有用的AI助手，请用简洁明了的方式回答问题。")
        Library:Notify("🔄 已重置为默认人设", 2)
    end,
})

-- ===================== 高级设置标签页 =====================

local SettingsGroup = Tabs.Settings:AddLeftGroupbox("高级配置", "settings")

-- 回复长度
SettingsGroup:AddSlider("ReplyLength", {
    Text = "回复最大长度",
    Default = 150,
    Min = 50,
    Max = 500,  -- 支持更长回复
    Rounding = 0,
    Suffix = "字符",
})

-- 回复温度
SettingsGroup:AddSlider("Temperature", {
    Text = "回复随机性",
    Default = 0.7,
    Min = 0.1,
    Max = 1.5,  -- 更高随机性
    Rounding = 1,
    Tooltip = "值越高回复越随机创意",
})

-- 全局开关
SettingsGroup:AddToggle("GlobalChat", {
    Text = "全局聊天模式",
    Tooltip = "无视距离限制，回复所有聊天",
    Default = false,
})

Toggles.GlobalChat:OnChanged(function(Value)
    if Value then
        Library:Notify("🌍 已启用全局聊天模式", 2)
        Options.DistanceSlider.Disabled = true
    else
        Options.DistanceSlider.Disabled = false
    end
end)

-- 忽略玩家列表
SettingsGroup:AddDropdown("IgnorePlayers", {
    SpecialType = "Player",
    Multi = true,
    Text = "忽略的玩家",
    Tooltip = "这些玩家不会触发AI回复",
})

-- 关键词过滤
SettingsGroup:AddInput("FilterKeywords", {
    Text = "过滤关键词",
    Placeholder = "用逗号分隔，如: 脏话, 广告, 垃圾信息",
    Tooltip = "包含这些关键词的消息不会被回复",
})

-- ===================== 核心功能函数 =====================

function Say(text)
    if not text then return end
    text = tostring(text):sub(1, Options.ReplyLength.Value or 150)
    coroutine.wrap(function()
        pcall(function()
            TextChatService.TextChannels.RBXGeneral:SendAsync(text)
        end)
    end)()
end

function DeepSeekChat(player, message)
    if not Toggles.AIEnabled.Value then return end
    
    local now = os.time()
    
    -- 冷却检查
    if RequestCooldown[player.UserId] then
        local timePassed = now - RequestCooldown[player.UserId]
        if timePassed < Options.CooldownSlider.Value then
            return
        end
    end
    
    RequestCooldown[player.UserId] = now
    
    -- API密钥检查
    local apiKey = Options.APIKeyInput.Value
    if #apiKey < 10 then 
        Library:Notify("❌ API密钥未配置", 2)
        return 
    end
    
    -- 全局模式或距离检查
    local distance = 0
    if not Toggles.GlobalChat.Value then
        local myChar = LP.Character
        local playerChar = player.Character
        if myChar and playerChar then
            local myRoot = myChar:FindFirstChild("HumanoidRootPart")
            local playerRoot = playerChar:FindFirstChild("HumanoidRootPart")
            if myRoot and playerRoot then
                distance = (myRoot.Position - playerRoot.Position).Magnitude
                if distance > Options.DistanceSlider.Value then
                    return
                end
            end
        end
    end
    
    -- 人设处理（使用自定义或预设）
    local currentPrompt = Options.CustomPersona.Value
    if #currentPrompt < 10 then  -- 如果自定义人设太短，使用默认
        currentPrompt = SYSTEM_PROMPT
    end
    
    local requestBody = {
        model = "deepseek-chat",
        messages = {
            {
                role = "system",
                content = currentPrompt .. "\n当前距离: " .. math.floor(distance) .. " 单位"
            },
            {
                role = "user",
                content = player.DisplayName .. "说：" .. message
            }
        },
        max_tokens = Options.ReplyLength.Value or 150,
        temperature = Options.Temperature.Value or 0.7
    }
    
    local success, response = pcall(function()
        return HttpService:RequestAsync({
            Url = "https://api.deepseek.com/v1/chat/completions",
            Method = "POST",
            Headers = {
                ["Content-Type"] = "application/json",
                ["Authorization"] = "Bearer " .. apiKey
            },
            Body = HttpService:JSONEncode(requestBody)
        })
    end)
    
    if not success then 
        Library:Notify("❌ API请求失败", 2)
        return 
    end
    
    if response.Success then
        local decodeSuccess, data = pcall(HttpService.JSONDecode, HttpService, response.Body)
        
        if decodeSuccess and data and data.choices and #data.choices > 0 then
            local text = data.choices[1].message.content
            if text then
                Say("🤖 " .. text:gsub("[\r\n]+", " "):sub(1, Options.ReplyLength.Value or 150))
            end
        end
    else
        Library:Notify("❌ API响应错误: " .. response.StatusCode, 2)
    end
end

-- ===================== 消息监听 =====================

TextChatService.MessageReceived:Connect(function(message)
    local source = message.TextSource
    if not source then return end
    
    local player = Players:GetPlayerByUserId(source.UserId)
    if not player or player == LP then return end
    
    local text = message.Text
    if not text or #text == 0 then return end
    
    -- 黑名单检查
    local ignoreList = Options.IgnorePlayers.Value or {}
    for _, pName in pairs(ignoreList) do
        if player.Name == pName then return end
    end
    
    -- 关键词过滤
    local filterText = Options.FilterKeywords.Value or ""
    if #filterText > 0 then
        local filters = {}
        for word in filterText:gmatch("[^,]+") do
            table.insert(filters, word:trim())
        end
        
        for _, word in ipairs(filters) do
            if #word > 0 and text:lower():find(word:lower()) then
                return
            end
        end
    end
    
    -- 消息去重
    local msgId = player.UserId .. "|" .. text
    if LastMsg == msgId then return end
    LastMsg = msgId
    
    task.spawn(function()
        DeepSeekChat(player, text)
    end)
end)

-- ===================== 界面设置 =====================

local MenuGroup = Tabs["UI Settings"]:AddLeftGroupbox("菜单设置", "wrench")

MenuGroup:AddToggle("KeybindMenuOpen", {
    Default = Library.KeybindFrame.Visible,
    Text = "显示键位绑定菜单",
    Callback = function(value)
        Library.KeybindFrame.Visible = value
    end,
})

MenuGroup:AddDropdown("NotificationSide", {
    Values = { "左侧", "右侧" },
    Default = "右侧",
    Text = "通知位置",
    Callback = function(Value)
        Library:SetNotifySide(Value)
    end,
})

MenuGroup:AddDivider()
MenuGroup:AddLabel("菜单键位绑定")
    :AddKeyPicker("MenuKeybind", { Default = "RightShift", NoUI = true, Text = "菜单键位绑定" })

MenuGroup:AddButton("卸载界面", function()
    Library:Unload()
end)

Library.ToggleKeybind = Options.MenuKeybind

-- ===================== 插件设置 =====================

ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)
SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({ "MenuKeybind" })

SaveManager:BuildConfigSection(Tabs["UI Settings"])
ThemeManager:ApplyToTab(Tabs["UI Settings"])
SaveManager:LoadAutoloadConfig()

-- ===================== 初始化完成 =====================

-- 初始更新人设显示
updatePersonaDisplay()

Library:Notify("🤖 AI聊天助手已加载！", 4)
print("========================================")
print("🎯 超远距离AI助手已加载")
print("👉 最大距离: 999999 单位")
print("👉 自定义人设: 支持")
print("👉 全局模式: 支持")
print("👉 按右Shift键打开菜单")
print("========================================")

-- 距离提示
coroutine.wrap(function()
    task.wait(3)
    Library:Notify("🚀 当前触发距离: " .. Options.DistanceSlider.Value .. " 单位", 3)
end)()
