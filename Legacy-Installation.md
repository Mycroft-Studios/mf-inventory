The functions listed below are taken from the default ESX resources, giving you both the original function to search for and the new function with the required changes made. You're best off looking at the changes yourself and modifying your own code so you don't overwrite any existing modifications required by other resources.

## FILE: es_extended/client/functions.lua
<details>
<summary><b>ESX.ShowInventory</b></summary>

<details>
<summary>Original</summary>

```lua
ESX.ShowInventory = function()
	local playerPed = ESX.PlayerData.ped
	local elements, currentWeight = {}, 0

	for k,v in pairs(ESX.PlayerData.accounts) do
		if v.money > 0 then
		local formattedMoney = _U('locale_currency', ESX.Math.GroupDigits(v.money))
			local canDrop = v.name ~= 'bank'

			table.insert(elements, {
				label = ('%s: <span style="color:green;">%s</span>'):format(v.label, formattedMoney),
				count = v.money,
				type = 'item_account',
				value = v.name,
				usable = false,
				rare = false,
				canRemove = canDrop
			})
		end
	end

	for k,v in ipairs(ESX.PlayerData.inventory) do
		if v.count > 0 then
			currentWeight = currentWeight + (v.weight * v.count)

			table.insert(elements, {
				label = ('%s x%s'):format(v.label, v.count),
				count = v.count,
				type = 'item_standard',
				value = v.name,
				usable = v.usable,
				rare = v.rare,
				canRemove = v.canRemove
			})
		end
	end

	for k,v in ipairs(Config.Weapons) do
		local weaponHash = GetHashKey(v.name)

		if HasPedGotWeapon(playerPed, weaponHash, false) then
			local ammo, label = GetAmmoInPedWeapon(playerPed, weaponHash)

			if v.ammo then
				label = ('%s - %s %s'):format(v.label, ammo, v.ammo.label)
			else
				label = v.label
			end

			table.insert(elements, {
				label = label,
				count = 1,
				type = 'item_weapon',
				value = v.name,
				usable = false,
				rare = false,
				ammo = ammo,
				canGiveAmmo = (v.ammo ~= nil),
				canRemove = true
			})
		end
	end

	ESX.UI.Menu.CloseAll()

	ESX.UI.Menu.Open('default', GetCurrentResourceName(), 'inventory', {
		title    = _U('inventory', currentWeight, ESX.PlayerData.maxWeight),
		align    = 'bottom-right',
		elements = elements
	}, function(data, menu)
		menu.close()
		local player, distance = ESX.Game.GetClosestPlayer()
		elements = {}

		if data.current.usable then
			table.insert(elements, {label = _U('use'), action = 'use', type = data.current.type, value = data.current.value})
		end

		if data.current.canRemove then
			if player ~= -1 and distance <= 3.0 then
				table.insert(elements, {label = _U('give'), action = 'give', type = data.current.type, value = data.current.value})
			end

			table.insert(elements, {label = _U('remove'), action = 'remove', type = data.current.type, value = data.current.value})
		end

		if data.current.type == 'item_weapon' and data.current.canGiveAmmo and data.current.ammo > 0 and player ~= -1 and distance <= 3.0 then
			table.insert(elements, {label = _U('giveammo'), action = 'give_ammo', type = data.current.type, value = data.current.value})
		end

		table.insert(elements, {label = _U('return'), action = 'return'})

		ESX.UI.Menu.Open('default', GetCurrentResourceName(), 'inventory_item', {
			title    = data.current.label,
			align    = 'bottom-right',
			elements = elements,
		}, function(data1, menu1)
			local item, type = data1.current.value, data1.current.type

			if data1.current.action == 'give' then
				local playersNearby = ESX.Game.GetPlayersInArea(GetEntityCoords(playerPed), 3.0)

				if #playersNearby > 0 then
					local players = {}
					elements = {}

					for k,playerNearby in ipairs(playersNearby) do
						players[GetPlayerServerId(playerNearby)] = true
					end

					ESX.TriggerServerCallback('esx:getPlayerNames', function(returnedPlayers)
						for playerId,playerName in pairs(returnedPlayers) do
							table.insert(elements, {
								label = playerName,
								playerId = playerId
							})
						end

						ESX.UI.Menu.Open('default', GetCurrentResourceName(), 'give_item_to', {
							title    = _U('give_to'),
							align    = 'bottom-right',
							elements = elements
						}, function(data2, menu2)
							local selectedPlayer, selectedPlayerId = GetPlayerFromServerId(data2.current.playerId), data2.current.playerId
							playersNearby = ESX.Game.GetPlayersInArea(GetEntityCoords(playerPed), 3.0)
							playersNearby = ESX.Table.Set(playersNearby)

							if playersNearby[selectedPlayer] then
								local selectedPlayerPed = GetPlayerPed(selectedPlayer)

								if IsPedOnFoot(selectedPlayerPed) and not IsPedFalling(selectedPlayerPed) then
									if type == 'item_weapon' then
										TriggerServerEvent('esx:giveInventoryItem', selectedPlayerId, type, item, nil)
										menu2.close()
										menu1.close()
									else
										ESX.UI.Menu.Open('dialog', GetCurrentResourceName(), 'inventory_item_count_give', {
											title = _U('amount')
										}, function(data3, menu3)
											local quantity = tonumber(data3.value)

											if quantity and quantity > 0 and data.current.count >= quantity then
												TriggerServerEvent('esx:giveInventoryItem', selectedPlayerId, type, item, quantity)
												menu3.close()
												menu2.close()
												menu1.close()
											else
												ESX.ShowNotification(_U('amount_invalid'))
											end
										end, function(data3, menu3)
											menu3.close()
										end)
									end
								else
									ESX.ShowNotification(_U('in_vehicle'))
								end
							else
								ESX.ShowNotification(_U('players_nearby'))
								menu2.close()
							end
						end, function(data2, menu2)
							menu2.close()
						end)
					end, players)
				else
					ESX.ShowNotification(_U('players_nearby'))
				end
			elseif data1.current.action == 'remove' then
				if IsPedOnFoot(playerPed) and not IsPedFalling(playerPed) then
					local dict, anim = 'weapons@first_person@aim_rng@generic@projectile@sticky_bomb@', 'plant_floor'
					ESX.Streaming.RequestAnimDict(dict)

					if type == 'item_weapon' then
						menu1.close()
						TaskPlayAnim(playerPed, dict, anim, 8.0, 1.0, 1000, 16, 0.0, false, false, false)
						Citizen.Wait(1000)
						TriggerServerEvent('esx:removeInventoryItem', type, item)
					else
						ESX.UI.Menu.Open('dialog', GetCurrentResourceName(), 'inventory_item_count_remove', {
							title = _U('amount')
						}, function(data2, menu2)
							local quantity = tonumber(data2.value)

							if quantity and quantity > 0 and data.current.count >= quantity then
								menu2.close()
								menu1.close()
								TaskPlayAnim(playerPed, dict, anim, 8.0, 1.0, 1000, 16, 0.0, false, false, false)
								Citizen.Wait(1000)
								TriggerServerEvent('esx:removeInventoryItem', type, item, quantity)
							else
								ESX.ShowNotification(_U('amount_invalid'))
							end
						end, function(data2, menu2)
							menu2.close()
						end)
					end
				end
			elseif data1.current.action == 'use' then
				TriggerServerEvent('esx:useItem', item)
			elseif data1.current.action == 'return' then
				ESX.UI.Menu.CloseAll()
				ESX.ShowInventory()
			elseif data1.current.action == 'give_ammo' then
				local closestPlayer, closestDistance = ESX.Game.GetClosestPlayer()
				local closestPed = GetPlayerPed(closestPlayer)
				local pedAmmo = GetAmmoInPedWeapon(playerPed, GetHashKey(item))

				if IsPedOnFoot(closestPed) and not IsPedFalling(closestPed) then
					if closestPlayer ~= -1 and closestDistance < 3.0 then
						if pedAmmo > 0 then
							ESX.UI.Menu.Open('dialog', GetCurrentResourceName(), 'inventory_item_count_give', {
								title = _U('amountammo')
							}, function(data2, menu2)
								local quantity = tonumber(data2.value)

								if quantity and quantity > 0 then
									if pedAmmo >= quantity then
										TriggerServerEvent('esx:giveInventoryItem', GetPlayerServerId(closestPlayer), 'item_ammo', item, quantity)
										menu2.close()
										menu1.close()
									else
										ESX.ShowNotification(_U('noammo'))
									end
								else
									ESX.ShowNotification(_U('amount_invalid'))
								end
							end, function(data2, menu2)
								menu2.close()
							end)
						else
							ESX.ShowNotification(_U('noammo'))
						end
					else
						ESX.ShowNotification(_U('players_nearby'))
					end
				else
					ESX.ShowNotification(_U('in_vehicle'))
				end
			end
		end, function(data1, menu1)
			ESX.UI.Menu.CloseAll()
			ESX.ShowInventory()
		end)
	end, function(data, menu)
		menu.close()
	end)
end
```

