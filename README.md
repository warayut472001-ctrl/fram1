-- AdminAllInOne.lua (LocalScript) - Full Version with All Features
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer

-- =================== Settings ===================
local programRunning = true
local livesFolderName = "Lives"
local targetMonsters = {
    "Lost Rider Lv.1","Armed Lost Rider Lv.5","Dragon User Lv.7",
    "Crab User Lv.10","Bat User Lv.12","Bull User Lv.15",
    "Foundation Soldier Lv.16","Cobra User Lv.20","Manta User Lv.25",
    "Swan User Lv.30","Zebra Monster Lv.35","Dark Dragon User Lv.40",
    "Gazelle User Lv.45","Violent Dragoon Lv.50","Shark Overloaded Lv.55",
    "Shark User Lv.60","Foundation Soldier Lv.64","Timeless Goon Lv.70",
    "Captain Goon Lv.75"
}

local selectedMonsters = {}
for _, m in ipairs(targetMonsters) do selectedMonsters[m] = true end

local state = {
    autoFarm=false, autoFarmInterval=0.1,
    lockPos=false,
    afkEnabled=false,
    autoBoss=false,
    autoDungeon=false,
    autoMine=false,
    autoAttack=false,
    autoHeavyAttack=false,
    autoKeyX=false,
    autoSkills={E=false, R=false, C=false, V=false},
    autoQuest=false,
    selectedQuest=nil,
    autoEvent=false
}

-- =================== Quest Database ===================
local questDatabase = {
    {
        id = "quest_nocap",
        name = "No CAP!",
        npcName = "Ryuga",
        npcPath = {"NPC", "Ryuga"},
        dialogueSteps = {"[ Repeatable Quest ]"},
        monsters = {"Utsumi Lv.80", "Mad Isurugi Lv.80"},
        questUIName = "No CAP!",
        questType = "kill",
        icon = "⚔️"
    },
    {
        id = "quest_agito",
        name = "Agito's Rules",
        npcName = "Shoichi",
        npcPath = {"NPC", "Shoichi"},
        dialogueSteps = {"[ Challenge ]", "[ Quest ]"},
        questUIName = "Agito's Rules",
        questType = "summon",
        summonLocation = {"KeyItem", "Spawn", "AgitoStone"},
        summonKey = "E",
        bossName = "Agito Lv.90",
        icon = "👹"
    }
}

local bp,bv,bg
local vu = game:GetService("VirtualUser")
player.Idled:Connect(function()
    if state.afkEnabled then
        pcall(function()
            vu:CaptureController()
            vu:ClickButton2(Vector2.new(0,0))
        end)
    end
end)

-- =================== Helper Functions ===================
local function getChar() return player.Character or player.CharacterAdded:Wait() end
local function getHRP() return getChar():FindFirstChild("HumanoidRootPart") end

local function getNearbyMonsters()
    local mobs = {}
    local livesFolder = workspace:FindFirstChild(livesFolderName)
    if livesFolder then
        for _, mob in ipairs(livesFolder:GetChildren()) do
            if selectedMonsters[mob.Name] and mob:FindFirstChild("HumanoidRootPart") then
                table.insert(mobs, mob)
            end
        end
    end
    local hrp = getHRP()
    if hrp then
        table.sort(mobs,function(a,b)
            return (hrp.Position - a.HumanoidRootPart.Position).Magnitude < (hrp.Position - b.HumanoidRootPart.Position).Magnitude
        end)
    end
    return mobs
end

local function getBoss()
    local livesFolder = workspace:FindFirstChild(livesFolderName)
    if not livesFolder then return nil end
    for _, mob in ipairs(livesFolder:GetChildren()) do
        if mob.Name == "Possessed Rider Lv.90" and mob:FindFirstChild("HumanoidRootPart") then
            return mob
        end
    end
    return nil
end

local function clickNPC(npc)
    if not npc then return false end
    local success = false
    pcall(function()
        for _, obj in pairs(npc:GetDescendants()) do
            if obj:IsA("ClickDetector") then
                if fireclickdetector then
                    fireclickdetector(obj)
                    success = true
                    break
                end
            end
        end
        if not success then
            for _, obj in pairs(npc:GetDescendants()) do
                if obj:IsA("ProximityPrompt") then
                    if fireproximityprompt then
                        fireproximityprompt(obj)
                        success = true
                        break
                    end
                end
            end
        end
        if not success then
            for _, obj in pairs(npc:GetDescendants()) do
                if obj:IsA("ClickDetector") then
                    obj.MouseClick:Fire(player)
                    success = true
                    break
                end
            end
        end
    end)
    return success
end

local function getNPCFromPath(path)
    local current = workspace
    for _, part in ipairs(path) do
        current = current:FindFirstChild(part)
        if not current then return nil end
    end
    return current
end

local function talkToNPC(questData)
    pcall(function()
        if questData.dialogueSteps then
            for i, step in ipairs(questData.dialogueSteps) do
                local args = {{Choice = step}}
                ReplicatedStorage:WaitForChild("Remote"):WaitForChild("Event"):WaitForChild("Dialogue"):FireServer(unpack(args))
                wait(0.5)
            end
        end
        local args2 = {{Exit = true}}
        ReplicatedStorage:WaitForChild("Remote"):WaitForChild("Event"):WaitForChild("Dialogue"):FireServer(unpack(args2))
    end)
end

local function submitQuest(questData, npc)
    pcall(function()
        clickNPC(npc)
        wait(0.5)
        if questData.dialogueSteps then
            for i, step in ipairs(questData.dialogueSteps) do
                local args = {{Choice = step}}
                ReplicatedStorage:WaitForChild("Remote"):WaitForChild("Event"):WaitForChild("Dialogue"):FireServer(unpack(args))
                wait(0.5)
            end
        end
        local args = {{Exit = true}}
        ReplicatedStorage:WaitForChild("Remote"):WaitForChild("Event"):WaitForChild("Dialogue"):FireServer(unpack(args))
    end)
end

local function hasActiveQuest(questData)
    local success, result = pcall(function()
        return player.PlayerGui.Main.QuestAlertFrame.QuestFrame.ScrollingFrame:FindFirstChild(questData.questUIName)
    end)
    return success and result ~= nil
end

local function getQuestMob(mobName)
    local livesFolder = workspace:FindFirstChild(livesFolderName)
    if not livesFolder then return nil end
    local mob = livesFolder:FindFirstChild(mobName)
    if mob and mob:FindFirstChild("HumanoidRootPart") and mob:FindFirstChild("Humanoid") and mob.Humanoid.Health > 0 then
        return mob
    end
    return nil
end

local function getSummonLocation(questData)
    local current = workspace
    for _, part in ipairs(questData.summonLocation) do
        current = current:FindFirstChild(part)
        if not current then return nil end
    end
    return current
end

local function pressSummonKey(key)
    pcall(function()
        vu:CaptureController()
        vu:SetKeyDown(string.lower(key))
        task.wait(0.1)
        vu:SetKeyUp(string.lower(key))
    end)
end

-- =================== Force Teleport Function ===================
local function forceTP(targetCFrame)
    local hrp = getHRP()
    if not hrp or not targetCFrame then 
        print("❌ [TP] ไม่พบ HRP หรือ Target")
        return false 
    end
    
    local char = getChar()
    print("🚀 [TP] กำลังวาร์ป...")
    
    for _, part in pairs(char:GetDescendants()) do
        if part:IsA("BasePart") then
            part.CanCollide = false
        end
    end
    
    for i = 1, 10 do
        hrp.CFrame = targetCFrame
        hrp.Velocity = Vector3.new(0, 0, 0)
        hrp.RotVelocity = Vector3.new(0, 0, 0)
        wait(0.05)
    end
    
    local distance = (hrp.Position - targetCFrame.Position).Magnitude
    print("📍 [TP] ระยะห่าง: " .. math.floor(distance) .. " studs")
    
    wait(0.3)
    for _, part in pairs(char:GetDescendants()) do
        if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" and part.Name ~= "Head" then
            part.CanCollide = true
        end
    end
    
    return distance < 30
end

