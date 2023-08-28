-- Constants

local REMOVE_BANANA_PEELS = true
local REMOVE_JEFFTHEKILLER_HITBOX = true -- makes jeff unable to deal damage
local REMOVE_GREED_RISK = true -- makes you unable to collect gold when you risk taking damage from greed
local REMOVE_AMBIENCE_SOUNDS = false -- not that necessary but if you want to hear better keep it
local REMOVE_EYES_DAMAGE = true -- makes eyes deal no damage
local REMOVE_SCREECH = true
local REMOVE_LIGHT_SHATTER = true -- makes lights not shatter from any entity moving (screech will still appear unless removed)
local REMOVE_LIBRARY_DOOR = true -- you can just go past it without doing the padlock

local COMPLETE_ELEVATOR_MINIGAME = true -- elevator breaker (be ready to run)
-- creates a guiding light so you know where to go
local MARK_NEXT_DOOR = true
local MARK_KEY_PICKUP = true
local MARK_GATE_LEVER = true
local MARK_HINT_BOOKS = true
local MARK_ELEVATOR_BREAKER_POLES = true -- for room 100

local SPEED_BOOST_ENABLED = true
local BOOST_EXTRA_WALKSPEED = 6 -- above makes you teleport back by anticheat

local CREATE_ENTITY_HINTS = true -- watch your topbar for hints when entities spawn

--- don't touch below (you can however change light properties, color is (red, green, blue) up to 255)

-- Services

local PlayersService = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- Vars

local CurrentRooms = workspace:WaitForChild("CurrentRooms")

local LocalPlayer = PlayersService.LocalPlayer

--

local Hint, OnRemovalConnection
local function HandleModels(model)
task.wait()

local modelIdentifier = model.Name

if modelIdentifier == "BananaPeel" and REMOVE_BANANA_PEELS then
model:Destroy()
elseif modelIdentifier == "JeffTheKiller" and REMOVE_JEFFTHEKILLER_HITBOX then
local knife = model:WaitForChild("Knife", 10)
if not knife then
return
end

for _, descendant in ipairs(model:GetDescendants()) do
if descendant:IsA("BasePart") then
descendant.CanTouch = false
descendant.CanQuery = false
end
end
elseif (modelIdentifier == "RushMoving" or modelIdentifier == "AmbushMoving") and CREATE_ENTITY_HINTS then
local function formatIdentifier(primaryPart)
local entityIdentifier = primaryPart.Name
return entityIdentifier == "RushNew" and string.gsub(model.Name, "Moving", "") or entityIdentifier
end

local function onRemovalFunction()
Hint:Destroy()
Hint = nil

OnRemovalConnection = nil
end

local primaryPart = model.PrimaryPart or model:FindFirstChildWhichIsA("BasePart")
if not primaryPart or primaryPart.Position.Y < -1000 then -- ignore fake ones
return
end

local entityIdentifier = formatIdentifier(primaryPart)

if Hint and OnRemovalConnection then
OnRemovalConnection:Disconnect()
-- let the player know there is another entity coming right after
Hint.Text = entityIdentifier.. " is coming right after! hide!!!!!"
-- clear stuff up later
OnRemovalConnection = model:GetPropertyChangedSignal("Parent"):Once(onRemovalFunction)
return
end

Hint = Instance.new("Hint")
Hint.Text = entityIdentifier.. " is coming! hide!!!"
Hint.Parent = workspace
-- clear stuff up
OnRemovalConnection = model:GetPropertyChangedSignal("Parent"):Once(onRemovalFunction)
end
end

local function HandleRooms(room)
local currentRoomId = tonumber(room.Name)
local nextRoomId = currentRoomId + 1

