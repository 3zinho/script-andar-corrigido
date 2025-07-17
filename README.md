local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerStorage = game:GetService("ServerStorage")
local TweenService = game:GetService("TweenService")
local Players = game:GetService("Players")

local Insert = require(ServerStorage.Modules:WaitForChild("Insert"))
local SetProperties = require(ServerStorage.Modules:WaitForChild("SetProperties"))

local Numbers = require(ReplicatedStorage.Modules:WaitForChild("Numbers"))
local Things = require(ReplicatedStorage.Configuration.Modules:WaitForChild("Things"))
local Mutations = require(ReplicatedStorage.Configuration.Modules:WaitForChild("Mutations"))
local Rarities = require(ReplicatedStorage.Configuration.Modules:WaitForChild("Rarities"))

local AddThingEvent = ServerStorage.Events:WaitForChild("AddThing")
local NotificationEvent = ReplicatedStorage.Events:WaitForChild("Notification")

local Waypoints = workspace:WaitForChild("Waypoints")

local ThingData = {}

local function GetRandomRarity()
    local UsedRarities = {}
    for _, Configuration in pairs(Things) do
        local Rarity = Configuration.Rarity
        if Rarities[Rarity] then
            UsedRarities[Rarity] = true
        end
    end

    local Total = 0
    for Rarity, Data in pairs(Rarities) do
        if UsedRarities[Rarity] then
            Total = Total + Data.Chance
        end
    end

    local Roll = math.random() * Total
    local Sum = 0

    for Rarity, Data in pairs(Rarities) do
        if UsedRarities[Rarity] then
            Sum = Sum + Data.Chance
            if Roll <= Sum then
                return Rarity
            end
        end
    end
end

local function GetRandomMutation()
    local Total = 0
    for Mutation, Data in pairs(Mutations) do
        Total = Total + Data.Chance
    end

    local Roll = math.random() * Total
    local Sum = 0

    for Mutation, Data in pairs(Mutations) do
        Sum = Sum + Data.Chance
        if Roll <= Sum then
            return Mutation
        end
    end
end