-- =================== Auto Dungeon Loop ===================
spawn(function()
    while programRunning do
        if state.autoDungeon then
            print("\n🏛️ ============ Auto Dungeon - เริ่มรอบใหม่ ============")
            local hrp = getHRP()
            if not hrp then
                print("❌ [Dungeon] ไม่พบ HumanoidRootPart")
                wait(2)
                continue
            end
            
            print("🔍 [Dungeon] กำลังเปิด Dungeon UI...")
            
            -- ลองคลิกปุ่ม Dungeon ในเกมเพื่อเปิด UI
            local dungeonButtonClicked = false
            pcall(function()
                local mainGui = player.PlayerGui:FindFirstChild("Main")
                if mainGui then
                    local functionFrame = mainGui:FindFirstChild("FunctionFrame")
                    if functionFrame then
                        local dungeonBtn = functionFrame:FindFirstChild("Dungeon")
                        if dungeonBtn then
                            print("✅ [Dungeon] พบ " .. dungeonBtn.ClassName .. " ชื่อ Dungeon")
                            
                            -- ลองหาปุ่มที่คลิกได้จริงๆ
                            for _, child in ipairs(dungeonBtn:GetDescendants()) do
                                if (child:IsA("TextButton") or child:IsA("ImageButton")) and child.Visible then
                                    print("🔘 [Dungeon] พบปุ่มคลิกได้: " .. child.Name)
                                    for _, connection in pairs(getconnections(child.MouseButton1Click)) do
                                        connection:Fire()
                                        dungeonButtonClicked = true
                                    end
                                    if not dungeonButtonClicked then
                                        child.MouseButton1Click:Fire()
                                        dungeonButtonClicked = true
                                    end
                                    break
                                end
                            end
                            
                            -- ถ้ายังไม่ได้ ลองคลิก dungeonBtn เอง
                            if not dungeonButtonClicked and (dungeonBtn:IsA("TextButton") or dungeonBtn:IsA("ImageButton")) then
                                print("🔘 [Dungeon] ลองคลิกตัว Dungeon เอง")
                                for _, connection in pairs(getconnections(dungeonBtn.MouseButton1Click)) do
                                    connection:Fire()
                                    dungeonButtonClicked = true
                                end
                                if not dungeonButtonClicked then
                                    dungeonBtn.MouseButton1Click:Fire()
                                    dungeonButtonClicked = true
                                end
                            end
                        else
                            print("❌ [Dungeon] ไม่พบ Main.FunctionFrame.Dungeon")
                        end
                    else
                        print("❌ [Dungeon] ไม่พบ Main.FunctionFrame")
                    end
                else
                    print("❌ [Dungeon] ไม่พบ Main GUI")
                end
            end)
            
            if dungeonButtonClicked then
                print("✅ [Dungeon] คลิกปุ่ม Dungeon แล้ว - รอ UI โหลด...")
                wait(2)
            else
                print("⚠️ [Dungeon] ไม่พบปุ่ม Dungeon หรือคลิกไม่ได้ - ลองต่อ...")
                wait(1)
            end
            
            -- ลองหา DungeonGUI หลายครั้ง
            local dungeonGUI = nil
            for attempt = 1, 3 do
                dungeonGUI = player.PlayerGui:FindFirstChild("DungeonGUI")
                if dungeonGUI then 
                    print("✅ [Dungeon] พบ DungeonGUI!")
                    break 
                end
                print("⏳ [Dungeon] รอ DungeonGUI... (" .. attempt .. "/3)")
                wait(1)
            end
            
            if not dungeonGUI then
                print("❌ [Dungeon] ไม่พบ DungeonGUI - โปรดเปิดเมนู Dungeon ด้วยตัวเอง")
                print("💡 [Dungeon] กด Dungeon ในเกม แล้วปล่อยให้สคริปต์ทำงานต่อ")
                wait(5)
                continue
            end
            
            local dungeonList = dungeonGUI:FindFirstChild("Base")
            if dungeonList then
                dungeonList = dungeonList:FindFirstChild("DungeonList")
            end
            
            if not dungeonList then
                print("❌ [Dungeon] ไม่พบ Base.DungeonList")
                wait(3)
                continue
            end
            
            print("🔍 [Dungeon] กำลังหา Trial of Ethernal...")
            local trialDungeon = dungeonList:FindFirstChild("Trial of Ethernal")
            if not trialDungeon then
                print("❌ [Dungeon] ไม่พบ 'Trial of Ethernal' ใน DungeonList")
                print("📋 [Dungeon] รายการดันเจี้ยนที่มี:")
                for _, child in ipairs(dungeonList:GetChildren()) do
                    print("  - " .. child.Name)
                end
                wait(3)
                continue
            end
            
            print("✅ [Dungeon] พบ Trial of Ethernal!")
            
            -- หาปุ่ม Enter
            local enterButton = nil
            print("🔍 [Dungeon] กำลังหาปุ่ม Enter...")
            for _, child in ipairs(trialDungeon:GetDescendants()) do
                if child:IsA("TextButton") then
                    print("  - พบปุ่ม: " .. child.Name .. " (Text: '" .. (child.Text or "ไม่มี") .. "')")
                    if child.Text == "Enter" or child.Name == "Enter" or string.find(string.lower(child.Text), "enter") then
                        enterButton = child
                        print("✅ [Dungeon] เจอปุ่ม Enter แล้ว!")
                        break
                    end
                end
            end
            
            if not enterButton then
                print("❌ [Dungeon] ไม่พบปุ่ม Enter - ลองหาใหม่...")
                wait(3)
                continue
            end
            
            print("🚪 [Dungeon] กำลังกดปุ่ม Enter...")
            local clickSuccess = false
            pcall(function()
                -- วิธี 1: ใช้ FireServer
                if enterButton:IsA("TextButton") then
                    for _, connection in pairs(getconnections(enterButton.MouseButton1Click)) do
                        connection:Fire()
                        clickSuccess = true
                    end
                end
            end)
            
            if not clickSuccess then
                pcall(function()
                    -- วิธี 2: จำลองการคลิก
                    enterButton.MouseButton1Click:Fire()
                end)
            end
            
            wait(2)
            
            print("✅ [Dungeon] กำลังหาปุ่ม Confirm...")
            local confirmButton = nil
            local attempts = 0
            local maxAttempts = 10
            
            while attempts < maxAttempts and not confirmButton do
                for _, gui in ipairs(player.PlayerGui:GetChildren()) do
                    for _, child in ipairs(gui:GetDescendants()) do
                        if child:IsA("TextButton") and child.Visible then
                            if child.Text == "Confirm" or child.Name == "Confirm" or string.find(string.lower(child.Text), "confirm") then
                                confirmButton = child
                                print("✅ [Dungeon] เจอปุ่ม Confirm แล้ว! (" .. child.Text .. ")")
                                break
                            end
                        end
                    end
                    if confirmButton then break end
                end
                
                if not confirmButton then
                    attempts = attempts + 1
                    print("⏳ [Dungeon] รอปุ่ม Confirm... (" .. attempts .. "/" .. maxAttempts .. ")")
                    wait(0.5)
                end
            end
            
            if not confirmButton then
                print("❌ [Dungeon] ไม่พบปุ่ม Confirm - กลับไปเริ่มใหม่...")
                wait(2)
                continue
            end
            
            print("✅ [Dungeon] กำลังกดปุ่ม Confirm...")
            pcall(function()
                for _, connection in pairs(getconnections(confirmButton.MouseButton1Click)) do
                    connection:Fire()
                end
            end)
            
            pcall(function()
                confirmButton.MouseButton1Click:Fire()
            end)
            
            print("⏳ [Dungeon] รอเข้าดันเจี้ยน 30 วินาที...")
            for i = 30, 1, -1 do
                if not state.autoDungeon or not programRunning then break end
                if i % 10 == 0 then
                    print("⏰ [Dungeon] เหลืออีก " .. i .. " วินาที...")
                end
                wait(1)
            end
            
            print("🏛️ [Dungeon] เข้าดันเจี้ยนแล้ว - เริ่มหาบอส...")
            
            local livesFolder = workspace:FindFirstChild(livesFolderName)
            if not livesFolder then
                print("❌ [Dungeon] ไม่พบ Lives folder")
                wait(5)
                continue
            end
            
            local bossFound = false
            local searchAttempts = 0
            local maxSearchAttempts = 90
            
            print("🔍 [Dungeon] กำลังค้นหาบอส Ethernal Lv.90...")
            while programRunning and state.autoDungeon and searchAttempts < maxSearchAttempts do
                local boss = livesFolder:FindFirstChild("Ethernal Lv.90")
                
                if boss and boss:FindFirstChild("HumanoidRootPart") and boss:FindFirstChild("Humanoid") then
                    if boss.Humanoid.Health > 0 then
                        bossFound = true
                        print("✅ [Dungeon] พบบอส Ethernal Lv.90! 🎯")
                        
                        local bossHRP = boss:FindFirstChild("HumanoidRootPart")
                        local bossHumanoid = boss:FindFirstChild("Humanoid")
                        
                        print("🚀 [Dungeon] วาร์ปไปหาบอส...")
                        local bossCFrame = bossHRP.CFrame * CFrame.new(0, 0, 10)
                        forceTP(bossCFrame)
                        wait(1.5)
                        
                        print("⚔️ [Dungeon] เริ่มโจมตีบอส! HP: " .. math.floor(bossHumanoid.Health))
                        local lastHPReport = os.time()
                        
                        while programRunning and state.autoDungeon and boss.Parent and bossHumanoid.Health > 0 do
                            if bossHRP and bossHRP.Parent then
                                local backDistance = 3
                                local backPos = bossHRP.Position - bossHRP.CFrame.LookVector * backDistance
                                
                                pcall(function()
                                    hrp = getHRP()
                                    if hrp then
                                        hrp.CFrame = CFrame.new(backPos, Vector3.new(bossHRP.Position.X, backPos.Y, bossHRP.Position.Z))
                                    end
                                end)
                                
                                -- รายงาน HP ทุก 5 วิ
                                if os.time() - lastHPReport >= 5 then
                                    print("💥 [Dungeon] Boss HP: " .. math.floor(bossHumanoid.Health))
                                    lastHPReport = os.time()
                                end
                            else
                                break
                            end
                            
                            wait(0.15)
                        end
                        
                        print("✅ [Dungeon] ฆ่าบอสสำเร็จ! 🎉")
                        break
                    end
                end
                
                if searchAttempts % 10 == 0 and searchAttempts > 0 then
                    print("⏳ [Dungeon] ยังไม่พบบอส... (" .. searchAttempts .. "/" .. maxSearchAttempts .. ")")
                end
                
                searchAttempts = searchAttempts + 1
                wait(1)
            end
            
            if not bossFound then
                print("⚠️ [Dungeon] ไม่พบบอสภายใน " .. maxSearchAttempts .. " วินาที")
                print("💡 [Dungeon] อาจต้องเข้าดันเจี้ยนด้วยตัวเอง หรือเช็คว่าอยู่ในห้องบอสหรือไม่")
            end
            
            print("✅ [Dungeon] รอบนี้เสร็จแล้ว - รอ 5 วิแล้วเริ่มใหม่...")
            wait(5)
        else
            wait(1)
        end
    end
end)
-- =================== Auto Event Halloween Loop ===================
spawn(function()
    while programRunning do
        if state.autoEvent then
            print("\n🎃 ============ เริ่มรอบใหม่ ============")
            local hrp = getHRP()
            if not hrp then
                print("❌ [Halloween] ไม่พบ HumanoidRootPart")
                wait(2)
                continue
            end
            
            print("🔍 [Halloween] กำลังค้นหากล่อง...")
            local chest = workspace:FindFirstChild("KeyItem")
            if chest then
                chest = chest:FindFirstChild("Halloween Chest")
            end
            
            if not chest then
                print("❌ [Halloween] ไม่พบกล่อง 'Halloween Chest' - รอ 5 วินาที...")
                wait(5)
                continue
            end
            
            print("✅ [Halloween] พบกล่องแล้ว!")
            
            local chestCFrame = nil
            if chest:IsA("Model") then
                local primary = chest.PrimaryPart or chest:FindFirstChild("HumanoidRootPart") or chest:FindFirstChildWhichIsA("BasePart")
                if primary then
                    chestCFrame = primary.CFrame * CFrame.new(0, 5, 5)
                end
            elseif chest:IsA("BasePart") then
                chestCFrame = chest.CFrame * CFrame.new(0, 5, 5)
            end
            
            if not chestCFrame then
                print("❌ [Halloween] ไม่สามารถหา CFrame ของกล่องได้")
                wait(5)
                continue
            end
            
            print("🚀 [Halloween] วาร์ปไปที่กล่อง...")
            local tpSuccess = forceTP(chestCFrame)
            
            if not tpSuccess then
                print("⚠️ [Halloween] วาร์ปครั้งแรกไม่สำเร็จ - ลองอีกครั้ง...")
                wait(1)
                forceTP(chestCFrame)
            end
            
            wait(1.5)
            
            print("🔮 [Halloween] กด E เพื่อเปิดกล่อง...")
            pressSummonKey("E")
            wait(3)
            
            print("⏳ [Halloween] รอมอนเกิด...")
            wait(2)
            
            local livesFolder = workspace:FindFirstChild(livesFolderName)
            if livesFolder then
                local function getHallowedGoons()
                    local goons = {}
                    for _, mob in ipairs(livesFolder:GetChildren()) do
                        if mob.Name == "Hollowed Goon Lv.80" and mob:FindFirstChild("HumanoidRootPart") and mob:FindFirstChild("Humanoid") then
                            if mob.Humanoid.Health > 0 then
                                table.insert(goons, mob)
                            end
                        end
                    end
                    return goons
                end
                
                local waitTime = 0
                local maxWait = 10
                while waitTime < maxWait do
                    local currentGoons = getHallowedGoons()
                    if #currentGoons >= 1 then
                        print("✅ [Halloween] พบมอน " .. #currentGoons .. " ตัว - เริ่มฆ่า!")
                        break
                    end
                    wait(1)
                    waitTime = waitTime + 1
                end
                
                local killAttempts = 0
                local maxKillAttempts = 100
                
                while programRunning and state.autoEvent and killAttempts < maxKillAttempts do
                    local goons = getHallowedGoons()
                    
                    if #goons == 0 then
                        print("✅ [Halloween] ฆ่ามอนหมดแล้ว!")
                        break
                    end
                    
                    table.sort(goons, function(a, b)
                        local distA = (hrp.Position - a.HumanoidRootPart.Position).Magnitude
                        local distB = (hrp.Position - b.HumanoidRootPart.Position).Magnitude
                        return distA < distB
                    end)
                    
                    local targetGoon = goons[1]
                    if targetGoon and targetGoon.Parent then
                        local mobHRP = targetGoon:FindFirstChild("HumanoidRootPart")
                        local mobHumanoid = targetGoon:FindFirstChild("Humanoid")
                        
                        if mobHRP and mobHumanoid and mobHumanoid.Health > 0 then
                            local backDistance = 3
                            while programRunning and state.autoEvent and targetGoon.Parent and mobHumanoid.Health > 0 do
                                local desired = mobHRP.Position - mobHRP.CFrame.LookVector * backDistance
                                pcall(function()
                                    hrp.CFrame = CFrame.new(desired, Vector3.new(mobHRP.Position.X, desired.Y, mobHRP.Position.Z))
                                end)
                                wait(0.15)
                            end
                            print("⚔️ [Halloween] ฆ่ามอน 1 ตัวแล้ว - เหลืออีก " .. (#goons - 1) .. " ตัว")
                            wait(0.5)
                        end
                    end
                    
                    killAttempts = killAttempts + 1
                    wait(0.1)
                end
                
                wait(2)
                print("🎁 [Halloween] กำลังวาร์ปกลับไปรับของ...")
                
                local chestAgain = workspace:FindFirstChild("KeyItem")
                if chestAgain then
                    chestAgain = chestAgain:FindFirstChild("Halloween Chest")
                end
                
                if chestAgain then
                    local chestCFrame2 = nil
                    if chestAgain:IsA("Model") then
                        local primary = chestAgain.PrimaryPart or chestAgain:FindFirstChild("HumanoidRootPart") or chestAgain:FindFirstChildWhichIsA("BasePart")
                        if primary then
                            chestCFrame2 = primary.CFrame * CFrame.new(0, 5, 5)
                        end
                    elseif chestAgain:IsA("BasePart") then
                        chestCFrame2 = chestAgain.CFrame * CFrame.new(0, 5, 5)
                    end
                    
                    if chestCFrame2 then
                        local tpSuccess2 = forceTP(chestCFrame2)
                        if not tpSuccess2 then
                            print("⚠️ [Halloween] วาร์ปกลับไม่สำเร็จ - ลองอีกครั้ง...")
                            wait(1)
                            forceTP(chestCFrame2)
                        end
                        wait(1.5)
                        
                        print("🎁 [Halloween] กด E เพื่อรับของ...")
                        pressSummonKey("E")
                        wait(2)
                        
                        print("✅ [Halloween] รอบนี้เสร็จแล้ว! เริ่มรอบใหม่ใน 3 วินาที...")
                        wait(3)
                    else
                        print("❌ [Halloween] ไม่พบ CFrame ของกล่อง")
                        wait(3)
                    end
                else
                    print("❌ [Halloween] ไม่พบกล่องเมื่อจะรับของ")
                    wait(3)
                end
            end
        else
            wait(1)
        end
    end
end)

-- =================== AutoQuest Loop ===================
spawn(function()
    while programRunning do
        if state.autoQuest and state.selectedQuest then
            local questData = state.selectedQuest
            local hrp = getHRP()
            if hrp then
                local npc = getNPCFromPath(questData.npcPath)
                if npc and npc:FindFirstChild("HumanoidRootPart") then
                    pcall(function()
                        hrp.CFrame = npc.HumanoidRootPart.CFrame * CFrame.new(0, 0, 4)
                    end)
                    wait(1)
                    clickNPC(npc)
                    wait(1)
                    talkToNPC(questData)
                    wait(2)
                    local questReceived = false
                    for i = 1, 5 do
                        if hasActiveQuest(questData) then
                            questReceived = true
                            break
                        else
                            wait(1)
                        end
                    end
                    if not questReceived then
                        wait(3)
                        continue
                    end
                    wait(1)
                    if questData.questType == "summon" then
                        local summonSpot = getSummonLocation(questData)
                        if summonSpot then
                            pcall(function()
                                hrp.CFrame = summonSpot.CFrame * CFrame.new(0, 3, 0)
                            end)
                            wait(1.5)
                            pressSummonKey(questData.summonKey)
                            wait(2)
                            local bossFound = false
                            local attempts = 0
                            local maxAttempts = 100
                            while programRunning and state.autoQuest and not bossFound and attempts < maxAttempts do
                                local boss = getQuestMob(questData.bossName)
                                if boss then
                                    bossFound = true
                                    local bossHRP = boss:FindFirstChild("HumanoidRootPart")
                                    local bossHumanoid = boss:FindFirstChild("Humanoid")
                                    while programRunning and state.autoQuest and boss.Parent and bossHumanoid and bossHumanoid.Health > 0 do
                                        if bossHRP then
                                            local backDistance = 3
                                            local backPos = bossHRP.Position - bossHRP.CFrame.LookVector * backDistance
                                            pcall(function()
                                                hrp.CFrame = CFrame.new(backPos, Vector3.new(bossHRP.Position.X, backPos.Y, bossHRP.Position.Z))
                                            end)
                                            wait(0.15)
                                        else
                                            break
                                        end
                                    end
                                    break
                                else
                                    attempts = attempts + 1
                                    wait(1)
                                end
                            end
                            if not bossFound then
                                wait(3)
                                continue
                            end
                        else
                            wait(3)
                            continue
                        end
                    elseif questData.questType == "kill" then
                        local killedMobs = {}
                        for _, mobName in ipairs(questData.monsters) do
                            local mobFound = false
                            local attempts = 0
                            local maxAttempts = 150
                            while programRunning and state.autoQuest and not killedMobs[mobName] and attempts < maxAttempts do
                                local mob = getQuestMob(mobName)
                                if not mob then
                                    if mobFound then
                                        killedMobs[mobName] = true
                                        break
                                    else
                                        attempts = attempts + 1
                                        wait(1)
                                    end
                                else
                                    mobFound = true
                                    local mobHRP = mob:FindFirstChild("HumanoidRootPart")
                                    local mobHumanoid = mob:FindFirstChild("Humanoid")
                                    if mobHRP and mobHumanoid and mobHumanoid.Health > 0 then
                                        local backDistance = 3
                                        local backPos = mobHRP.Position - mobHRP.CFrame.LookVector * backDistance
                                        pcall(function()
                                            hrp.CFrame = CFrame.new(backPos, Vector3.new(mobHRP.Position.X, backPos.Y, mobHRP.Position.Z))
                                        end)
                                        wait(0.15)
                                    else
                                        killedMobs[mobName] = true
                                        break
                                    end
                                end
                                wait(0.1)
                            end
                            if killedMobs[mobName] then
                                wait(2)
                            end
                        end
                    end
                    wait(2)
                    pcall(function()
                        hrp.CFrame = npc.HumanoidRootPart.CFrame * CFrame.new(0, 0, 4)
                    end)
                    wait(1.5)
                    submitQuest(questData, npc)
                    wait(2)
                    local questSubmitted = false
                    for i = 1, 5 do
                        if not hasActiveQuest(questData) then
                            questSubmitted = true
                            break
                        else
                            wait(1)
                        end
                    end
                    if questSubmitted then
                        wait(3)
                    else
                        wait(2)
                    end
                end
            end
        else
            if state.autoQuest and not state.selectedQuest then
                state.autoQuest = false
            end
        end
        wait(1)
    end
end)

-- =================== AutoFarm Loop ===================
spawn(function()
    while programRunning do
        if state.autoFarm then
            local char = getChar()
            local event = char:FindFirstChild("PlayerHandler") and char.PlayerHandler:FindFirstChild("HandlerEvent")
            if event then
                local mobs = getNearbyMonsters()
                if #mobs > 0 then
                    local target = mobs[1]
                    local thrp = target:FindFirstChild("HumanoidRootPart")
                    if thrp then
                        local backDistance = 3
                        local backPos = thrp.Position - thrp.CFrame.LookVector * backDistance
                        local hrp = getHRP()
                        if hrp then
                            pcall(function()
                                hrp.CFrame = CFrame.new(backPos, Vector3.new(thrp.Position.X, backPos.Y, thrp.Position.Z))
                            end)
                        end
                        wait(0.05)
                        while programRunning and state.autoFarm and target.Parent and target:FindFirstChild("Humanoid") and target.Humanoid.Health > 0 do
                            local hrp2 = getHRP()
                            local thrp2 = target:FindFirstChild("HumanoidRootPart")
                            if not (hrp2 and thrp2) then break end
                            pcall(function()
                                local desired = thrp2.Position - thrp2.CFrame.LookVector * backDistance
                                hrp2.CFrame = CFrame.new(desired, Vector3.new(thrp2.Position.X, desired.Y, thrp2.Position.Z))
                            end)
                            wait(state.autoFarmInterval)
                        end
                    end
                end
            end
        end
        wait(0.1)
    end
end)

-- =================== Auto Boss Loop ===================
spawn(function()
    while programRunning do
        if state.autoBoss then
            local hrp = getHRP()
            if hrp then
                local rushZone = workspace.MAP:FindFirstChild("Trial") and workspace.MAP.Trial:FindFirstChild("Rush - Zone")
                if rushZone then
                    pcall(function()
                        hrp.CFrame = rushZone.CFrame + Vector3.new(0,5,0)
                    end)
                    wait(0.5)
                end
                local boss = getBoss()
                if boss and boss.Parent and boss:FindFirstChild("Humanoid") and boss.Humanoid.Health > 0 then
                    local bossHRP = boss:FindFirstChild("HumanoidRootPart")
                    if bossHRP then
                        local backDistance = 3
                        local backPos = bossHRP.Position - bossHRP.CFrame.LookVector*backDistance
                        pcall(function()
                            hrp.CFrame = CFrame.new(backPos, Vector3.new(bossHRP.Position.X, backPos.Y, bossHRP.Position.Z))
                        end)
                        wait(0.05)
                        while programRunning and state.autoBoss and boss.Parent and boss.Humanoid.Health > 0 do
                            local desired = bossHRP.Position - bossHRP.CFrame.LookVector*backDistance
                            pcall(function()
                                hrp.CFrame = CFrame.new(desired, Vector3.new(bossHRP.Position.X, desired.Y, bossHRP.Position.Z))
                            end)
                            wait(0.1)
                        end
                    end
                else
                    wait(1)
                end
            end
        end
        wait(0.1)
    end
end)

-- =================== Auto Mine Loop ===================
spawn(function()
    while programRunning do
        if state.autoMine then
            local hrp = getHRP()
            local livesFolder = workspace:FindFirstChild(livesFolderName)
            if hrp and livesFolder then
                local miners = {}
                for _, mob in ipairs(livesFolder:GetChildren()) do
                    if mob.Name=="Miner Goon Lv.50" and mob:FindFirstChild("HumanoidRootPart") then
                        table.insert(miners,mob)
                    end
                end
                table.sort(miners,function(a,b)
                    return (hrp.Position - a.HumanoidRootPart.Position).Magnitude < (hrp.Position - b.HumanoidRootPart.Position).Magnitude
                end)
                local targetMiners={}
                for i=1,math.min(3,#miners) do table.insert(targetMiners,miners[i]) end
                local distance = 5
                local spacing = 1.5
                for i,mob in ipairs(targetMiners) do
                    local mobHRP = mob:FindFirstChild("HumanoidRootPart")
                    if mobHRP then
                        mobHRP.Anchored=false
                        local offset = Vector3.new((i-1)*spacing,0,0)
                        mobHRP.CFrame = CFrame.new(hrp.Position + hrp.CFrame.LookVector * distance + offset)
                    end
                end
                for _, mob in ipairs(targetMiners) do
                    local mobHRP = mob:FindFirstChild("HumanoidRootPart")
                    local humanoid = mob:FindFirstChild("Humanoid")
                    if mobHRP and humanoid and humanoid.Health>0 then
                        local backDistance = 3
                        local backPos = mobHRP.Position - mobHRP.CFrame.LookVector*backDistance
                        pcall(function() hrp.CFrame=CFrame.new(backPos,Vector3.new(mobHRP.Position.X,backPos.Y,mobHRP.Position.Z)) end)
                        wait(0.05)
                        while programRunning and state.autoMine and mob.Parent and humanoid.Health>0 do
                            local desired = mobHRP.Position - mobHRP.CFrame.LookVector*backDistance
                            pcall(function() hrp.CFrame=CFrame.new(desired,Vector3.new(mobHRP.Position.X,desired.Y,mobHRP.Position.Z)) end)
                            wait(0.1)
                        end
                    end
                end
            end
        end
        wait(0.1)
    end
end)

-- =================== Auto Light Attack Loop ===================
spawn(function()
    while programRunning do
        if state.autoAttack then
            local char = getChar()
            local event = char:FindFirstChild("PlayerHandler") and char.PlayerHandler:FindFirstChild("HandlerEvent")
            if event then
                local args = {
                    {
                        CombatAction = true,
                        MouseData = CFrame.new(-1230.8199462890625, 38.287479400634766, -65.46951293945312),
                        Input = "Mouse1",
                        LightAttack = true,
                        Attack = true
                    }
                }
                pcall(function() event:FireServer(unpack(args)) end)
            end
        end
        wait(0.5)
    end
end)

-- =================== Auto Heavy Attack Loop ===================
spawn(function()
    while programRunning do
        if state.autoHeavyAttack then
            local char = getChar()
            local event = char:FindFirstChild("PlayerHandler") and char.PlayerHandler:FindFirstChild("HandlerEvent")
            if event then
                local args = {
                    {
                        CombatAction = true,
                        MouseData = CFrame.new(-1230.8199462890625, 38.287479400634766, -65.46951293945312),
                        Input = "Mouse2",
                        HeavyAttack = true,
                        Attack = true
                    }
                }
                pcall(function() event:FireServer(unpack(args)) end)
            end
        end
        wait(1)
    end
end)

-- =================== Auto Skills Loop ===================
local skills = {
    {Key="E", Pos=CFrame.new(-1345.676,38.287,-55.023,1,0,0,0,1,0,0,0,1)},
    {Key="R", Pos=CFrame.new(-1345.676,38.287,-55.023,1,0,0,0,1,0,0,0,1)},
    {Key="C", Pos=CFrame.new(-1345.676,38.287,-55.023,1,0,0,0,1,0,0,0,1)},
    {Key="V", Pos=CFrame.new(-1362.577,38.287,-40.816,1,0,0,0,1,0,0,0,1)},
}

for _, skill in ipairs(skills) do
    spawn(function()
        while programRunning do
            if state.autoSkills[skill.Key] then
                local char = getChar()
                local event = char:FindFirstChild("PlayerHandler") and char.PlayerHandler:FindFirstChild("HandlerEvent")
                if event then
                    local args = {
                        {
                            Key = skill.Key,
                            Skill = true,
                            MouseData = skill.Pos
                        }
                    }
                    pcall(function() event:FireServer(unpack(args)) end)
                end
            end
            wait(0.5)
        end
    end)
end

-- =================== Auto Press X Loop ===================
spawn(function()
    while programRunning do
        if state.autoKeyX then
            pcall(function()
                vu:CaptureController()
                vu:SetKeyDown("x")
                task.wait(0.05)
                vu:SetKeyUp("x")
            end)
        end
        task.wait(0.15)
    end
end)

-- =================== AFK Loop ===================
spawn(function()
    while programRunning do
        if state.afkEnabled then
            vu:CaptureController()
            vu:ClickButton2(Vector2.new(0,0))
        end
        wait(55)
    end
end)

-- =================== Physics ===================
local function ensureBP()
    local hrp = getHRP()
    if hrp and (not bp or not bp.Parent) then
        bp = Instance.new("BodyPosition")
        bp.MaxForce = Vector3.new(1e5,1e5,1e5)
        bp.P = 1e5
        bp.D = 1e4
        bp.Position = hrp.Position
        bp.Parent = hrp
    end
end
local function destroyBP() if bp then bp:Destroy() bp=nil end end

spawn(function()
    while programRunning do
        local char = getChar()
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid.WalkSpeed = state.lockPos and 0 or 16
        end
        local hrp = getHRP()
        if state.lockPos and hrp then ensureBP() bp.Position = hrp.Position else destroyBP() end
        wait(0.05)
    end
end)

-- =================== GUI ===================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AdminModern"
screenGui.ResetOnSpawn = false
screenGui.Parent = player:WaitForChild("PlayerGui")

local colors = {
    bg = Color3.fromRGB(20, 20, 25),
    panel = Color3.fromRGB(28, 28, 35),
    accent = Color3.fromRGB(88, 101, 242),
    success = Color3.fromRGB(67, 181, 129),
    danger = Color3.fromRGB(237, 66, 69),
    warning = Color3.fromRGB(250, 166, 26),
    text = Color3.fromRGB(220, 220, 225),
    textDim = Color3.fromRGB(140, 142, 150)
}

local mainFrame = Instance.new("Frame")
mainFrame.Name = "Main"
mainFrame.Size = UDim2.new(0, 480, 0, 520)
mainFrame.Position = UDim2.new(0.5, -240, 0.5, -260)
mainFrame.BackgroundColor3 = colors.bg
mainFrame.BorderSizePixel = 0
mainFrame.Parent = screenGui

local mainCorner = Instance.new("UICorner", mainFrame)
mainCorner.CornerRadius = UDim.new(0, 16)

local mainStroke = Instance.new("UIStroke", mainFrame)
mainStroke.Color = colors.accent
mainStroke.Thickness = 2
mainStroke.Transparency = 0.5

local topBar = Instance.new("Frame", mainFrame)
topBar.Size = UDim2.new(1, 0, 0, 45)
topBar.BackgroundColor3 = colors.panel
topBar.BorderSizePixel = 0

local topCorner = Instance.new("UICorner", topBar)
topCorner.CornerRadius = UDim.new(0, 16)

local title = Instance.new("TextLabel", topBar)
title.Size = UDim2.new(0, 250, 1, 0)
title.Position = UDim2.new(0, 15, 0, 0)
title.BackgroundTransparency = 1
title.Text = "⚡ Admin All In One"
title.TextColor3 = colors.text
title.Font = Enum.Font.GothamBold
title.TextSize = 17
title.TextXAlignment = Enum.TextXAlignment.Left

local btnMin = Instance.new("TextButton", topBar)
btnMin.Size = UDim2.new(0, 30, 0, 30)
btnMin.Position = UDim2.new(1, -75, 0.5, -15)
btnMin.Text = "—"
btnMin.Font = Enum.Font.GothamBold
btnMin.TextSize = 18
btnMin.BackgroundColor3 = colors.accent
btnMin.TextColor3 = Color3.fromRGB(255, 255, 255)
btnMin.BorderSizePixel = 0

local minCorner = Instance.new("UICorner", btnMin)
minCorner.CornerRadius = UDim.new(0, 8)

local btnClose = Instance.new("TextButton", topBar)
btnClose.Size = UDim2.new(0, 30, 0, 30)
btnClose.Position = UDim2.new(1, -38, 0.5, -15)
btnClose.Text = "✕"
btnClose.Font = Enum.Font.GothamBold
btnClose.TextSize = 16
btnClose.BackgroundColor3 = colors.danger
btnClose.TextColor3 = Color3.fromRGB(255, 255, 255)
btnClose.BorderSizePixel = 0

local closeCorner = Instance.new("UICorner", btnClose)
closeCorner.CornerRadius = UDim.new(0, 8)

local dragging = false
local dragStart, startPos

topBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = mainFrame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

topBar.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStart
        local newX = startPos.X.Offset + delta.X
        local newY = startPos.Y.Offset + delta.Y
        local vs = workspace.CurrentCamera.ViewportSize
        newX = math.clamp(newX, 0, vs.X - mainFrame.AbsoluteSize.X)
        newY = math.clamp(newY, 0, vs.Y - mainFrame.AbsoluteSize.Y)
        mainFrame.Position = UDim2.new(0, newX, 0, newY)
    end
end)

local tabContainer = Instance.new("Frame", mainFrame)
tabContainer.Size = UDim2.new(1, -20, 0, 40)
tabContainer.Position = UDim2.new(0, 10, 0, 55)
tabContainer.BackgroundTransparency = 1

local tabLayout = Instance.new("UIListLayout", tabContainer)
tabLayout.FillDirection = Enum.FillDirection.Horizontal
tabLayout.Padding = UDim.new(0, 5)
tabLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

local contentContainer = Instance.new("Frame", mainFrame)
contentContainer.Size = UDim2.new(1, -20, 1, -115)
contentContainer.Position = UDim2.new(0, 10, 0, 105)
contentContainer.BackgroundColor3 = colors.panel
contentContainer.BorderSizePixel = 0

local contentCorner = Instance.new("UICorner", contentContainer)
contentCorner.CornerRadius = UDim.new(0, 12)

local function createTab(text, order)
    local btn = Instance.new("TextButton", tabContainer)
    btn.Size = UDim2.new(0, 75, 1, 0)
    btn.Text = text
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 11
    btn.BackgroundColor3 = colors.bg
    btn.TextColor3 = colors.textDim
    btn.BorderSizePixel = 0
    btn.AutoButtonColor = false
    btn.LayoutOrder = order
    local corner = Instance.new("UICorner", btn)
    corner.CornerRadius = UDim.new(0, 8)
    return btn
end

local tabFarm = createTab("🎯 Farm", 1)
local tabBoss = createTab("👹 Boss", 2)
local tabMine = createTab("⛏️ Mine", 3)
local tabQuest = createTab("📜 เควส", 4)
local tabEvent = createTab("🎃 Event", 5)
local tabOther = createTab("⚙️ Other", 6)

local function createContent()
    local scroll = Instance.new("ScrollingFrame", contentContainer)
    scroll.Size = UDim2.new(1, -20, 1, -20)
    scroll.Position = UDim2.new(0, 10, 0, 10)
    scroll.BackgroundTransparency = 1
    scroll.BorderSizePixel = 0
    scroll.ScrollBarThickness = 6
    scroll.ScrollBarImageColor3 = colors.accent
    scroll.Visible = false
    local layout = Instance.new("UIListLayout", scroll)
    layout.Padding = UDim.new(0, 10)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    return scroll, layout
end

local farmContent, farmLayout = createContent()
local bossContent, bossLayout = createContent()
local mineContent, mineLayout = createContent()
local questContent, questLayout = createContent()
local eventContent, eventLayout = createContent()
local otherContent, otherLayout = createContent()

local function createToggle(parent, text, callback)
    local container = Instance.new("Frame", parent)
    container.Size = UDim2.new(1, 0, 0, 50)
    container.BackgroundColor3 = colors.bg
    container.BorderSizePixel = 0
    local corner = Instance.new("UICorner", container)
    corner.CornerRadius = UDim.new(0, 10)
    local label = Instance.new("TextLabel", container)
    label.Size = UDim2.new(1, -70, 1, 0)
    label.Position = UDim2.new(0, 15, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = colors.text
    label.Font = Enum.Font.GothamBold
    label.TextSize = 15
    label.TextXAlignment = Enum.TextXAlignment.Left
    local toggle = Instance.new("TextButton", container)
    toggle.Size = UDim2.new(0, 50, 0, 30)
    toggle.Position = UDim2.new(1, -60, 0.5, -15)
    toggle.Text = ""
    toggle.BackgroundColor3 = colors.textDim
    toggle.BorderSizePixel = 0
    local toggleCorner = Instance.new("UICorner", toggle)
    toggleCorner.CornerRadius = UDim.new(1, 0)
    local indicator = Instance.new("Frame", toggle)
    indicator.Size = UDim2.new(0, 22, 0, 22)
    indicator.Position = UDim2.new(0, 4, 0.5, -11)
    indicator.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    indicator.BorderSizePixel = 0
    local indCorner = Instance.new("UICorner", indicator)
    indCorner.CornerRadius = UDim.new(1, 0)
    local isOn = false
    toggle.MouseButton1Click:Connect(function()
        isOn = not isOn
        toggle.BackgroundColor3 = isOn and colors.success or colors.textDim
        indicator:TweenPosition(
            isOn and UDim2.new(1, -26, 0.5, -11) or UDim2.new(0, 4, 0.5, -11),
            Enum.EasingDirection.Out,
            Enum.EasingStyle.Quad,
            0.2,
            true
        )
        callback(isOn)
    end)
    return container
end

-- Farm Tab
farmContent.Visible = true
local farmTitle = Instance.new("TextLabel", farmContent)
farmTitle.Size = UDim2.new(1, 0, 0, 30)
farmTitle.BackgroundTransparency = 1
farmTitle.Text = "เลือกมอนสเตอร์ที่จะฟาร์ม"
farmTitle.TextColor3 = colors.accent
farmTitle.Font = Enum.Font.GothamBold
farmTitle.TextSize = 16
farmTitle.TextXAlignment = Enum.TextXAlignment.Left

createToggle(farmContent, "🎯 เปิด AutoFarm", function(on) state.autoFarm = on end)

local quickFrame = Instance.new("Frame", farmContent)
quickFrame.Size = UDim2.new(1, 0, 0, 40)
quickFrame.BackgroundTransparency = 1
local quickLayout = Instance.new("UIListLayout", quickFrame)
quickLayout.FillDirection = Enum.FillDirection.Horizontal
quickLayout.Padding = UDim.new(0, 10)

local function createQuickBtn(text, color)
    local btn = Instance.new("TextButton", quickFrame)
    btn.Size = UDim2.new(0.48, 0, 1, 0)
    btn.Text = text
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 14
    btn.BackgroundColor3 = color
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.BorderSizePixel = 0
    local corner = Instance.new("UICorner", btn)
    corner.CornerRadius = UDim.new(0, 8)
    return btn
end

local btnAll = createQuickBtn("✓ เลือกทั้งหมด", colors.success)
local btnNone = createQuickBtn("✗ ยกเลิกทั้งหมด", colors.danger)

local monsterScroll = Instance.new("ScrollingFrame", farmContent)
monsterScroll.Size = UDim2.new(1, 0, 0, 200)
monsterScroll.BackgroundColor3 = colors.bg
monsterScroll.BorderSizePixel = 0
monsterScroll.ScrollBarThickness = 4
local monsterScrollCorner = Instance.new("UICorner", monsterScroll)
monsterScrollCorner.CornerRadius = UDim.new(0, 10)
local monsterLayout = Instance.new("UIListLayout", monsterScroll)
monsterLayout.Padding = UDim.new(0, 5)
monsterLayout.SortOrder = Enum.SortOrder.LayoutOrder

local monsterButtons = {}
for i, mName in ipairs(targetMonsters) do
    local btn = Instance.new("TextButton", monsterScroll)
    btn.Size = UDim2.new(1, -10, 0, 35)
    btn.Text = "✓ " .. mName
    btn.Font = Enum.Font.Gotham
    btn.TextSize = 13
    btn.BackgroundColor3 = colors.success
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.BorderSizePixel = 0
    btn.TextXAlignment = Enum.TextXAlignment.Left
    btn.TextTruncate = Enum.TextTruncate.AtEnd
    local padding = Instance.new("UIPadding", btn)
    padding.PaddingLeft = UDim.new(0, 10)
    local corner = Instance.new("UICorner", btn)
    corner.CornerRadius = UDim.new(0, 6)
    btn.MouseButton1Click:Connect(function()
        selectedMonsters[mName] = not selectedMonsters[mName]
        btn.Text = (selectedMonsters[mName] and "✓ " or "✗ ") .. mName
        btn.BackgroundColor3 = selectedMonsters[mName] and colors.success or colors.textDim
    end)
    table.insert(monsterButtons, btn)
end
monsterScroll.CanvasSize = UDim2.new(0, 0, 0, #targetMonsters * 40)

btnAll.MouseButton1Click:Connect(function()
    for i, mName in ipairs(targetMonsters) do
        selectedMonsters[mName] = true
        monsterButtons[i].Text = "✓ " .. mName
        monsterButtons[i].BackgroundColor3 = colors.success
    end
end)

btnNone.MouseButton1Click:Connect(function()
    for i, mName in ipairs(targetMonsters) do
        selectedMonsters[mName] = false
        monsterButtons[i].Text = "✗ " .. mName
        monsterButtons[i].BackgroundColor3 = colors.textDim
    end
end)

farmLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    farmContent.CanvasSize = UDim2.new(0, 0, 0, farmLayout.AbsoluteContentSize.Y + 20)
end)

-- Boss Tab
local bossTitle = Instance.new("TextLabel", bossContent)
bossTitle.Size = UDim2.new(1, 0, 0, 30)
bossTitle.BackgroundTransparency = 1
bossTitle.Text = "ฟาร์มบอสอัตโนมัติ"
bossTitle.TextColor3 = colors.accent
bossTitle.Font = Enum.Font.GothamBold
bossTitle.TextSize = 16
bossTitle.TextXAlignment = Enum.TextXAlignment.Left

createToggle(bossContent, "👹 Farm Possessed Rider Lv.90", function(on) state.autoBoss = on end)
createToggle(bossContent, "🏛️ Auto Dungeon (Trial of Ethernal)", function(on) state.autoDungeon = on end)

local dungeonInfo = Instance.new("TextLabel", bossContent)
dungeonInfo.Size = UDim2.new(1, 0, 0, 70)
dungeonInfo.BackgroundColor3 = colors.bg
dungeonInfo.BorderSizePixel = 0
dungeonInfo.Text = "💡 คำแนะนำ Dungeon:\n• เปิด Auto Attack + Auto Skills ก่อนใช้งาน\n• ระบบจะ: กด Enter → Confirm → รอ 30 วิ → วาร์ปหาบอส → ฆ่า\n• ดู Console (F9) เพื่อดูสถานะการทำงาน"
dungeonInfo.TextColor3 = colors.textDim
dungeonInfo.Font = Enum.Font.Gotham
dungeonInfo.TextSize = 10
dungeonInfo.TextWrapped = true
dungeonInfo.TextXAlignment = Enum.TextXAlignment.Left
dungeonInfo.TextYAlignment = Enum.TextYAlignment.Top
local dungeonInfoPadding = Instance.new("UIPadding", dungeonInfo)
dungeonInfoPadding.PaddingLeft = UDim.new(0, 10)
dungeonInfoPadding.PaddingRight = UDim.new(0, 10)
dungeonInfoPadding.PaddingTop = UDim.new(0, 8)
dungeonInfoPadding.PaddingBottom = UDim.new(0, 8)
local dungeonInfoCorner = Instance.new("UICorner", dungeonInfo)
dungeonInfoCorner.CornerRadius = UDim.new(0, 10)

bossLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    bossContent.CanvasSize = UDim2.new(0, 0, 0, bossLayout.AbsoluteContentSize.Y + 20)
end)

-- Mine Tab
local mineTitle = Instance.new("TextLabel", mineContent)
mineTitle.Size = UDim2.new(1, 0, 0, 30)
mineTitle.BackgroundTransparency = 1
mineTitle.Text = "ฟาร์มเหมืองอัตโนมัติ"
mineTitle.TextColor3 = colors.accent
mineTitle.Font = Enum.Font.GothamBold
mineTitle.TextSize = 16
mineTitle.TextXAlignment = Enum.TextXAlignment.Left
createToggle(mineContent, "⛏️ Auto Mine (Miner Goon Lv.50)", function(on) state.autoMine = on end)
mineLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    mineContent.CanvasSize = UDim2.new(0, 0, 0, mineLayout.AbsoluteContentSize.Y + 20)
end)

-- Quest Tab
local questTitle = Instance.new("TextLabel", questContent)
questTitle.Size = UDim2.new(1, 0, 0, 30)
questTitle.BackgroundTransparency = 1
questTitle.Text = "📜 เลือกเควสที่ต้องการ"
questTitle.TextColor3 = colors.accent
questTitle.Font = Enum.Font.GothamBold
questTitle.TextSize = 16
questTitle.TextXAlignment = Enum.TextXAlignment.Center

local questButtons = {}
for idx, questData in ipairs(questDatabase) do
    local questCard = Instance.new("Frame", questContent)
    questCard.Size = UDim2.new(1, 0, 0, 100)
    questCard.BackgroundColor3 = colors.bg
    questCard.BorderSizePixel = 0
    local cardCorner = Instance.new("UICorner", questCard)
    cardCorner.CornerRadius = UDim.new(0, 10)
    local cardStroke = Instance.new("UIStroke", questCard)
    cardStroke.Color = colors.textDim
    cardStroke.Thickness = 1
    cardStroke.Transparency = 0.7
    
    local iconFrame = Instance.new("Frame", questCard)
    iconFrame.Size = UDim2.new(0, 70, 1, -20)
    iconFrame.Position = UDim2.new(0, 10, 0, 10)
    iconFrame.BackgroundColor3 = colors.panel
    iconFrame.BorderSizePixel = 0
    local iconCorner = Instance.new("UICorner", iconFrame)
    iconCorner.CornerRadius = UDim.new(0, 8)
    local iconLabel = Instance.new("TextLabel", iconFrame)
    iconLabel.Size = UDim2.new(1, 0, 1, 0)
    iconLabel.BackgroundTransparency = 1
    iconLabel.Text = questData.icon
    iconLabel.Font = Enum.Font.GothamBold
    iconLabel.TextSize = 32
    
    local infoFrame = Instance.new("Frame", questCard)
    infoFrame.Size = UDim2.new(1, -180, 1, -20)
    infoFrame.Position = UDim2.new(0, 90, 0, 10)
    infoFrame.BackgroundTransparency = 1
    
    local questNameLabel = Instance.new("TextLabel", infoFrame)
    questNameLabel.Size = UDim2.new(1, 0, 0, 22)
    questNameLabel.BackgroundTransparency = 1
    questNameLabel.Text = questData.name
    questNameLabel.TextColor3 = colors.text
    questNameLabel.Font = Enum.Font.GothamBold
    questNameLabel.TextSize = 14
    questNameLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    local npcLabel = Instance.new("TextLabel", infoFrame)
    npcLabel.Size = UDim2.new(1, 0, 0, 18)
    npcLabel.Position = UDim2.new(0, 0, 0, 24)
    npcLabel.BackgroundTransparency = 1
    npcLabel.Text = "👤 NPC: " .. questData.npcName
    npcLabel.TextColor3 = colors.textDim
    npcLabel.Font = Enum.Font.Gotham
    npcLabel.TextSize = 11
    npcLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    local typeLabel = Instance.new("TextLabel", infoFrame)
    typeLabel.Size = UDim2.new(1, 0, 0, 18)
    typeLabel.Position = UDim2.new(0, 0, 0, 44)
    typeLabel.BackgroundTransparency = 1
    if questData.questType == "kill" then
        typeLabel.Text = "⚔️ ฆ่ามอน: " .. #questData.monsters .. " ตัว"
    elseif questData.questType == "summon" then
        typeLabel.Text = "👹 Summon Boss: " .. questData.bossName
    end
    typeLabel.TextColor3 = colors.textDim
    typeLabel.Font = Enum.Font.Gotham
    typeLabel.TextSize = 11
    typeLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    local statusLabel = Instance.new("TextLabel", infoFrame)
    statusLabel.Size = UDim2.new(1, 0, 0, 16)
    statusLabel.Position = UDim2.new(0, 0, 0, 64)
    statusLabel.BackgroundTransparency = 1
    statusLabel.Text = "📊 สถานะ: รอเลือก"
    statusLabel.TextColor3 = colors.warning
    statusLabel.Font = Enum.Font.GothamBold
    statusLabel.TextSize = 10
    statusLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    local selectBtn = Instance.new("TextButton", questCard)
    selectBtn.Size = UDim2.new(0, 80, 0, 80)
    selectBtn.Position = UDim2.new(1, -90, 0.5, -40)
    selectBtn.Text = "เลือก"
    selectBtn.Font = Enum.Font.GothamBold
    selectBtn.TextSize = 13
    selectBtn.BackgroundColor3 = colors.accent
    selectBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    selectBtn.BorderSizePixel = 0
    local selectBtnCorner = Instance.new("UICorner", selectBtn)
    selectBtnCorner.CornerRadius = UDim.new(0, 8)
    
    selectBtn.MouseButton1Click:Connect(function()
        state.selectedQuest = questData
        for _, cardData in ipairs(questButtons) do
            cardData.card.BackgroundColor3 = colors.bg
            cardData.stroke.Color = colors.textDim
            cardData.stroke.Thickness = 1
            cardData.btn.BackgroundColor3 = colors.accent
            cardData.btn.Text = "เลือก"
            cardData.status.Text = "📊 สถานะ: รอเลือก"
            cardData.status.TextColor3 = colors.warning
        end
        questCard.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
        cardStroke.Color = colors.success
        cardStroke.Thickness = 2
        selectBtn.BackgroundColor3 = colors.success
        selectBtn.Text = "✓ เลือกแล้ว"
        statusLabel.Text = "✅ สถานะ: เลือกแล้ว"
        statusLabel.TextColor3 = colors.success
    end)
    
    table.insert(questButtons, {card = questCard, btn = selectBtn, stroke = cardStroke, status = statusLabel})
end

local questToggleContainer = Instance.new("Frame", questContent)
questToggleContainer.Size = UDim2.new(1, 0, 0, 55)
questToggleContainer.BackgroundColor3 = colors.panel
questToggleContainer.BorderSizePixel = 0
local questToggleCorner = Instance.new("UICorner", questToggleContainer)
questToggleCorner.CornerRadius = UDim.new(0, 10)

local questToggleLabel = Instance.new("TextLabel", questToggleContainer)
questToggleLabel.Size = UDim2.new(1, -70, 1, 0)
questToggleLabel.Position = UDim2.new(0, 15, 0, 0)
questToggleLabel.BackgroundTransparency = 1
questToggleLabel.Text = "🚀 เปิดระบบเควสอัตโนมัติ"
questToggleLabel.TextColor3 = colors.text
questToggleLabel.Font = Enum.Font.GothamBold
questToggleLabel.TextSize = 14
questToggleLabel.TextXAlignment = Enum.TextXAlignment.Left

local questToggle = Instance.new("TextButton", questToggleContainer)
questToggle.Size = UDim2.new(0, 50, 0, 30)
questToggle.Position = UDim2.new(1, -60, 0.5, -15)
questToggle.Text = ""
questToggle.BackgroundColor3 = colors.textDim
questToggle.BorderSizePixel = 0
local questToggleCorner2 = Instance.new("UICorner", questToggle)
questToggleCorner2.CornerRadius = UDim.new(1, 0)

local questIndicator = Instance.new("Frame", questToggle)
questIndicator.Size = UDim2.new(0, 22, 0, 22)
questIndicator.Position = UDim2.new(0, 4, 0.5, -11)
questIndicator.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
questIndicator.BorderSizePixel = 0
local questIndCorner = Instance.new("UICorner", questIndicator)
questIndCorner.CornerRadius = UDim.new(1, 0)

local questOn = false
questToggle.MouseButton1Click:Connect(function()
    if not state.selectedQuest then
        for i = 1, 3 do
            questToggleContainer.BackgroundColor3 = colors.danger
            wait(0.15)
            questToggleContainer.BackgroundColor3 = colors.panel
            wait(0.15)
        end
        return
    end
    questOn = not questOn
    state.autoQuest = questOn
    questToggle.BackgroundColor3 = questOn and colors.success or colors.textDim
    questIndicator:TweenPosition(
        questOn and UDim2.new(1, -26, 0.5, -11) or UDim2.new(0, 4, 0.5, -11),
        Enum.EasingDirection.Out,
        Enum.EasingStyle.Quad,
        0.2,
        true
    )
end)

local questInfo = Instance.new("TextLabel", questContent)
questInfo.Size = UDim2.new(1, 0, 0, 60)
questInfo.BackgroundColor3 = colors.bg
questInfo.BorderSizePixel = 0
questInfo.Text = "💡 คำแนะนำ:\n• เลือกเควสที่ต้องการก่อน\n• เปิด Auto Attack + Auto Skills เพื่อประสิทธิภาพสูงสุด"
questInfo.TextColor3 = colors.textDim
questInfo.Font = Enum.Font.Gotham
questInfo.TextSize = 10
questInfo.TextWrapped = true
questInfo.TextXAlignment = Enum.TextXAlignment.Left
questInfo.TextYAlignment = Enum.TextYAlignment.Top
local questInfoPadding = Instance.new("UIPadding", questInfo)
questInfoPadding.PaddingLeft = UDim.new(0, 10)
questInfoPadding.PaddingRight = UDim.new(0, 10)
questInfoPadding.PaddingTop = UDim.new(0, 8)
questInfoPadding.PaddingBottom = UDim.new(0, 8)
local questInfoCorner = Instance.new("UICorner", questInfo)
questInfoCorner.CornerRadius = UDim.new(0, 10)

questLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    questContent.CanvasSize = UDim2.new(0, 0, 0, questLayout.AbsoluteContentSize.Y + 20)
end)

-- Event Tab
local eventTitle = Instance.new("TextLabel", eventContent)
eventTitle.Size = UDim2.new(1, 0, 0, 30)
eventTitle.BackgroundTransparency = 1
eventTitle.Text = "🎃 Halloween Event"
eventTitle.TextColor3 = colors.accent
eventTitle.Font = Enum.Font.GothamBold
eventTitle.TextSize = 16
eventTitle.TextXAlignment = Enum.TextXAlignment.Center

local eventInfoCard = Instance.new("Frame", eventContent)
eventInfoCard.Size = UDim2.new(1, 0, 0, 80)
eventInfoCard.BackgroundColor3 = colors.bg
eventInfoCard.BorderSizePixel = 0
local eventInfoCorner = Instance.new("UICorner", eventInfoCard)
eventInfoCorner.CornerRadius = UDim.new(0, 10)
local eventInfoStroke = Instance.new("UIStroke", eventInfoCard)
eventInfoStroke.Color = Color3.fromRGB(255, 140, 0)
eventInfoStroke.Thickness = 2
eventInfoStroke.Transparency = 0.5

local eventIcon = Instance.new("TextLabel", eventInfoCard)
eventIcon.Size = UDim2.new(0, 70, 1, -16)
eventIcon.Position = UDim2.new(0, 8, 0, 8)
eventIcon.BackgroundTransparency = 1
eventIcon.Text = "🎃"
eventIcon.Font = Enum.Font.GothamBold
eventIcon.TextSize = 40

local eventInfoText = Instance.new("TextLabel", eventInfoCard)
eventInfoText.Size = UDim2.new(1, -90, 1, -16)
eventInfoText.Position = UDim2.new(0, 86, 0, 8)
eventInfoText.BackgroundTransparency = 1
eventInfoText.Text = "Halloween Chest Event\n📦 วาร์ปหากล่อง → 🔮 กด E → ⚔️ ฆ่ามอน → 🎁 รับรางวัล"
eventInfoText.TextColor3 = colors.text
eventInfoText.Font = Enum.Font.Gotham
eventInfoText.TextSize = 11
eventInfoText.TextWrapped = true
eventInfoText.TextXAlignment = Enum.TextXAlignment.Left
eventInfoText.TextYAlignment = Enum.TextYAlignment.Center

createToggle(eventContent, "🎃 เปิด Auto Halloween Event", function(on) state.autoEvent = on end)

local eventWarning = Instance.new("TextLabel", eventContent)
eventWarning.Size = UDim2.new(1, 0, 0, 70)
eventWarning.BackgroundColor3 = colors.bg
eventWarning.BorderSizePixel = 0
eventWarning.Text = "⚠️ คำเตือน:\n• เปิด Auto Attack + Auto Skills ก่อนใช้งาน\n• ระบบจะวาร์ปและวนลูปอัตโนมัติ\n• ดู Console (F9) เพื่อดูสถานะการทำงาน"
eventWarning.TextColor3 = colors.warning
eventWarning.Font = Enum.Font.GothamBold
eventWarning.TextSize = 10
eventWarning.TextWrapped = true
eventWarning.TextXAlignment = Enum.TextXAlignment.Left
eventWarning.TextYAlignment = Enum.TextYAlignment.Top
local eventWarningPadding = Instance.new("UIPadding", eventWarning)
eventWarningPadding.PaddingLeft = UDim.new(0, 10)
eventWarningPadding.PaddingRight = UDim.new(0, 10)
eventWarningPadding.PaddingTop = UDim.new(0, 8)
eventWarningPadding.PaddingBottom = UDim.new(0, 8)
local eventWarningCorner = Instance.new("UICorner", eventWarning)
eventWarningCorner.CornerRadius = UDim.new(0, 10)

eventLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    eventContent.CanvasSize = UDim2.new(0, 0, 0, eventLayout.AbsoluteContentSize.Y + 20)
end)