</details>
<details>
<summary>Modified</summary>

```lua
ESX.ShowInventory = function()
  exports["mf-inventory"]:openInventory()
end
```

</details>
</details>

## FILE: es_extended/client/main.lua
<details>
<summary><b>ShowInventory Loop</b></summary>

<details>
<summary>Original</summary>

```lua
if Config.EnableDefaultInventory then
	RegisterCommand('showinv', function()
		if not ESX.PlayerData.dead and not ESX.UI.Menu.IsOpen('default', 'es_extended', 'inventory') then
			ESX.ShowInventory()
		end
	end)

	RegisterKeyMapping('showinv', _U('keymap_showinventory'), 'keyboard', 'F2')
end
```

</details>
<details>
<summary>Modified</summary>

```lua
Citizen.CreateThread(function()
	while true do
		Citizen.Wait(0)

		if IsControlJustReleased(0, 289) then
			if IsInputDisabled(0) and not isDead and not ESX.UI.Menu.IsOpen('default', 'es_extended', 'inventory') then
				ESX.ShowInventory()
			end
		end
	end
end)
```

</details>
</details>
<details>
<summary><b>StartServerSyncLoops</b></summary>

<details>
<summary>Original</summary>

```lua
function StartServerSyncLoops()
	-- keep track of ammo
	Citizen.CreateThread(function()
		local currentWeapon = {timer=0}
		while ESX.PlayerLoaded do
			local sleep = 5

			if currentWeapon.timer == sleep then
				local ammoCount = GetAmmoInPedWeapon(ESX.PlayerData.ped, currentWeapon.hash)
				TriggerServerEvent('esx:updateWeaponAmmo', currentWeapon.name, ammoCount)
				currentWeapon.timer = 0
			elseif currentWeapon.timer > sleep then
				currentWeapon.timer = currentWeapon.timer - sleep
			end

			if IsPedArmed(ESX.PlayerData.ped, 4) then
				if IsPedShooting(ESX.PlayerData.ped) then
					local _,weaponHash = GetCurrentPedWeapon(ESX.PlayerData.ped, true)
					local weapon = ESX.GetWeaponFromHash(weaponHash)

					if weapon then
						currentWeapon.name = weapon.name
						currentWeapon.hash = weaponHash	
						currentWeapon.timer = 100 * sleep		
					end
				end
			else
				sleep = 200
			end
			Citizen.Wait(sleep)
		end
	end)

	-- sync current player coords with server
	Citizen.CreateThread(function()
		local previousCoords = vector3(ESX.PlayerData.coords.x, ESX.PlayerData.coords.y, ESX.PlayerData.coords.z)

		while ESX.PlayerLoaded do
			local playerPed = PlayerPedId()
			if ESX.PlayerData.ped ~= playerPed then ESX.SetPlayerData('ped', playerPed) end

			if DoesEntityExist(ESX.PlayerData.ped) then
				local playerCoords = GetEntityCoords(ESX.PlayerData.ped)
				local distance = #(playerCoords - previousCoords)

				if distance > 1 then
					previousCoords = playerCoords
					local playerHeading = ESX.Math.Round(GetEntityHeading(ESX.PlayerData.ped), 1)
					local formattedCoords = {x = ESX.Math.Round(playerCoords.x, 1), y = ESX.Math.Round(playerCoords.y, 1), z = ESX.Math.Round(playerCoords.z, 1), heading = playerHeading}
					TriggerServerEvent('esx:updateCoords', formattedCoords)
				end
			end
			Citizen.Wait(1500)
		end
	end)
end
```

