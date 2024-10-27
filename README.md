-- Function to check and auto-upgrade bakery level
local function CheckAndUpgradeBakery()
    local currentLevel = Library.Variables.MyBakery.bakeryLevel
    local experience = Library.Variables.MyBakery.experience
    local requiredExperience = Library.Experience.BakeryExperienceToLevel(currentLevel + 1)

    if experience >= requiredExperience then
        Library.Network.Fire("RequestBakeryLevelUp")
        print("Bakery leveled up to level " .. (currentLevel + 1))
    end
end

-- Periodically check for upgrades
task.spawn(function()
    while true do
        CheckAndUpgradeBakery()
        wait(10) -- Check every 10 seconds
    end
end)

-- Enhanced Customer Handling with Fast Seating and Ordering
Customer.ChangeToWaitForOrderState = function(customer)
    if not FastOrder then 
        Original_ChangeToWaitForOrderState(customer)
        return
    end

    if customer.state ~= "WalkingToSeat" then return end
    
    local seatLeaf = customer:EntityTable()[customer.stateData.seatUID]
    
    if seatLeaf.isDeleted then
        customer:ForcedToLeave()
        return
    end

    customer:SetCustomerState("ThinkingAboutOrder")
    customer:SitInSeat(seatLeaf).Completed:Connect(function()
        customer:ReadMenu()
        wait(0.1)

        if customer.isDeleted then return end
        
        customer:StopReadingMenu()
        customer:SetCustomerState("DecidedOnOrder")

        -- Notify all group members to ready up
        for _, partner in ipairs(customer.stateData.queueGroup) do
            if not partner.isDeleted then
                partner:ReadyToOrder()
            end
        end
    end)
end

-- Auto-collect gifts and optimize for faster gathering
task.spawn(function()
    while AutoGift do
        local gifts = Workspace:GetChildren() -- adjust if gifts are located in a specific folder
        for _, gift in ipairs(gifts) do
            if gift.Name == "SantaPresent" and gift:IsA("Part") then
                Customer.DropPresent(gift)
                wait(0.5) -- slight delay to avoid overwhelming the server
            end
        end
        wait(5) -- periodic check every 5 seconds
    end
end)

-- Enhance Waiter delivery times
Waiter.CheckForFoodDelivery = function(waiter)
    if not GoldFood then 
        return Original_CheckForFoodDelivery(waiter)
    end

    -- Optimize food delivery logic; improve wander time
    local orderStand = waiter:GetMyFloor():GatherOrderStandsWithDeliveryReady()[1]
    if orderStand then
        waiter.state = "WalkingToPickupFood"
        waiter:WalkToNewFloor(orderStand:GetMyFloor(), function()
            waiter:WalkToPoint(orderStand.Position.X, orderStand.Position.Y, orderStand.Position.Z, function()
                waiter:HoldFood(orderStand.stateData.foodReadyList[1].ID, true) -- holding gold food
                waiter.state = "WalkingToDeliverFood"
                waiter:WalkToNewFloor(customers[1]:GetMyFloor(), function()
                    waiter:DropFood()
                    waiter.state = "Idle"
                end)
            end)
        end)
    end
end
