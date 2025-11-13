-- [[ 🔰 AutoFarm Test Script with UI by ChatGPT 🔰 ]]
-- สคริปต์นี้ใช้สำหรับทดสอบระบบออโต้ฟาร์มเปลี่ยนมอนอัตโนมัติ

local player = game.Players.LocalPlayer
player.leaderstats = player.leaderstats or Instance.new("Folder", player)
player.leaderstats.Name = "leaderstats"

-- สร้างตัวแปร Level จำลอง
local level = Instance.new("IntValue")
level.Name = "Level"
level.Value = 1
level.Parent = player.leaderstats

-- ===================== UI =====================
local ScreenGui = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
ScreenGui.Name = "AutoFarmTestUI"

local Frame = Instance.new("Frame", ScreenGui)
Frame.Size = UDim2.new(0, 220, 0, 140)
Frame.Position = UDim2.new(0.5, -110, 0.5, -70)
Frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
Frame.BorderSizePixel = 0
Frame.Visible = true
Frame.Active = true
Frame.Draggable = true

local Title = Instance.new("TextLabel", Frame)
Title.Size = UDim2.new(1, 0, 0, 30)
Title.BackgroundTransparency = 1
Title.Text = "🌀 AutoFarm Test"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 20

local ToggleButton = Instance.new("TextButton", Frame)
ToggleButton.Size = UDim2.new(1, -20, 0, 30)
ToggleButton.Position = UDim2.new(0, 10, 0, 40)
ToggleButton.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
ToggleButton.Text = "▶️ เริ่มฟาร์ม"
ToggleButton.TextColor3 = Color3.new(1, 1, 1)
ToggleButton.Font = Enum.Font.SourceSansBold
ToggleButton.TextSize = 18
ToggleButton.AutoButtonColor = true

local Status = Instance.new("TextLabel", Frame)
Status.Size = UDim2.new(1, -20, 0, 40)
Status.Position = UDim2.new(0, 10, 0, 80)
Status.BackgroundTransparency = 1
Status.TextColor3 = Color3.new(1, 1, 1)
Status.Font = Enum.Font.SourceSans
Status.TextSize = 16
Status.Text = "สถานะ: ❌ ยังไม่เริ่ม"

-- ===================== Logic =====================
local programRunning = false
local currentTarget = nil

-- ตารางมอนที่ฟาร์มตามเลเวล
local levelTargets = {
    [1] = "Lost Rider Lv.1",
    [5] = "Armed Lost Rider Lv.5"
}

local function getPlayerLevel()
    return level.Value
end

local function updateTargetByLevel()
    local lv = getPlayerLevel()
    local best = nil
    for reqLevel, mobName in pairs(levelTargets) do
        if lv >= reqLevel then
            best = mobName
        end
    end
    if best and best ~= currentTarget then
        currentTarget = best
        Status.Text = "🎯 ฟาร์ม: " .. currentTarget
        print("[AUTO FARM] เปลี่ยนเป้าหมายเป็น:", currentTarget)
    end
end

-- ระบบจำลองเวลอัปและเปลี่ยนมอน
local function startAutoFarm()
    programRunning = true
    ToggleButton.Text = "⏸️ หยุดฟาร์ม"
    Status.Text = "⚙️ กำลังเริ่มฟาร์ม..."
    print("🚀 เริ่มระบบออโต้ฟาร์ม")

    spawn(function()
        while programRunning do
            wait(2)
            level.Value += 1
            print("🧭 Level:", level.Value)
            updateTargetByLevel()

            if level.Value >= 7 then
                programRunning = false
                ToggleButton.Text = "▶️ เริ่มฟาร์ม"
                Status.Text = "✅ เควสเสร็จแล้ว!"
                print("✅ หยุดฟาร์ม (ถึงเวล 7)")
            end
        end
    end)
end

local function stopAutoFarm()
    programRunning = false
    ToggleButton.Text = "▶️ เริ่มฟาร์ม"
    Status.Text = "⏹️ หยุดฟาร์มแล้ว"
    print("🛑 หยุดระบบออโต้ฟาร์ม")
end

ToggleButton.MouseButton1Click:Connect(function()
    if programRunning then
        stopAutoFarm()
    else
        startAutoFarm()
    end
end)

print("✅ AutoFarm Test UI Loaded สำเร็จ!")