</details>
<details>
<summary>Modified</summary>

```lua
function StartServerSyncLoops()
  -- keep track of ammo
  Citizen.CreateThread(function()
    while true do
      Citizen.Wait(0)

      if isDead then
        Citizen.Wait(500)
      else
        local playerPed = PlayerPedId()

        if IsPedShooting(playerPed) then
          local _,weaponHash = GetCurrentPedWeapon(playerPed, true)
          local weapon = ESX.GetWeaponFromHash(weaponHash)

          if weapon then
            local ammoCount = GetAmmoInPedWeapon(playerPed, weaponHash)
            local removeWeapon = false

            if ammoCount <= 0 then
              local damageType = GetWeaponDamageType(weaponHash)

              if damageType == 1
              or damageType == 5
              or damageType == 6
              or damageType == 12
              or damageType == 13
              then                
                Wait(1000)

                if not HasPedGotWeapon(playerPed, weaponHash, false) then
                  removeWeapon = true
                end
              end
            end

            TriggerServerEvent('esx:updateWeaponAmmo', weapon.name, ammoCount, removeWeapon)
          end
        end
      end
    end
  end)

  -- sync current player coords with server
  Citizen.CreateThread(function()
    local previousCoords = vector3(ESX.PlayerData.coords.x, ESX.PlayerData.coords.y, ESX.PlayerData.coords.z)

    while true do
      Citizen.Wait(1000)
      local playerPed = PlayerPedId()

      if DoesEntityExist(playerPed) then
        local playerCoords = GetEntityCoords(playerPed)
        local distance = #(playerCoords - previousCoords)

        if distance > 1 then
          previousCoords = playerCoords
          local playerHeading = ESX.Math.Round(GetEntityHeading(playerPed), 1)
          local formattedCoords = {x = ESX.Math.Round(playerCoords.x, 1), y = ESX.Math.Round(playerCoords.y, 1), z = ESX.Math.Round(playerCoords.z, 1), heading = playerHeading}
          TriggerServerEvent('esx:updateCoords', formattedCoords)
        end
      end
    end
  end)
end
```