-- Other Tab
local otherTitle = Instance.new("TextLabel", otherContent)
otherTitle.Size = UDim2.new(1,0, 0, 30)
otherTitle.BackgroundTransparency = 1
otherTitle.Text = "การตั้งค่าอื่นๆ"
otherTitle.TextColor3 = colors.accent
otherTitle.Font = Enum.Font.GothamBold
otherTitle.TextSize = 16
otherTitle.TextXAlignment = Enum.TextXAlignment.Left

createToggle(otherContent, "🔒 ล็อคตำแหน่ง (Lock Position)", function(on) state.lockPos = on end)
createToggle(otherContent, "😴 โหมด AFK (ป้องกันโดนเตะ)", function(on) state.afkEnabled = on end)
createToggle(otherContent, "⚔️ Auto Light Attack", function(on) state.autoAttack = on end)
createToggle(otherContent, "💥 Auto Heavy Attack", function(on) state.autoHeavyAttack = on end)
createToggle(otherContent, "❌ Auto Press X", function(on) state.autoKeyX = on end)
createToggle(otherContent, "🔥 Auto Skill E", function(on) state.autoSkills.E = on end)
createToggle(otherContent, "💫 Auto Skill R", function(on) state.autoSkills.R = on end)
createToggle(otherContent, "⚡ Auto Skill C", function(on) state.autoSkills.C = on end)
createToggle(otherContent, "✨ Auto Skill V", function(on) state.autoSkills.V = on end)

otherLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    otherContent.CanvasSize = UDim2.new(0, 0, 0, otherLayout.AbsoluteContentSize.Y + 20)
end)

-- Tab Switching
local function switchTab(tabName)
    farmContent.Visible = (tabName == "Farm")
    bossContent.Visible = (tabName == "Boss")
    mineContent.Visible = (tabName == "Mine")
    questContent.Visible = (tabName == "Quest")
    eventContent.Visible = (tabName == "Event")
    otherContent.Visible = (tabName == "Other")
    
    tabFarm.BackgroundColor3 = tabName == "Farm" and colors.accent or colors.bg
    tabFarm.TextColor3 = tabName == "Farm" and Color3.fromRGB(255,255,255) or colors.textDim
    
    tabBoss.BackgroundColor3 = tabName == "Boss" and colors.accent or colors.bg
    tabBoss.TextColor3 = tabName == "Boss" and Color3.fromRGB(255,255,255) or colors.textDim
    
    tabMine.BackgroundColor3 = tabName == "Mine" and colors.accent or colors.bg
    tabMine.TextColor3 = tabName == "Mine" and Color3.fromRGB(255,255,255) or colors.textDim
    
    tabQuest.BackgroundColor3 = tabName == "Quest" and colors.accent or colors.bg
    tabQuest.TextColor3 = tabName == "Quest" and Color3.fromRGB(255,255,255) or colors.textDim
    
    tabEvent.BackgroundColor3 = tabName == "Event" and colors.accent or colors.bg
    tabEvent.TextColor3 = tabName == "Event" and Color3.fromRGB(255,255,255) or colors.textDim
    
    tabOther.BackgroundColor3 = tabName == "Other" and colors.accent or colors.bg
    tabOther.TextColor3 = tabName == "Other" and Color3.fromRGB(255,255,255) or colors.textDim