local function GetRandomThing()
    for _ = 1, 20 do
        local Rarity = GetRandomRarity()
        local MatchingNames = {}

        for Name, Configuration in pairs(Things) do
            if Configuration.Rarity == Rarity then
                table.insert(MatchingNames, Name)
            end
        end

        if #MatchingNames > 0 then
            local Name = MatchingNames[math.random(1, #MatchingNames)]
            local Configuration = Things[Name]
            if not Configuration then return end

            local Thing = ServerStorage.Things:FindFirstChild(Name)
            if not Thing then return end

            return Thing:Clone(), Name, Configuration, Rarity, GetRandomMutation()
        end
    end

    return nil
end

local function AddRandomThing()
    local Thing, Name, Configuration, Rarity, Mutation = GetRandomThing()
    if not (Thing and Name and Configuration and Rarity and Mutation) then return end

    Thing:SetAttribute("Mutation", Mutation)

    local MutationColor = Mutations[Mutation] and Mutations[Mutation].Color or Color3.new(1, 1, 1)
    for _, Descendant in ipairs(Thing:GetDescendants()) do
        if Descendant:IsA("BasePart") then
            local OriginalColor = Descendant.Color
            Descendant.Color = Color3.new(
                OriginalColor.R * MutationColor.R,
                OriginalColor.G * MutationColor.G,
                OriginalColor.B * MutationColor.B
            )
        end
    end

    local HeightOffset = Thing:GetExtentsSize().Y / 2
    Thing:PivotTo(Waypoints.Waypoint1.CFrame + Vector3.new(0, HeightOffset, 0))
    Thing.Parent = workspace.Things

    local ThingAttachment, ThingGui = Insert.Thing(Thing, Configuration, Mutation)

    local ProximityPrompt = Insert.ProximityPrompt(Thing.PrimaryPart)
    ProximityPrompt.ActionText = string.format("Purchase $%s", Numbers.Format(Configuration.Price))
    ProximityPrompt.HoldDuration = 0.25
    ProximityPrompt.ObjectText = Name

    SetProperties.AllClients(ProximityPrompt, {Enabled = true})

    local AnimationController = Instance.new("AnimationController", Thing)
    local Animator = Instance.new("Animator", AnimationController)

    local ThingConfiguration = Instance.new("Configuration")
    ThingConfiguration.Parent = Thing

    local AnimationsFolder = Instance.new("Folder")
    AnimationsFolder.Name = "Animations"
    AnimationsFolder.Parent = ThingConfiguration

    local WalkTrack
    if Configuration.WalkAnimation and Configuration.WalkAnimation ~= "" then
        local WalkAnimation = Instance.new("Animation")
        WalkAnimation.Name = "WalkAnimation"
        WalkAnimation.AnimationId = Configuration.WalkAnimation
        WalkAnimation.Parent = AnimationsFolder

        WalkTrack = Animator:LoadAnimation(WalkAnimation)
    end

    if WalkTrack then WalkTrack:Play() end

    local WaypointRotationTween
    local WaypointMoveTween

    ProximityPrompt.Triggered:Connect(function(Player)
        if ThingData[Thing] and ThingData[Thing].Owner == Player then return end

        local Leaderstats = Player:FindFirstChild("Leaderstats")
        local Money = Leaderstats and Leaderstats:FindFirstChild("Money")
        local Rebirths = Leaderstats and Leaderstats:FindFirstChild("Rebirths")

        local Price = Configuration.Price or 0
        local PurchasedAmount = ThingData[Thing] and ThingData[Thing].PurchasedAmount or 0
        PurchasedAmount += 1

        if not Money or Money.Value < Price * PurchasedAmount then return end

        local ConfigurationFolder = Player:FindFirstChild("Configuration")
        local BaseValue = ConfigurationFolder and ConfigurationFolder:FindFirstChild("Base")
        local Base = BaseValue and BaseValue.Value
        if not Base then return end

        local CollectZone = Base:FindFirstChild("CollectZone")
        if not CollectZone then return end

        local SlotsFolder = Base:FindFirstChild("Slots")
        if not SlotsFolder then return end

        local Slots = SlotsFolder:GetChildren()
        table.sort(Slots, function(A, B)
            local AIndex = tonumber(string.match(A.Name, "%d+")) or 0
            local BIndex = tonumber(string.match(B.Name, "%d+")) or 0
            return AIndex < BIndex
        end)

        local PreviousSlot = ThingData[Thing] and ThingData[Thing].Slot
        if PreviousSlot then
            PreviousSlot.Configuration.Occupied.Value = false
        end

        local PlayerRebirths = Rebirths and Rebirths.Value or 0

        local UnoccupiedSlot = nil
        for _, Slot in ipairs(Slots) do
            if Slot.Configuration.Occupied.Value == false and Slot.Configuration.Rebirth.Value <= PlayerRebirths then
                Slot.Configuration.Occupied.Value = true
                UnoccupiedSlot = Slot
                break
            end
        end

        if not UnoccupiedSlot then
            NotificationEvent:FireClient(Player, "Not enough space!", false)
            return
        end

        Money.Value = Money.Value - Price * PurchasedAmount

        ThingData[Thing] = {
            Owner = Player,
            Slot = UnoccupiedSlot,
            PurchasedAmount = PurchasedAmount,
            PurchaseRebirths = PlayerRebirths
        }

        SetProperties.Client(Player, ProximityPrompt, {Enabled = false})
        ProximityPrompt.ActionText = string.format("Purchase $%s", Numbers.Format(Price * (ThingData[Thing].PurchasedAmount + 1)))

        local CurrentCFrame = Thing.PrimaryPart.CFrame
        if WaypointRotationTween then
            WaypointRotationTween:Cancel()
            WaypointRotationTween = nil
        end
        if WaypointMoveTween then
            WaypointMoveTween:Cancel()
            WaypointMoveTween = nil
        end

        Thing:PivotTo(CurrentCFrame)

        local StartPosition = CurrentCFrame.Position
        local CollectZonePosition = Vector3.new(CollectZone.Position.X, CollectZone.Position.Y + HeightOffset, CollectZone.Position.Z)
        local Direction = (CollectZonePosition - StartPosition).Unit
        local YAngle = math.atan2(-Direction.X, -Direction.Z)
        local TargetRotation = CFrame.new(StartPosition) * CFrame.Angles(0, YAngle, 0)

        WaypointRotationTween = TweenService:Create(Thing.PrimaryPart, TweenInfo.new(0.3, Enum.EasingStyle.Linear), {
            CFrame = TargetRotation
        })

        WaypointRotationTween:Play()
        WaypointRotationTween.Completed:Wait()

        local Distance = (CollectZonePosition - StartPosition).Magnitude
        WaypointMoveTween = TweenService:Create(Thing.PrimaryPart, TweenInfo.new(Distance / 8, Enum.EasingStyle.Linear), {
            CFrame = CFrame.new(CollectZonePosition) * CFrame.Angles(0, YAngle, 0)
        })

        WaypointMoveTween:Play()
        WaypointMoveTween.Completed:Wait()

        if not ThingData[Thing] or ThingData[Thing].Owner ~= Player then return end

        local StoredRebirths = ThingData[Thing] and ThingData[Thing].PurchaseRebirths or 0
        local CurrentRebirths = Rebirths and Rebirths.Value or 0

        if CurrentRebirths > StoredRebirths then
            if UnoccupiedSlot then
                UnoccupiedSlot.Configuration.Occupied.Value = false
            end

            ThingData[Thing] = nil
            Thing:Destroy()
            return
        end

        ThingData[Thing] = nil
        Thing:Destroy()

        if not Player then return end
        if not UnoccupiedSlot then return end
        if not Name then return end

        AddThingEvent:Fire(Player, UnoccupiedSlot, Name, Mutation)
    end)

    local WaypointList = Waypoints:GetChildren()
    table.sort(WaypointList, function(A, B)
        return tonumber(string.match(A.Name, "%d+")) < tonumber(string.match(B.Name, "%d+"))
    end)

    for _, Waypoint in ipairs(WaypointList) do
        if ThingData[Thing] then break end

        local TargetPosition = Vector3.new(Waypoint.Position.X, Waypoint.Position.Y + HeightOffset, Waypoint.Position.Z)
        local CurrentPosition = Thing.PrimaryPart.Position
        local StartPosition = Vector3.new(CurrentPosition.X, TargetPosition.Y, CurrentPosition.Z)
        local Direction = (TargetPosition - StartPosition).Unit
        local YAngle = math.atan2(-Direction.X, -Direction.Z)

        local CurrentCFrame = Thing.PrimaryPart.CFrame
        local _, CurrentY, _ = CurrentCFrame:ToEulerAnglesYXZ()

        if math.abs(CurrentY - YAngle) > 0.01 then
            WaypointRotationTween = TweenService:Create(Thing.PrimaryPart, TweenInfo.new(0.3, Enum.EasingStyle.Linear), {
                CFrame = CFrame.new(Vector3.new(CurrentCFrame.Position.X, TargetPosition.Y, CurrentCFrame.Position.Z)) * CFrame.Angles(0, YAngle, 0)
            })
            WaypointRotationTween:Play()
            WaypointRotationTween.Completed:Wait()
        else
            Thing:SetPrimaryPartCFrame(CFrame.new(StartPosition) * CFrame.Angles(0, YAngle, 0))
        end

        WaypointMoveTween = TweenService:Create(Thing.PrimaryPart, TweenInfo.new((TargetPosition - StartPosition).Magnitude / 8, Enum.EasingStyle.Linear), {
            CFrame = CFrame.new(TargetPosition) * CFrame.Angles(0, YAngle, 0)
        })
        WaypointMoveTween:Play()
        WaypointMoveTween.Completed:Wait()

        local FinalOrientation = Waypoint.Orientation
        local FinalCFrame = CFrame.new(TargetPosition) * CFrame.Angles(
            math.rad(FinalOrientation.X),
            math.rad(FinalOrientation.Y),
            math.rad(FinalOrientation.Z)
        )

        if not ThingData[Thing] then
            Thing:SetPrimaryPartCFrame(FinalCFrame)
        end
    end

    if not ThingData[Thing] then
        ThingAttachment:Destroy()
        ProximityPrompt:Destroy()
        if WalkTrack then
            WalkTrack:Stop()
        end
        Thing:Destroy()
    end
end

-- Main spawning loop (now with 3 second delay between spawns)
while task.wait(3) do
    task.spawn(AddRandomThing)
end