local nextRoomNumber = string.rep("0", 4 - #tostring(nextRoomId))..nextRoomId

local isLibrary = currentRoomId == 50
local isTheEnd = not isLibrary and currentRoomId == 100

local function waitForAttribute(instance, attribute, expectedValue)
local function isValid()
local currentValue = instance:GetAttribute(attribute)
return currentValue == expectedValue
end

if isValid() then
return true
end

instance:GetAttributeChangedSignal(attribute):Wait()

return isValid()
end

local function handleDescendant(descendant)
local guidingLight = Instance.new("Part")
if MARK_NEXT_DOOR and descendant.Name == "Door" and waitForAttribute(descendant, "RoomID", nextRoomId) then
guidingLight.Range = 5
guidingLight.Color = Color3.fromRGB(255, 255, 255)
guidingLight.Shadows = true
guidingLight.Parent = descendant:WaitForChild("Door", 7)
elseif MARK_KEY_PICKUP and descendant.Name == "KeyObtain" and waitForAttribute(descendant, "LockID", nextRoomNumber) then
local hitbox = descendant:WaitForChild("Hitbox", 5)
local keyBase = hitbox and hitbox:WaitForChild("Key", 5)
if keyBase then
local hintAttachment = keyBase:WaitForChild("Attachment", 5)
local hintLight = hintAttachment and hintAttachment:WaitForChild("PointLight", 5)
if hintLight then
hintLight.Enabled = true
hintLight.Shadows = true
hintLight.Range = 60
hintLight.Brightness = 2
end
end
elseif MARK_GATE_LEVER and descendant.Name == "LeverForGate" then
guidingLight.Range = 60
guidingLight.Color = Color3.fromRGB(0, 255, 255)
guidingLight.Shadows = true
guidingLight.Parent = descendant:WaitForChild("Main", 5)
elseif MARK_HINT_BOOKS and descendant.Name == "LiveHintBook" then
guidingLight.Range = 20
guidingLight.Color = Color3.fromRGB(255, 0, 255)
guidingLight.Shadows = false
guidingLight.Parent = descendant:WaitForChild("Base", 5)
elseif MARK_ELEVATOR_BREAKER_POLES and isTheEnd and descendant.Name == "LiveBreakerPolePickup" then
local base = descendant:WaitForChild("Base", 5)
local hintAttachment = base and base:WaitForChild("Attachment", 5)
if hintAttachment then
local hintParticle = hintAttachment:WaitForChild("HintParticle", 5)
local hintLight = hintAttachment:WaitForChild("HintLight", 5)
if hintParticle then
hintParticle.Enabled = true
end
if hintLight then
hintLight.Enabled = true
end
end
end
end

if REMOVE_LIBRARY_DOOR and isLibrary then
task.wait()

local door = room:WaitForChild("Door", 60)
if door then
door:Destroy()
end
end

for _, descendant in ipairs(room:GetDescendants()) do
task.spawn(handleDescendant, descendant)
end

room.DescendantAdded:Connect(handleDescendant)
end

local function HandleLoot(prompt)
if not REMOVE_GREED_RISK then
return
end

if prompt.Name ~= "LootPrompt" or prompt.ActionText ~= "Collect" then
return
end
-- if this isn't done script won't be able to use holdbegan signal to prevent collecting
prompt.HoldDuration = 0.05

local holdBeganConnection
local ancestryChangedConnection

holdBeganConnection = prompt.PromptButtonHoldBegan:Connect(function(playerWhoTriggered)
if LocalPlayer ~= playerWhoTriggered or LocalPlayer:GetAttribute("Greed") ~= 6 or prompt.HoldDuration == 999999 then
return
end

prompt.HoldDuration = 999999
-- yield until greed level changes
LocalPlayer:GetAttributeChangedSignal("Greed"):Wait()

prompt.HoldDuration = 0.05

end)
ancestryChangedConnection = game.AncestryChanged:Connect(function()
if prompt:IsDescendantOf(CurrentRooms) then
return
end
-- prompt was removed from workspace; these connections will only take memory
holdBeganConnection:Disconnect()
ancestryChangedConnection:Disconnect()

holdBeganConnection = nil
ancestryChangedConnection = nil
end)
end

local function HandleEntities()
local entitiesFolder = ReplicatedStorage:WaitForChild("Entities", 5)
if not entitiesFolder then
return
end

if REMOVE_SCREECH then
local screechModel = entitiesFolder:WaitForChild("Screech", 5)
if not screechModel then
return
end

screechModel:Destroy()
end
end

local function HandleAmbience()
if not REMOVE_AMBIENCE_SOUNDS then
return
end

local ambience_dark = workspace:WaitForChild("Ambience_Dark", 5)
if ambience_dark then
ambience_dark.Volume = 0
end

local ambienceFolder = workspace:WaitForChild("Ambience", 5)
if not ambienceFolder then
return
end

for _, descendant in ipairs(ambienceFolder:GetDescendants()) do
if descendant.ClassName == "Sound" then
descendant.Volume = 0
end
end
end

local function HandleSpeedBoost()
if not SPEED_BOOST_ENABLED then
return
end

local function handleCharacter(character)
local humanoid = character:WaitForChild("Humanoid")

local updating = false
local function updateSpeedBoost()
if updating then
return
end
-- basic debounce
updating = true

humanoid:SetAttribute("SpeedBoostBehind", BOOST_EXTRA_WALKSPEED)

updating = false
end

humanoid:GetAttributeChangedSignal("SpeedBoostBehind"):Connect(updateSpeedBoost)

updateSpeedBoost()
end
-- on revive
LocalPlayer.CharacterAdded:Connect(handleCharacter)

handleCharacter(LocalPlayer.Character)
end

local function HandleGameEvents()
local function isValid(func)
return type(func) == "function" and islclosure(func) and not is_synapse_function(func)
end

local gc = getgc()

if REMOVE_LIGHT_SHATTER then
for _, value in pairs(gc) do
if isValid(value) then
local info = debug.getinfo(value)
if info.currentline == 14 and string.find(info.short_src, "Module_Events") then
local functionEnvironment = getfenv(value)
-- remove luau optimizations (thats necessary)
setuntouched(functionEnvironment, false)
-- overwrite spawn
functionEnvironment.spawn = function()end
break
end
end
end
end
end

local function HandleRemotes()
local entityInfo = ReplicatedStorage:WaitForChild("EntityInfo", 5)
if not entityInfo then
return
end

if REMOVE_EYES_DAMAGE then
local motorReplicationEvent = entityInfo:WaitForChild("MotorReplication", 5)
if not motorReplicationEvent then
return
end

local findFirstChild = game.FindFirstChild

local originalNC
originalNC = hookmetamethod(game, "__namecall", function(self, ...)
if self and rawequal(self, motorReplicationEvent) then
if findFirstChild(workspace, "Eyes") then
local function filter(x, y, ...) -- x, y, z, crouching
return x, -89 - math.random(), ...
end

return originalNC(self, filter(...))
end
end

return originalNC(self, ...)
end, true)
end

if COMPLETE_ELEVATOR_MINIGAME then
local engageMinigame = entityInfo:WaitForChild("EngageMinigame", 5)
local elevatorBreakerFinished = entityInfo:WaitForChild("EBF", 5)
if not engageMinigame or not elevatorBreakerFinished then
return
end

local minigameFunction

for _, connection in ipairs(getconnections(engageMinigame.OnClientEvent)) do
local func = connection.Function
if not func then
return -- ???
end

local minigame = debug.getupvalue(func, 1)
-- grab minigame func
if not minigameFunction and type(minigame) == "function" then
minigameFunction = minigame
end

connection:Disable()
end
-- create our own handler (i dont want to use hookfunction that much)
engageMinigame.OnClientEvent:Connect(function(minigame, ...)
if minigame == "ElevatorBreaker" then
task.wait(20)
-- mark as completed
elevatorBreakerFinished:FireServer()
return
end

if not minigameFunction then
return
end

task.spawn(minigameFunction, minigame, ...)
end)
end
end

local function SetUp()
workspace.ChildAdded:Connect(HandleModels)

for _, child in ipairs(workspace:GetChildren()) do
task.spawn(HandleModels, child)
end

--

CurrentRooms.ChildAdded:Connect(HandleRooms)

for _, room in ipairs(CurrentRooms:GetChildren()) do
task.spawn(HandleRooms, room)
end

--

game.DescendantAdded:Connect(HandleLoot)

for _, descendant in ipairs(game:GetDescendants()) do
task.spawn(HandleLoot,descendant)
end

--

task.spawn(HandleEntities)
task.spawn(HandleAmbience)
task.spawn(HandleSpeedBoost)
task.spawn(HandleGameEvents)
task.spawn(HandleRemotes)
end

SetUp()