end

tabFarm.MouseButton1Click:Connect(function() switchTab("Farm") end)
tabBoss.MouseButton1Click:Connect(function() switchTab("Boss") end)
tabMine.MouseButton1Click:Connect(function() switchTab("Mine") end)
tabQuest.MouseButton1Click:Connect(function() switchTab("Quest") end)
tabEvent.MouseButton1Click:Connect(function() switchTab("Event") end)
tabOther.MouseButton1Click:Connect(function() switchTab("Other") end)

-- Minimize/Restore Button
local minimized = false
local rBtn = Instance.new("TextButton")
rBtn.Size = UDim2.new(0, 60, 0, 60)
rBtn.Position = UDim2.new(0, 50, 0, 100)
rBtn.BackgroundColor3 = colors.accent
rBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
rBtn.Text = "R"
rBtn.Font = Enum.Font.GothamBold
rBtn.TextSize = 28
rBtn.Visible = false
rBtn.Parent = screenGui

local rCorner = Instance.new("UICorner", rBtn)
rCorner.CornerRadius = UDim.new(1, 0)

local rStroke = Instance.new("UIStroke", rBtn)
rStroke.Color = Color3.fromRGB(255, 255, 255)
rStroke.Thickness = 3

btnMin.MouseButton1Click:Connect(function()
    minimized = true
    mainFrame.Visible = false
    rBtn.Visible = true
end)