</details>
</details>

## FILE: es_extended/server/functions.lua
<details>
<summary><b>ESX.UseItem</b></summary>

<details>
<summary>Original</summary>

```lua
ESX.UseItem = function(source, item)
  ESX.UsableItemsCallbacks[item](source, item)
end
```

</details><details>
<summary>Modified</summary>

```lua
ESX.UseItem = function(source, item, remove, ...)
  if ESX.UsableItemsCallbacks[item] then
    ESX.UsableItemsCallbacks[item](source,remove,...)
  end
end
```

</details>
</details>

<details>
<summary><b>ESX.SavePlayer</b></summary>

<details>
<summary>Original</summary>

```lua
ESX.SavePlayer = function(xPlayer, cb)
	local asyncTasks = {}

	table.insert(asyncTasks, function(cb2)
		MySQL.Async.execute(savePlayers, {
			json.encode(xPlayer.getAccounts(true)),
			xPlayer.job.name,
			xPlayer.job.grade,
			xPlayer.getGroup(),
			json.encode(xPlayer.getCoords()),
			json.encode(xPlayer.getInventory(true)),
			json.encode(xPlayer.getLoadout(true)),
			xPlayer.getIdentifier()
		}, function(rowsChanged)
			cb2()
		end)
	end)

	Async.parallel(asyncTasks, function(results)
		print(('[^2INFO^7] Saved player ^5"%s^7"'):format(xPlayer.getName()))

		if cb then
			cb()
		end
	end)
end
```

