## Ensure Ready
```lua
-- server side:
exports["mf-inventory"]:onReady(function()
  print("Inventory is ready to be used.")
end)
```

## Get Inventory Data
```lua
-- server side:
local inv = exports["mf-inventory"]:getInventory(identifier)
  -- {
  --   identifier = "",
  --   weight = 0.0,
  --   maxWeight = 50.0,
  --   maxSlots = 40,
  --   items = {}
  -- }
```

## Get Inventory Items
```lua
-- client side:
exports["mf-inventory"]:getInventoryItems(identifier,function(items)
  -- Do something with items here...
end)

-- server-side:
local inventory = exports["mf-inventory"]:getInventoryItems(identifier)
```

## New Inventory
You can create and access a new inventory (e.g: housing) or a temporary (e.g: admin item dump) inventory through a few simple function calls, as shown below.
If you have a pre-existing items table (general ESX inventory layout), you can convert the items to the format required by the inventory by calling the export "buildInventoryItems", E.G:

```lua
-- Server side, called once for creation:
local uniqueIdentifier = "houseUniqueIdentifier:123:ABC"          -- Unique identifier for this inventory.
local inventoryType = "inventory"                                 -- Inventory type. Default is "inventory", other types are "shop" and "recipe".
local inventorySubType = "housing"                                -- Inventory sub-type, used to modify degrade modifiers by the config table.
local inventoryLabel = "house_storage"                            -- The inventorys UI label index (which will pull the translation value).
local maxWeight = 250.0                                           -- Max weight for the inventory.
local maxSlots = 50                                               -- Max slots for the inventory.
local items = exports["mf-inventory"]:buildInventoryItems({       -- Construct table of valid items for the inventory from pre-existing ESX items table (OR a blank table/{}).
  {
    name = 'water_bottle',
    label = 'Water Bottle',
    count = 5
  }
})

-- To create an inventory that will save to the database:
exports["mf-inventory"]:createInventory(uniqueIdentifier,inventoryType,inventorySubType,inventoryLabel,maxWeight,maxSlots,items)
-- To create a temporary inventory that will never save to the database (think disposable: item dump for admins).
local tempInvId = exports["mf-inventory"]:createTemporaryInventory(inventoryType,inventorySubType,inventoryLabel,maxWeight,maxSlots,items)
```

```lua
-- Client side, called any time you want to open the inventory:
exports["mf-inventory"]:openOtherInventory("houseUniqueIdentifier:123:ABC")
```

NOTE: This must be called AFTER the script has authorized.

## New Vehicle Inventory
When a new vehicle is purchased, you should create an inventory for it by triggering the export, passing the vehicles plate, and optionally the vehicles class/category OR model hash.
```lua
exports["mf-inventory"]:registerVehicleInventory(plate,class,modelHash)
```

Alternative net event:
```lua
TriggerServerEvent("inventory:registerVehicleInventory",plate,class,modelHash)
```

## Adding container item with content directly to player inventory:
```lua
local xPlayer = ESX.GetPlayerFromId(source)
local added = exports['mf-inventory']:addContainerItem(xPlayer.identifier,"basic_tools",{
  {
    name = "lockpick",
    count = 5
  },
  {
    name = "water_bottle",
    count = 1
  }
})

if added then
  print("Item added to inventory.")
else
  print("Adding item to inventory failed.")
end
```

## Shops
You can create shops that allow both buying and selling for fixed values via the config.
While the inventory logic is handled automatically, you will still need to implement a way to open these shops via the export:

```lua
RegisterCommand('inventory:openOther',function(source,args)
  local identifier = args and args[1] or "example_shop:1"
  exports["mf-inventory"]:openOtherInventory(identifier)
end)
```

To create a shop, check out the config.lua and read the example shop provided.

## Crafting
You can create recipes sets which contain a list of recipes, directly craftable from the inventory via the config.
Again, crafting logic is all handled internally but you will need to implement a method for opening the recipe set, e.g:

```lua
RegisterCommand('inventory:openCrafting',function(source,args)
  local identifier = args and args[1] or "example_recipe"
  exports["mf-inventory"]:openCrafting(identifier)
end)
```

To create a recipe set, check out the config.lua and read the example recipe set provided.

## Minigame
A small minigame is included. Press space to use. Example usage:

```lua
RegisterCommand('inventory:minigame',function(source,args)
  local count = 4
  local speed = 0.6
  exports["mf-inventory"]:startMinigame(count,speed,function(res)
    print("Minigame complete",res)
  end)
end)
```

## Forcing inventories to save
Inventories save to the database on a thread roughly every 5 minutes (only as required). To force inventories to save immediately, call the export:
```lua
exports["mf-inventory"]:saveInventories(function()
  print("Inventories saved.")
end)

-- Example for txAdmin:
AddEventHandler('txAdmin:events:scheduledRestart',function(eventData)
  if eventData.secondsRemaining == 30 then
    exports["mf-inventory"]:saveInventories()
  end
end)
```

Allow some time for sql operations before closing server after this function is called.

## Forcing individual inventory to save
```lua
exports["mf-inventory"]:saveInventory(uniqueIdentifier)
```

## Deleting inventories
You can delete inventories that you've created from the server side by calling the export:
```lua
exports["mf-inventory"]:deleteInventory(uniqueIdentifier)
```

## Register usable item
ESX.RegisterUsableItem is now passed a callback function to remove the item that was used. Optional param count of items to remove.
The item being used is also passed.
```lua
ESX.RegisterUsableItem('meth_raw',function(source,removeCb,item)
  print(json.encode(item))
  
  local removeCount = 2
  removeCb(removeCount)
end)
```

## Account money as inventory items
To have account money show up as an inventory item, you need to add the account name to the config.lua table `Config.DisplayAccounts`, and insert the account name as an item in the database.
The default ESX 'money' and 'black_money' items have been provided by default in the database and in the config.