rBtn.MouseButton1Click:Connect(function()
    minimized = false
    mainFrame.Visible = true
    rBtn.Visible = false
end)

local rDragging = false
local rDragStart, rStartPos

rBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        rDragging = true
        rDragStart = input.Position
        rStartPos = rBtn.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                rDragging = false
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if rDragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - rDragStart
        local newX = rStartPos.X.Offset + delta.X
        local newY = rStartPos.Y.Offset + delta.Y
        local vs = workspace.CurrentCamera.ViewportSize
        newX = math.clamp(newX, 0, vs.X - rBtn.AbsoluteSize.X)
        newY = math.clamp(newY, 0, vs.Y - rBtn.AbsoluteSize.Y)
        rBtn.Position = UDim2.new(0, newX, 0, newY)
    end
end)

-- Close Confirmation
local confirmFrame = Instance.new("Frame")
confirmFrame.Size = UDim2.new(0, 350, 0, 150)
confirmFrame.Position = UDim2.new(0.5, -175, 0.5, -75)
confirmFrame.BackgroundColor3 = colors.bg
confirmFrame.BorderSizePixel = 0
confirmFrame.Visible = false
confirmFrame.Parent = screenGui

local confirmCorner = Instance.new("UICorner", confirmFrame)
confirmCorner.CornerRadius = UDim.new(0, 16)