</details><details>
<summary>Modified</summary>

```lua

ESX.SavePlayer = function(xPlayer, cb)
	local asyncTasks = {}

	table.insert(asyncTasks, function(cb2)
		MySQL.Async.execute(savePlayers, {
			json.encode(xPlayer.getAccounts(true)),
			xPlayer.job.name,
			xPlayer.job.grade,
			xPlayer.getGroup(),
			json.encode(xPlayer.getCoords()),
			json.encode(xPlayer.getInventory(true)),
			json.encode(xPlayer.getLoadout(true)),
			xPlayer.getIdentifier()
		}, function(rowsChanged)
			cb2()
		end)
    exports["mf-inventory"]:saveInventory(xPlayer.getIdentifier())
	end)

	Async.parallel(asyncTasks, function(results)
		print(('[^2INFO^7] Saved player ^5"%s^7"'):format(xPlayer.getName()))

		if cb then
			cb()
		end
	end)
end
```

</details>
</details>

## FILE: es_extended/server/commands.lua
<details>
<summary><b>clearinventory</b></summary>

<details>
<summary>Original</summary>

```lua
ESX.RegisterCommand('clearinventory', 'admin', function(xPlayer, args, showError)
  for k,v in ipairs(args.playerId.inventory) do
    if v.count > 0 then
      args.playerId.setInventoryItem(v.name, 0)
    end
  end
end, true, {help = _U('command_clearinventory'), validate = true, arguments = {
  {name = 'playerId', help = _U('commandgeneric_playerid'), type = 'player'}
}})
```

</details>

<details>
<summary>Modified</summary>

```lua

ESX.RegisterCommand('clearinventory', 'admin', function(xPlayer, args, showError)
  exports["mf-inventory"]:clearInventory(args.playerId.identifier)
end, true, {help = _U('command_clearinventory'), validate = true, arguments = {
  {name = 'playerId', help = _U('commandgeneric_playerid'), type = 'player'}
}})
```

</details>
</details>

<details>
<summary><b>clearloadout</b></summary>

<details>
<summary>Original</summary>

```lua
ESX.RegisterCommand('clearloadout', 'admin', function(xPlayer, args, showError)
	for i=#args.playerId.loadout, 1, -1 do
		args.playerId.removeWeapon(args.playerId.loadout[i].name)
	end
end, true, {help = _U('command_clearloadout'), validate = true, arguments = {
	{name = 'playerId', help = _U('commandgeneric_playerid'), type = 'player'}
}})
```

</details>

<details>
<summary>Modified</summary>

```lua
ESX.RegisterCommand('clearloadout', 'admin', function(xPlayer, args, showError)
  for i=#args.playerId.loadout, 1, -1 do
    args.playerId.removeWeapon(v.name,v.ammo,true)
  end

  exports["mf-inventory"]:clearLoadout(args.playerId.identifier)
end, true, {help = _U('command_clearloadout'), validate = true, arguments = {
  {name = 'playerId', help = _U('commandgeneric_playerid'), type = 'player'}
}})
```

</details>
</details>

<details>
<summary><b>giveitem</b></summary>

<details>
<summary>Original</summary>

```lua
ESX.RegisterCommand('giveitem', 'admin', function(xPlayer, args, showError)
	args.playerId.addInventoryItem(args.item, args.count)
end, true, {help = _U('command_giveitem'), validate = true, arguments = {
	{name = 'playerId', help = _U('commandgeneric_playerid'), type = 'player'},
	{name = 'item', help = _U('command_giveitem_item'), type = 'item'},
	{name = 'count', help = _U('command_giveitem_count'), type = 'number'}
}})
```

</details>