local confirmStroke = Instance.new("UIStroke", confirmFrame)
confirmStroke.Color = colors.danger
confirmStroke.Thickness = 2

local confirmText = Instance.new("TextLabel", confirmFrame)
confirmText.Size = UDim2.new(1, -40, 0, 60)
confirmText.Position = UDim2.new(0, 20, 0, 20)
confirmText.BackgroundTransparency = 1
confirmText.Text = "คุณต้องการปิดการใช้งานใช่หรือไม่?"
confirmText.Font = Enum.Font.GothamBold
confirmText.TextSize = 16
confirmText.TextColor3 = colors.text
confirmText.TextWrapped = true

local btnYes = Instance.new("TextButton", confirmFrame)
btnYes.Size = UDim2.new(0, 130, 0, 40)
btnYes.Position = UDim2.new(0.5, -140, 1, -55)
btnYes.Text = "ใช่ ปิดเลย"
btnYes.Font = Enum.Font.GothamBold
btnYes.TextSize = 14
btnYes.BackgroundColor3 = colors.success
btnYes.TextColor3 = Color3.fromRGB(255, 255, 255)
btnYes.BorderSizePixel = 0

local yesCorner = Instance.new("UICorner", btnYes)
yesCorner.CornerRadius = UDim.new(0, 8)