<details>
<summary>Modified</summary>

```lua
ESX.RegisterCommand('giveitem', 'admin', function(xPlayer, args, showError)
  local item = args.item:lower()
  local IsWeapon = false
  for i=1,#Config.Weapons do
    if Config.Weapons[i].name:lower() == item then
      IsWeapon = true
    end
  end

  if IsWeapon then 
    args.playerId.addWeapon(args.item, args.count)
  else
    args.playerId.addInventoryItem(args.item, args.count)
  end
end, true, {help = _U('command_giveitem'), validate = true, arguments = {
  {name = 'playerId', help = _U('commandgeneric_playerid'), type = 'player'},
  {name = 'item', help = _U('command_giveitem_item'), type = 'item'},
  {name = 'count', help = _U('command_giveitem_count'), type = 'number'}
}})
```

</details>
</details>

## es_extended/server/main.lua
<details>
<summary>print invalid item</summary>

<details>
<summary>Original</summary>

```lua
print(('[^3WARNING^7] Ignoring invalid item "%s" for "%s"'):format(name, identifier))
```

</details>

<details>
<summary>Modified</summary>

```lua
-- print(('[^3WARNING^7] Ignoring invalid item "%s" for "%s"'):format(name, identifier))
```

</details>
</details>

<details>
<summary>esx:updateWeaponAmmo</summary>

<details>
<summary>Original</summary>

```lua
RegisterNetEvent('esx:updateWeaponAmmo')
AddEventHandler('esx:updateWeaponAmmo', function(weaponName, ammoCount)
  local xPlayer = ESX.GetPlayerFromId(source)

  if xPlayer then
    xPlayer.updateWeaponAmmo(weaponName, ammoCount)
  end
end)
```

</details>

<details>
<summary>Modified</summary>

```lua
RegisterNetEvent('esx:updateWeaponAmmo')
AddEventHandler('esx:updateWeaponAmmo', function(weaponName, ammoCount, removeWeapon)
  local xPlayer = ESX.GetPlayerFromId(source)

  if xPlayer then
    xPlayer.updateWeaponAmmo(weaponName, ammoCount, removeWeapon)
  end
end)
```

</details>
</details>

## es_extended/server/classes/player.lua
<details><summary>self.setInventory</summary>

> Does Not Exist In Original, Add This Function In
```lua
self.setInventory = function(inv)
  self.inventory = inv
end
```
</details>
<details><summary>self.getInventoryItem</summary>
<details><summary>Original</summary>

```lua
self.getInventoryItem = function(name)
  for k,v in ipairs(self.inventory) do
    if v.name == name then
      return v
    end
  end

  return
end
```

</details><details><summary>Modified</summary>

```lua
self.getInventoryItem = function(name, count, ...)
  return exports["mf-inventory"]:getInventoryItem(self.identifier, name, count, ...)
end
```

</details></details>
<details><summary>self.addInventoryItem</summary>
<details><summary>Original</summary>

```lua
self.addInventoryItem = function(name, count)
  local item = self.getInventoryItem(name)

  if item then
    count = ESX.Math.Round(count)
    item.count = item.count + count
    self.weight = self.weight + (item.weight * count)

    TriggerEvent('esx:onAddInventoryItem', self.source, item.name, item.count)
    self.triggerEvent('esx:addInventoryItem', item.name, item.count)
  end
end
```

</details><details><summary>Modified</summary>

```lua
self.addInventoryItem = function(name, count, quality, ...)
  return exports["mf-inventory"]:addInventoryItem(self.identifier, name, count, self.source, quality, ...)
end
```

</details></details>
<details><summary>self.removeInventoryItem</summary>
<details><summary>Original</summary>

```lua
self.removeInventoryItem = function(name, count)
  local item = self.getInventoryItem(name)

  if item then
    count = ESX.Math.Round(count)
    local newCount = item.count - count

    if newCount >= 0 then
      item.count = newCount
      self.weight = self.weight - (item.weight * count)

      TriggerEvent('esx:onRemoveInventoryItem', self.source, item.name, item.count)
      self.triggerEvent('esx:removeInventoryItem', item.name, item.count)
    end
  end
end
```

</details><details><summary>Modified</summary>

```lua
self.removeInventoryItem = function(name, count, ...)
  return exports["mf-inventory"]:removeInventoryItem(self.identifier, name, count, self.source, ...)
end
```

</details></details>
<details><summary>self.addWeapon</summary>
<details><summary>Original</summary>

```lua
self.addWeapon = function(weaponName, ammo)
  if not self.hasWeapon(weaponName) then
    local weaponLabel = ESX.GetWeaponLabel(weaponName)

    table.insert(self.loadout, {
      name = weaponName,
      ammo = ammo,
      label = weaponLabel,
      components = {},
      tintIndex = 0
    })

    self.triggerEvent('esx:addWeapon', weaponName, ammo)
    self.triggerEvent('esx:addInventoryItem', weaponLabel, false, true)
  end
end
```

</details><details><summary>Modified</summary>

```lua
self.addWeapon = function(weaponName, ammo, ignoreInventory)
  if not self.hasWeapon(weaponName) then
    local weaponLabel = ESX.GetWeaponLabel(weaponName)

    table.insert(self.loadout, {
      name = weaponName,
      ammo = ammo,
      label = weaponLabel,
      components = {},
      tintIndex = 0
    })

    self.triggerEvent('esx:addWeapon', weaponName, ammo)
    self.triggerEvent('esx:addInventoryItem', weaponLabel, false, true)

    if not ignoreInventory then
      exports["mf-inventory"]:addInventoryItem(self.identifier,weaponName,1,self.source)
    end
  end
end
```

</details></details>
<details><summary>self.removeWeapon</summary>
<details><summary>Original</summary>

```lua
self.removeWeapon = function(weaponName)
  local weaponLabel

  for k,v in ipairs(self.loadout) do
    if v.name == weaponName then
      weaponLabel = v.label

      for k2,v2 in ipairs(v.components) do
        self.removeWeaponComponent(weaponName, v2)
      end

      table.remove(self.loadout, k)
      break
    end
  end

  if weaponLabel then
    self.triggerEvent('esx:removeWeapon', weaponName)
    self.triggerEvent('esx:removeInventoryItem', weaponLabel, false, true)
  end
end
```

</details><details><summary>Modified</summary>

```lua
self.removeWeapon = function(weaponName, ammo, ignoreInventory)
  local weaponLabel

  for k,v in ipairs(self.loadout) do
    if v.name == weaponName then
      weaponLabel = v.label

      for k2,v2 in ipairs(v.components) do
        self.removeWeaponComponent(weaponName, v2)
      end

      table.remove(self.loadout, k)
      break
    end
  end

  if weaponLabel then
    self.triggerEvent('esx:removeWeapon', weaponName, ammo)
    self.triggerEvent('esx:removeInventoryItem', weaponLabel, false, true)

    if not ignoreInventory then
      exports["mf-inventory"]:removeInventoryItem(self.identifier,weaponName,1,self.source)
    end
  end
end
```

</details></details>
<details><summary>self.setAccountMoney</summary>
<details><summary>Original</summary>

```lua
self.setAccountMoney = function(accountName, money)
  if money >= 0 then
    local account = self.getAccount(accountName)

    if account then
      local prevMoney = account.money
      local newMoney = ESX.Math.Round(money)
      account.money = newMoney

      self.triggerEvent('esx:setAccountMoney', account)
    end
  end
end
```

</details><details><summary>Modified</summary>

```lua
self.setAccountMoney = function(accountName, money)
  if money >= 0 then
    local account = self.getAccount(accountName)

    if account then
      local prevMoney = account.money
      local newMoney = ESX.Math.Round(money)
      account.money = newMoney

      self.triggerEvent('esx:setAccountMoney', account)

      exports["mf-inventory"]:setAccountMoney(self.source,self.identifier,accountName,account.money)
    end
  end
end
```