local btnNo = Instance.new("TextButton", confirmFrame)
btnNo.Size = UDim2.new(0, 130, 0, 40)
btnNo.Position = UDim2.new(0.5, 10, 1, -55)
btnNo.Text = "ไม่ ยกเลิก"
btnNo.Font = Enum.Font.GothamBold
btnNo.TextSize = 14
btnNo.BackgroundColor3 = colors.danger
btnNo.TextColor3 = Color3.fromRGB(255, 255, 255)
btnNo.BorderSizePixel = 0

local noCorner = Instance.new("UICorner", btnNo)
noCorner.CornerRadius = UDim.new(0, 8)

btnClose.MouseButton1Click:Connect(function()
    confirmFrame.Visible = true
end)

btnYes.MouseButton1Click:Connect(function()
    programRunning = false
    screenGui:Destroy()
end)

btnNo.MouseButton1Click:Connect(function()
    confirmFrame.Visible = false
end)

print("✅ AdminAllInOne - Full Version โหลดสำเร็จ!")
print("🏛️ Auto Dungeon: Trial of Ethernal (ใหม่!)")
print("🎃 Halloween Event: Auto Farm + Teleport")
print("📜 Auto Quest: Kill & Summon Boss")
print("📊 กด F9 เพื่อเปิด Console ดูสถานะการทำงาน")