</details></details>
<details><summary>self.addAccountMoney</summary>
<details><summary>Original</summary>

```lua
self.addAccountMoney = function(accountName, money)
  if money > 0 then
    local account = self.getAccount(accountName)

    if account then
      local newMoney = account.money + ESX.Math.Round(money)
      account.money = newMoney

      self.triggerEvent('esx:setAccountMoney', account)
    end
  end
end
```

</details><details><summary>Modified</summary>

```lua  
self.addAccountMoney = function(accountName, money, ignoreInventory)
  if money > 0 then
    local account = self.getAccount(accountName)

    if account then
      local newMoney = account.money + ESX.Math.Round(money)
      account.money = newMoney

      self.triggerEvent('esx:setAccountMoney', account)

      if not ignoreInventory then
        exports["mf-inventory"]:setAccountMoney(self.source,self.identifier,accountName,account.money)
      end
    end
  end
end
```

</details></details>
<details><summary>self.removeAccountMoney</summary>
<details><summary>Original</summary>

```lua
self.removeAccountMoney = function(accountName, money)
  if money > 0 then
    local account = self.getAccount(accountName)

    if account then
      local newMoney = account.money - ESX.Math.Round(money)
      account.money = newMoney

      self.triggerEvent('esx:setAccountMoney', account)
    end
  end
end
```

</details><details><summary>Modified</summary>

```lua
self.removeAccountMoney = function(accountName, money, ignoreInventory)
  if money > 0 then
    local account = self.getAccount(accountName)

    if account then
      local newMoney = account.money - ESX.Math.Round(money)
      account.money = newMoney

      self.triggerEvent('esx:setAccountMoney', account)

      if not ignoreInventory then
        exports["mf-inventory"]:setAccountMoney(self.source,self.identifier,accountName,account.money)
      end
    end
  end
end
```

</details></details>
<details><summary>self.canCarryItem</summary>
<details><summary>Original</summary>

```lua
self.canCarryItem = function(name, count)
	local currentWeight, itemWeight = self.weight, ESX.Items[name].weight
	local newWeight = currentWeight + (itemWeight * count)

	return newWeight <= self.maxWeight
end
```

</details><details><summary>Modified</summary>

```lua
self.canCarryItem = function(...)
  return exports['mf-inventory']:canCarry(self.identifier,...)
end
```

</details></details>
<details><summary>self.canSwapItem</summary>
<details><summary>Original</summary>

```lua
self.canSwapItem = function(firstItem, firstItemCount, testItem, testItemCount)
  local firstItemObject = self.getInventoryItem(firstItem)
  local testItemObject = self.getInventoryItem(testItem)

  if firstItemObject.count >= firstItemCount then
    local weightWithoutFirstItem = ESX.Math.Round(self.weight - (firstItemObject.weight * firstItemCount))
    local weightWithTestItem = ESX.Math.Round(weightWithoutFirstItem + (testItemObject.weight * testItemCount))

    return weightWithTestItem <= self.maxWeight
  end

  return false
end
```

</details><details><summary>Modified</summary>

```lua
self.canSwapItem = function(...)
  return exports['mf-inventory']:canSwap(self.identifier,...)
end
```

</details></details>
<details><summary>self.updateWeaponAmmo</summary>
<details><summary>Original</summary>

```lua
self.updateWeaponAmmo = function(weaponName, ammoCount)
  local loadoutNum, weapon = self.getWeapon(weaponName)

  if weapon then
    if ammoCount < weapon.ammo then
      weapon.ammo = ammoCount
    end
  end
end
```

</details><details><summary>Modified</summary>

```lua
self.updateWeaponAmmo = function(weaponName, ammoCount, weaponRemoved)
  local loadoutNum, weapon = self.getWeapon(weaponName)

  if weapon then
    if weaponRemoved then
      self.removeWeapon(weaponName)
    elseif ammoCount < weapon.ammo then
      weapon.ammo = ammoCount
    end
  end
end
```

</details></details>