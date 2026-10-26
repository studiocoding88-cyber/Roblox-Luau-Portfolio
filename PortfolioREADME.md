# Chronos Edge — Backend Systems Showcase

A source-code showcase of three Roblox Luau backend modules from **Chronos Edge**:

1. **ItemConfig** — validated item definitions and protected item data.
2. **InventoryService** — inventory queries and validated add/remove operations.
3. **DataService** — player-data normalization, session ownership, serialized saving, autosaving, and shutdown handling.

This README packages the supplied scripts for review and portfolio presentation. It does not claim the modules are a complete game or independently audited.

## Roblox Studio setup

Create this structure in **ServerScriptService**:

```text
ServerScriptService
└── Services (Folder)
    ├── ItemConfig (ModuleScript)
    ├── DataService (ModuleScript)
    └── InventoryService (ModuleScript)
```

Paste each matching code section below into its ModuleScript. The names and folder placement must match because the scripts use `WaitForChild` and `require` with those names.

### Setup notes

- `DataService` uses Roblox `DataStoreService` and the data store `ChronosEdge_PlayerData_v1`.
- To test DataStores in Studio, publish the experience and enable **Game Settings → Security → Enable Studio Access to API Services**. Prefer a separate test experience so test data cannot affect production data.
- `InventoryService` requires `DataService` and `ItemConfig` in the same `Services` folder.
- `InventoryChanged` is a `BindableEvent` for server-side listeners, not a client-facing `RemoteEvent`.
- Inventory changes should be requested through server-validated code; never trust a client to decide item IDs or quantities.

## 1. ItemConfig

**Purpose**
- Defines `HealthPotion`, `GoblinCore`, and `TrainingBlade`.
- Validates item definitions when the module loads.
- Freezes internal definitions and returns copies from item lookup functions.
- Exposes helpers for item IDs, types, equipment slots, stacking, and maximum stack size.

```lua
--==================================================
-- CHRONOS EDGE | ItemConfig
-- Validated, protected item definitions
--==================================================

local ItemConfig = {}

local function isFiniteNumber(value)
	return type(value) == "number"
		and value == value
		and value ~= math.huge
		and value ~= -math.huge
end

local function deepCopy(value, seen)
	if type(value) ~= "table" then
		return value
	end

	seen = seen or {}

	if seen[value] then
		error("ItemConfig cannot copy cyclic tables", 2)
	end

	seen[value] = true

	local result = {}

	for key, child in pairs(value) do
		result[deepCopy(key, seen)] = deepCopy(child, seen)
	end

	seen[value] = nil
	return result
end

local function deepFreeze(value, seen)
	if type(value) ~= "table" then
		return value
	end

	seen = seen or {}

	if seen[value] then
		return value
	end

	seen[value] = true

	for _, child in pairs(value) do
		if type(child) == "table" then
			deepFreeze(child, seen)
		end
	end

	return table.freeze(value)
end

local definitions = {
	HealthPotion = {
		Id = "HealthPotion",
		Name = "Health Potion",
		Description = "Restores health when consumed.",
		Type = "Consumable",
		Rarity = "Common",
		Stackable = true,
		MaxStack = 99,
		SellValue = 10,
		Effects = {
			HealthRestore = 50,
		},
	},

	GoblinCore = {
		Id = "GoblinCore",
		Name = "Goblin Core",
		Description = "A core collected from a defeated Goblin.",
		Type = "Material",
		Rarity = "Common",
		Stackable = true,
		MaxStack = 999,
		SellValue = 5,
	},

	TrainingBlade = {
		Id = "TrainingBlade",
		Name = "Training Blade",
		Description = "A basic blade for new Edgewalkers.",
		Type = "Weapon",
		Slot = "Weapon",
		Rarity = "Common",
		Stackable = false,
		MaxStack = 1,
		SellValue = 25,
		Stats = {
			PhysicalDamage = 10,
		},
		Requirements = {
			Level = 1,
		},
	},
}

local VALID_TYPES = {
	Consumable = true,
	Material = true,
	Weapon = true,
	Armor = true,
	Accessory = true,
}

local VALID_RARITIES = {
	Common = true,
	Uncommon = true,
	Rare = true,
	Epic = true,
	Legendary = true,
	Mythic = true,
	Chrono = true,
}

local VALID_SLOTS = {
	Weapon = true,
	Armor = true,
	Helmet = true,
	Accessory = true,
	Boots = true,
	Gloves = true,
}

local function validateNumericTable(itemId, fieldName, values, minimum)
	if values == nil then
		return
	end

	assert(type(values) == "table",
		itemId .. "." .. fieldName .. " must be a table")

	for key, value in pairs(values) do
		assert(type(key) == "string" and key ~= "",
			itemId .. "." .. fieldName .. " contains an invalid key")

		assert(isFiniteNumber(value) and value >= minimum,
			itemId .. "." .. fieldName .. "." .. key .. " is invalid")
	end
end

local function validateDefinition(itemId, definition)
	assert(type(itemId) == "string" and itemId:match("^[%w_]+$"),
		"Invalid item definition key")

	assert(type(definition) == "table",
		"Definition for " .. itemId .. " must be a table")

	assert(definition.Id == itemId,
		"Item ID mismatch for " .. itemId)

	assert(type(definition.Name) == "string"
		and definition.Name:match("%S"),
		itemId .. " has an invalid Name")

	assert(type(definition.Description) == "string",
		itemId .. " has an invalid Description")

	assert(VALID_TYPES[definition.Type] == true,
		itemId .. " has an invalid Type")

	assert(VALID_RARITIES[definition.Rarity] == true,
		itemId .. " has an invalid Rarity")

	assert(type(definition.Stackable) == "boolean",
		itemId .. " has an invalid Stackable value")

	assert(isFiniteNumber(definition.MaxStack)
		and definition.MaxStack >= 1
		and definition.MaxStack % 1 == 0,
		itemId .. " has an invalid MaxStack")

	assert(definition.Stackable or definition.MaxStack == 1,
		itemId .. " is non-stackable but MaxStack is not 1")

	assert(isFiniteNumber(definition.SellValue)
		and definition.SellValue >= 0,
		itemId .. " has an invalid SellValue")

	if definition.Type == "Weapon"
		or definition.Type == "Armor"
		or definition.Type == "Accessory" then

		assert(type(definition.Slot) == "string"
			and VALID_SLOTS[definition.Slot] == true,
			itemId .. " has an invalid equipment Slot")
	else
		assert(definition.Slot == nil,
			itemId .. " defines a Slot for a non-equipment item")
	end

	validateNumericTable(itemId, "Stats", definition.Stats, 0)
	validateNumericTable(itemId, "Effects", definition.Effects, 0)
	validateNumericTable(itemId, "Requirements", definition.Requirements, 1)

	if definition.Type == "Consumable" then
		assert(type(definition.Effects) == "table"
			and next(definition.Effects) ~= nil,
			itemId .. " must define consumable Effects")
	end

	for key, value in pairs(definition) do
		assert(type(key) == "string",
			itemId .. " contains a non-string property name")

		local valueType = type(value)
		assert(valueType == "string"
			or valueType == "number"
			or valueType == "boolean"
			or valueType == "table",
			itemId .. " contains an unsupported property value")
	end
end

-- Validate all definitions before exposing the module.
for itemId, definition in pairs(definitions) do
	validateDefinition(itemId, definition)
	deepFreeze(definition)
end

table.freeze(definitions)

function ItemConfig.IsValidItem(itemId)
	if type(itemId) ~= "string" then
		return false
	end

	return definitions[itemId] ~= nil
end

function ItemConfig.GetItem(itemId)
	if type(itemId) ~= "string" then
		return nil
	end

	local definition = definitions[itemId]
	if not definition then
		return nil
	end

	return deepCopy(definition)
end

function ItemConfig.GetAllItems()
	return deepCopy(definitions)
end

function ItemConfig.GetItemIds()
	local ids = {}

	for itemId in pairs(definitions) do
		table.insert(ids, itemId)
	end

	table.sort(ids)
	return ids
end

function ItemConfig.GetItemType(itemId)
	if type(itemId) ~= "string" then
		return nil
	end

	local definition = definitions[itemId]
	return definition and definition.Type or nil
end

function ItemConfig.GetEquipmentSlot(itemId)
	if type(itemId) ~= "string" then
		return nil
	end

	local definition = definitions[itemId]
	return definition and definition.Slot or nil
end

function ItemConfig.CanStack(itemId)
	if type(itemId) ~= "string" then
		return false
	end

	local definition = definitions[itemId]
	return definition ~= nil and definition.Stackable
end

function ItemConfig.GetMaxStack(itemId)
	if type(itemId) ~= "string" then
		return nil
	end

	local definition = definitions[itemId]
	return definition and definition.MaxStack or nil
end

return table.freeze(ItemConfig)

```

## 2. InventoryService

**Purpose**
- Requires loaded player data before reading or changing inventory.
- Validates item IDs, quantities, existing inventory entries, and stack limits.
- Returns inventory snapshots so callers cannot mutate saved inventory by editing a returned table.
- Fires `InventoryChanged` after successful add/remove operations.

```lua
--==================================================
-- CHRONOS EDGE | InventoryService
-- Validated mutations and protected inventory reads
--==================================================

local ServerScriptService = game:GetService("ServerScriptService")
local Services = ServerScriptService:WaitForChild("Services")

local DataService = require(Services:WaitForChild("DataService"))
local ItemConfig = require(Services:WaitForChild("ItemConfig"))

local InventoryService = {}

local InventoryChanged = Instance.new("BindableEvent")
InventoryChanged.Name = "InventoryChanged"
InventoryService.InventoryChanged = InventoryChanged

local function validPlayer(player)
	return typeof(player) == "Instance"
		and player:IsA("Player")
end

local function validQuantity(quantity)
	return type(quantity) == "number"
		and quantity == quantity
		and quantity ~= math.huge
		and quantity ~= -math.huge
		and quantity >= 1
		and quantity % 1 == 0
end

local function getLoadedData(player)
	if not validPlayer(player) then
		return nil, "Invalid player"
	end

	if player.Parent ~= game:GetService("Players") then
		return nil, "Player is no longer connected"
	end

	if not DataService.IsLoaded(player) then
		return nil, "Player data is not loaded"
	end

	local data = DataService.GetData(player)
	if type(data) ~= "table" or type(data.Inventory) ~= "table" then
		return nil, "Player inventory data is invalid"
	end

	return data
end

local function getDefinition(itemId)
	if type(itemId) ~= "string" then
		return nil
	end

	if not ItemConfig.IsValidItem(itemId) then
		return nil
	end

	return ItemConfig.GetItem(itemId)
end

local function validateExistingInventory(inventory)
	for existingId, count in pairs(inventory) do
		if type(existingId) ~= "string" then
			return false, "Inventory contains an invalid item ID"
		end

		local definition = getDefinition(existingId)
		if not definition then
			return false, "Inventory contains an unknown item: " .. existingId
		end

		if not validQuantity(count) then
			return false, "Inventory contains an invalid quantity for " .. existingId
		end

		if count > definition.MaxStack then
			return false, "Inventory count exceeds maximum stack for " .. existingId
		end

		if not definition.Stackable and count > 1 then
			return false, "Non-stackable item has an invalid count: " .. existingId
		end
	end

	return true
end

local function fireChanged(player, itemId, newCount, action)
	InventoryChanged:Fire(player, itemId, newCount, action)
end

-- Returns a copy. Editing it never edits saved player data.
function InventoryService.GetInventory(player)
	local data = getLoadedData(player)
	if not data then
		return nil
	end

	local valid = validateExistingInventory(data.Inventory)
	if not valid then
		return nil
	end

	local snapshot = {}
	for itemId, quantity in pairs(data.Inventory) do
		snapshot[itemId] = quantity
	end

	return snapshot
end

function InventoryService.GetItemCount(player, itemId)
	if type(itemId) ~= "string" or not getDefinition(itemId) then
		return 0
	end

	local data = getLoadedData(player)
	if not data then
		return 0
	end

	local valid = validateExistingInventory(data.Inventory)
	if not valid then
		return 0
	end

	return data.Inventory[itemId] or 0
end

function InventoryService.HasItem(player, itemId, quantity)
	if quantity == nil then
		quantity = 1
	end

	if not validQuantity(quantity) or not getDefinition(itemId) then
		return false
	end

	local data = getLoadedData(player)
	if not data then
		return false
	end

	local valid = validateExistingInventory(data.Inventory)
	if not valid then
		return false
	end

	return (data.Inventory[itemId] or 0) >= quantity
end

function InventoryService.AddItem(player, itemId, quantity)
	if not validPlayer(player) then
		return false, "Invalid player"
	end

	if player.Parent ~= game:GetService("Players") then
		return false, "Player is no longer connected"
	end

	if quantity == nil then
		quantity = 1
	end

	if not validQuantity(quantity) then
		return false, "Quantity must be a positive finite whole number"
	end

	local definition = getDefinition(itemId)
	if not definition then
		return false, "Unknown or invalid item ID"
	end

	local data, dataError = getLoadedData(player)
	if not data then
		return false, dataError
	end

	local valid, inventoryError = validateExistingInventory(data.Inventory)
	if not valid then
		return false, inventoryError
	end

	local current = data.Inventory[itemId] or 0

	if not definition.Stackable and current + quantity > 1 then
		return false, "Item cannot be stacked"
	end

	if current + quantity > definition.MaxStack then
		return false, "Maximum stack size exceeded"
	end

	local newCount = current + quantity
	data.Inventory[itemId] = newCount

	fireChanged(player, itemId, newCount, "Added")
	return true, newCount
end

function InventoryService.RemoveItem(player, itemId, quantity)
	if not validPlayer(player) then
		return false, "Invalid player"
	end

	if player.Parent ~= game:GetService("Players") then
		return false, "Player is no longer connected"
	end

	if quantity == nil then
		quantity = 1
	end

	if not validQuantity(quantity) then
		return false, "Quantity must be a positive finite whole number"
	end

	if not getDefinition(itemId) then
		return false, "Unknown or invalid item ID"
	end

	local data, dataError = getLoadedData(player)
	if not data then
		return false, dataError
	end

	local valid, inventoryError = validateExistingInventory(data.Inventory)
	if not valid then
		return false, inventoryError
	end

	local current = data.Inventory[itemId] or 0
	if current < quantity then
		return false, "Not enough items"
	end

	local newCount = current - quantity

	if newCount == 0 then
		data.Inventory[itemId] = nil
	else
		data.Inventory[itemId] = newCount
	end

	fireChanged(player, itemId, newCount, "Removed")
	return true, newCount
end

function InventoryService.GetItemDefinition(itemId)
	return getDefinition(itemId)
end

function InventoryService.GetInventoryDetails(player)
	local inventory = InventoryService.GetInventory(player)
	if not inventory then
		return nil
	end

	local details = {}

	for itemId, quantity in pairs(inventory) do
		local definition = getDefinition(itemId)

		if not definition then
			return nil
		end

		details[itemId] = {
			Id = itemId,
			Name = definition.Name,
			Type = definition.Type,
			Rarity = definition.Rarity,
			Quantity = quantity,
		}
	end

	return details
end

return table.freeze(InventoryService)

```

## 3. DataService

**Purpose**
- Provides default player data and normalizes saved data.
- Validates inventory, equipment, stats, ranks, and skills during data handling.
- Uses expiring session ownership to reduce the risk of multiple servers writing the same player's data.
- Serializes saves per player, retries DataStore operations, autosaves, renews session locks, and attempts final save/release on leave or shutdown.
- Uses `UpdateAsync` to check ownership before writing.

```lua
--==================================================
-- CHRONOS EDGE | DataService
-- Validated data, atomic sessions, serialized saves
--==================================================

local Players = game:GetService("Players")
local DataStoreService = game:GetService("DataStoreService")
local HttpService = game:GetService("HttpService")
local ServerScriptService = game:GetService("ServerScriptService")

local Services = ServerScriptService:WaitForChild("Services")
local ItemConfig = require(Services:WaitForChild("ItemConfig"))

local DataService = {}

local DATASTORE_NAME = "ChronosEdge_PlayerData_v1"

local AUTOSAVE_INTERVAL = 120
local LOCK_RENEW_INTERVAL = 45
local LOCK_DURATION = 180
local MAX_ATTEMPTS = 3
local SHUTDOWN_TIMEOUT = 25

local PlayerStore = DataStoreService:GetDataStore(DATASTORE_NAME)

local Sessions = {}
local Loading = {}
local Saving = {}
local Releasing = {}
local shuttingDown = false
local activeOperations = 0

local DEFAULT_DATA = {
	Level = 1,
	XP = 0,
	Gold = 0,
	ChronoShards = 0,
	Gems = 0,
	StatPoints = 0,

	Stats = {
		Strength = 1,
		Agility = 1,
		Vitality = 1,
		Intelligence = 1,
		Luck = 1,
	},

	HunterRank = "E",
	Inventory = {},
	Equipment = {},
	Skills = {},
}

local NUMERIC_FIELDS = {
	Level = {Min = 1},
	XP = {Min = 0},
	Gold = {Min = 0},
	ChronoShards = {Min = 0},
	Gems = {Min = 0},
	StatPoints = {Min = 0},
}

local STAT_FIELDS = {
	Strength = true,
	Agility = true,
	Vitality = true,
	Intelligence = true,
	Luck = true,
}

local VALID_RANKS = {
	E = true,
	D = true,
	C = true,
	B = true,
	A = true,
	S = true,
	["S+"] = true,
}

local VALID_EQUIPMENT_SLOTS = {
	Weapon = true,
	Armor = true,
	Helmet = true,
	Accessory = true,
	Boots = true,
	Gloves = true,
}

local function deepCopy(value, seen)
	if type(value) ~= "table" then
		return value
	end

	seen = seen or {}

	if seen[value] then
		error("Cannot copy cyclic player data")
	end

	seen[value] = true

	local copy = {}
	for key, child in pairs(value) do
		copy[deepCopy(key, seen)] = deepCopy(child, seen)
	end

	seen[value] = nil
	return copy
end

local function finiteNumber(value)
	return type(value) == "number"
		and value == value
		and value ~= math.huge
		and value ~= -math.huge
end

local function isPositiveInteger(value)
	return finiteNumber(value)
		and value >= 1
		and value % 1 == 0
end

local function isNonNegativeInteger(value)
	return finiteNumber(value)
		and value >= 0
		and value % 1 == 0
end

local function playerKey(player)
	return "Player_" .. tostring(player.UserId)
end

local function validPlayer(player)
	return typeof(player) == "Instance"
		and player:IsA("Player")
end

local function normalizeNumeric(value, minimum)
	if not finiteNumber(value) then
		return minimum
	end

	return math.max(minimum, math.floor(value))
end

local function normalizeData(saved)
	if saved ~= nil and type(saved) ~= "table" then
		return nil, "Saved player data is not a table"
	end

	local data = type(saved) == "table" and deepCopy(saved)
		or deepCopy(DEFAULT_DATA)

	for field, rules in pairs(NUMERIC_FIELDS) do
		data[field] = normalizeNumeric(data[field], rules.Min)
	end

	if type(data.Stats) ~= "table" then
		data.Stats = deepCopy(DEFAULT_DATA.Stats)
	end

	local normalizedStats = {}
	for statName in pairs(STAT_FIELDS) do
		normalizedStats[statName] = normalizeNumeric(
			data.Stats[statName],
			1
		)
	end
	data.Stats = normalizedStats

	if type(data.HunterRank) ~= "string"
		or not VALID_RANKS[data.HunterRank] then
		data.HunterRank = DEFAULT_DATA.HunterRank
	end

	-- Inventory: ItemId -> Quantity.
	if type(data.Inventory) ~= "table" then
		data.Inventory = {}
	else
		for itemId, quantity in pairs(data.Inventory) do
			local definition = type(itemId) == "string"
				and ItemConfig.GetItem(itemId)
				or nil

			local valid = definition ~= nil
				and isPositiveInteger(quantity)
				and quantity <= definition.MaxStack

			if valid and not definition.Stackable and quantity > 1 then
				valid = false
			end

			if not valid then
				warn("[Chronos Edge] Removing invalid inventory entry:",
					tostring(itemId))
				data.Inventory[itemId] = nil
			end
		end
	end

	-- Equipment: supported slot -> item ID.
	if type(data.Equipment) ~= "table" then
		data.Equipment = {}
	else
		for slot, itemId in pairs(data.Equipment) do
			local definition = type(itemId) == "string"
				and ItemConfig.GetItem(itemId)
				or nil

			local valid = type(slot) == "string"
				and VALID_EQUIPMENT_SLOTS[slot] == true
				and definition ~= nil
				and definition.Slot == slot

			if not valid then
				warn("[Chronos Edge] Removing invalid equipment entry:",
					tostring(slot))
				data.Equipment[slot] = nil
			end
		end
	end

	-- Skills schema: SkillId -> true or a positive integer level.
	-- This does not grant skills; it only validates saved structure.
	if type(data.Skills) ~= "table" then
		data.Skills = {}
	else
		for skillId, skillValue in pairs(data.Skills) do
			local validId = type(skillId) == "string"
				and skillId:match("^[%w_]+$") ~= nil

			local validValue = skillValue == true
				or isPositiveInteger(skillValue)

			if not validId or not validValue then
				warn("[Chronos Edge] Removing invalid skill entry:",
					tostring(skillId))
				data.Skills[skillId] = nil
			end
		end
	end

	return data
end

local function isEnvelope(record)
	return type(record) == "table"
		and record.__ChronosEnvelope == true
end

local function recordData(record)
	if record == nil then
		return deepCopy(DEFAULT_DATA)
	end

	if isEnvelope(record) then
		if type(record.Data) ~= "table" then
			return nil, "Stored data envelope has invalid Data"
		end

		return record.Data
	end

	-- Existing version-1 records were plain player-data tables.
	-- Preserve them and migrate them when acquiring the session.
	if type(record) == "table" then
		return record
	end

	return nil, "Stored player record has an unsupported format"
end

local function ownsSession(record, session)
	if not isEnvelope(record) or type(record.Session) ~= "table" then
		return false
	end

	local owner = record.Session

	return owner.Token == session.Token
		and owner.JobId == game.JobId
		and finiteNumber(owner.ExpiresAt)
		and owner.ExpiresAt > os.time()
end

local function retryDelay(attempt)
	task.wait(attempt * 2)
end

local function acquireSession(player, session)
	local key = playerKey(player)

	for attempt = 1, MAX_ATTEMPTS do
		if player.Parent ~= Players or Releasing[player] or shuttingDown then
			return false, "Player left or server is shutting down"
		end

		local success, result = pcall(function()
			return PlayerStore:UpdateAsync(key, function(current)
				local now = os.time()
				local oldData, dataError = recordData(current)

				if not oldData then
					-- Do not replace an unreadable/unsupported record.
					return nil
				end

				local currentOwner = isEnvelope(current)
					and current.Session
					or nil

				if type(currentOwner) == "table"
					and finiteNumber(currentOwner.ExpiresAt)
					and currentOwner.ExpiresAt > now
					and currentOwner.Token ~= session.Token then
					return nil
				end

				local normalized, normalizeError = normalizeData(oldData)
				if not normalized then
					return nil
				end

				return {
					__ChronosEnvelope = true,
					Data = normalized,
					Session = {
						Token = session.Token,
						JobId = game.JobId,
						ExpiresAt = now + LOCK_DURATION,
					},
				}
			end)
		end)

		if success and ownsSession(result, session) then
			local normalized, normalizeError = normalizeData(result.Data)

			if not normalized then
				return false, normalizeError
			end

			session.Data = normalized
			return true
		end

		if not success then
			warn("[Chronos Edge] Session acquisition attempt failed:",
				player.Name, tostring(result))
		end

		if success and result == nil then
			-- Could mean an active owner or an invalid record.
			-- Retry a limited number of times; never assume ownership.
		end

		if attempt < MAX_ATTEMPTS then
			retryDelay(attempt)
		end
	end

	return false, "Could not acquire session; another session may own the data"
end

local function updateOwnedRecord(player, session, transform)
	local key = playerKey(player)

	return PlayerStore:UpdateAsync(key, function(current)
		if not ownsSession(current, session) then
			return nil
		end

		local currentData = current.Data
		local nextData = transform(currentData)

		if nextData == nil then
			return nil
		end

		return {
			__ChronosEnvelope = true,
			Data = nextData,
			Session = current.Session,
		}
	end)
end

local function waitForSave(player)
	while Saving[player] do
		task.wait(0.05)
	end
end

local function waitForLoad(player)
	while Loading[player] do
		task.wait(0.05)
	end
end

function DataService.GetDefaultData()
	return deepCopy(DEFAULT_DATA)
end

function DataService.GetData(player)
	if not validPlayer(player) then
		return nil
	end

	local session = Sessions[player]
	if not session or session.Closing then
		return nil
	end

	return session.Data
end

function DataService.IsLoaded(player)
	if not validPlayer(player) then
		return false
	end

	local session = Sessions[player]
	return session ~= nil and not session.Closing
end

function DataService.LoadPlayer(player)
	if not validPlayer(player) then
		return false, "Invalid player"
	end

	if shuttingDown then
		return false, "Server is shutting down"
	end

	if Sessions[player] then
		return true, Sessions[player].Data
	end

	if Loading[player] then
		return false, "Data is already loading"
	end

	if Releasing[player] then
		return false, "Player data is being released"
	end

	Loading[player] = true
	activeOperations += 1

	local session = {
		Token = HttpService:GenerateGUID(false),
		Data = nil,
		Closing = false,
	}

	local acquired, acquireError = acquireSession(player, session)

	if not acquired then
		Loading[player] = nil
		activeOperations -= 1

		if player.Parent == Players and not shuttingDown then
			player:Kick(
				"Your data could not be secured. Please rejoin shortly."
			)
		end

		return false, acquireError
	end

	if player.Parent ~= Players or Releasing[player] or shuttingDown then
		-- Release only our own session, retaining the data.
		pcall(function()
			updateOwnedRecord(player, session, function(currentData)
				return currentData
			end)

			PlayerStore:UpdateAsync(playerKey(player), function(current)
				if not ownsSession(current, session) then
					return nil
				end

				return {
					__ChronosEnvelope = true,
					Data = current.Data,
					Session = nil,
				}
			end)
		end)

		Loading[player] = nil
		activeOperations -= 1
		return false, "Player left during loading"
	end

	Sessions[player] = session
	Loading[player] = nil
	activeOperations -= 1

	print("[Chronos Edge] Data loaded and session acquired for", player.Name)
	return true, session.Data
end

function DataService.SavePlayer(player)
	if not validPlayer(player) then
		return false, "Invalid player"
	end

	waitForSave(player)

	local session = Sessions[player]
	if not session then
		return false, "Player data is not loaded"
	end

	if session.Closing and not Releasing[player] and not shuttingDown then
		return false, "Player session is closing"
	end

	Saving[player] = true
	activeOperations += 1

	local snapshot
	local snapshotOk, snapshotError = pcall(function()
		local normalized, err = normalizeData(session.Data)
		if not normalized then
			error(err)
		end
		snapshot = deepCopy(normalized)
	end)

	if not snapshotOk then
		Saving[player] = nil
		activeOperations -= 1
		return false, "Could not validate save snapshot: " .. tostring(snapshotError)
	end

	local saved = false
	local lastError = "Session ownership lost"

	for attempt = 1, MAX_ATTEMPTS do
		local success, result = pcall(function()
			return updateOwnedRecord(player, session, function(_currentData)
				return snapshot
			end)
		end)

		if success and ownsSession(result, session) then
			saved = true
			break
		end

		if not success then
			lastError = tostring(result)
			warn("[Chronos Edge] Save attempt failed:", player.Name, lastError)
		else
			lastError = "Save rejected because session ownership was not confirmed"
		end

		if attempt < MAX_ATTEMPTS then
			retryDelay(attempt)
		end
	end

	Saving[player] = nil
	activeOperations -= 1

	if saved then
		print("[Chronos Edge] Data saved for", player.Name)
		return true
	end

	warn("[Chronos Edge] Save failed:", player.Name, lastError)
	return false, lastError
end

function DataService.ReleasePlayer(player)
	if not validPlayer(player) then
		return false, "Invalid player"
	end

	if Releasing[player] then
		return false, "Player is already releasing"
	end

	Releasing[player] = true

	-- If loading is in progress, let it notice Releasing[player].
	waitForLoad(player)

	local session = Sessions[player]
	if not session then
		Releasing[player] = nil
		return true
	end

	session.Closing = true
	waitForSave(player)

	local saved, saveError = DataService.SavePlayer(player)
	if not saved then
		-- Keep session ownership until expiry; do not claim a successful release.
		warn("[Chronos Edge] Final save failed; session retained until expiry:",
			player.Name, tostring(saveError))

		Releasing[player] = nil
		return false, saveError
	end

	local released = false
	local lastError

	for attempt = 1, MAX_ATTEMPTS do
		local success, result = pcall(function()
			return PlayerStore:UpdateAsync(playerKey(player), function(current)
				if not ownsSession(current, session) then
					return nil
				end

				return {
					__ChronosEnvelope = true,
					Data = current.Data,
					Session = nil,
				}
			end)
		end)

		if success and isEnvelope(result) and result.Session == nil then
			released = true
			break
		end

		if not success then
			lastError = tostring(result)
			warn("[Chronos Edge] Session release attempt failed:",
				player.Name, lastError)
		else
			lastError = "Session release was not confirmed"
		end

		if attempt < MAX_ATTEMPTS then
			retryDelay(attempt)
		end
	end

	if released then
		Sessions[player] = nil
		Releasing[player] = nil
		print("[Chronos Edge] Session released for", player.Name)
		return true
	end

	-- Keep local state and do not report a successful release.
	-- The persistent session expires if it cannot be released.
	Releasing[player] = nil
	warn("[Chronos Edge] Could not confirm session release:",
		player.Name, tostring(lastError))

	return false, lastError
end

local function renewSession(player, session)
	if Sessions[player] ~= session or session.Closing then
		return false, "Session is no longer active"
	end

	local success, result = pcall(function()
		return PlayerStore:UpdateAsync(playerKey(player), function(current)
			if not ownsSession(current, session) then
				return nil
			end

			return {
				__ChronosEnvelope = true,
				Data = current.Data,
				Session = {
					Token = session.Token,
					JobId = game.JobId,
					ExpiresAt = os.time() + LOCK_DURATION,
				},
			}
		end)
	end)

	if not success then
		return false, tostring(result)
	end

	if not ownsSession(result, session) then
		return false, "Session renewal not confirmed"
	end

	return true
end

Players.PlayerAdded:Connect(function(player)
	task.spawn(function()
		DataService.LoadPlayer(player)
	end)
end)

Players.PlayerRemoving:Connect(function(player)
	task.spawn(function()
		DataService.ReleasePlayer(player)
	end)
end)

for _, player in ipairs(Players:GetPlayers()) do
	task.spawn(function()
		DataService.LoadPlayer(player)
	end)
end

task.spawn(function()
	while not shuttingDown do
		task.wait(LOCK_RENEW_INTERVAL)

		if shuttingDown then
			break
		end

		for player, session in pairs(Sessions) do
			if player.Parent == Players and not session.Closing then
				task.spawn(function()
					local renewed, err = renewSession(player, session)

					if not renewed then
						warn("[Chronos Edge] Session renewal failed:",
							player.Name, tostring(err))

						-- A failed renewal is not proof that ownership is
						-- still valid, so stop allowing further writes.
						session.Closing = true

						if player.Parent == Players then
							player:Kick(
								"Your data session could not be secured. Please rejoin."
							)
						end
					end
				end)
			end
		end
	end
end)

task.spawn(function()
	while not shuttingDown do
		task.wait(AUTOSAVE_INTERVAL)

		if shuttingDown then
			break
		end

		for player in pairs(Sessions) do
			task.spawn(function()
				local success, err = DataService.SavePlayer(player)
				if not success then
					warn("[Chronos Edge] Autosave failed:",
						player.Name, tostring(err))
				end
			end)
		end
	end
end)

game:BindToClose(function()
	shuttingDown = true

	local pending = 0

	for player in pairs(Sessions) do
		pending += 1

		task.spawn(function()
			DataService.ReleasePlayer(player)
			pending -= 1
		end)
	end

	local deadline = os.clock() + SHUTDOWN_TIMEOUT

	while (pending > 0 or activeOperations > 0)
		and os.clock() < deadline do
		task.wait(0.1)
	end

	if pending > 0 or activeOperations > 0 then
		warn("[Chronos Edge] Shutdown deadline reached with operations pending:",
			pending, activeOperations)
	end
end)

return table.freeze(DataService)
```

## Quick smoke-test checklist

Use a **separate test experience** first.

- [ ] Join the game and confirm Output reports that player data loaded and a session was acquired.
- [ ] Leave and confirm Output reports a save and session release.
- [ ] Confirm there are no `require` errors and all three ModuleScripts are named and located correctly.
- [ ] After the player data has loaded, use a temporary server Script to test `AddItem`, `GetItemCount`, `HasItem`, and `RemoveItem`.
- [ ] Rejoin and confirm that saved data persists.

Example server-side inventory check (run only after the player's data has loaded):

```lua
local Players = game:GetService("Players")
local Services = game:GetService("ServerScriptService"):WaitForChild("Services")
local InventoryService = require(Services:WaitForChild("InventoryService"))
local DataService = require(Services:WaitForChild("DataService"))

local player = Players:GetPlayers()[1]
if player and DataService.IsLoaded(player) then
    local added, result = InventoryService.AddItem(player, "HealthPotion", 2)
    print("Added:", added, result)
    print("Potion count:", InventoryService.GetItemCount(player, "HealthPotion"))
end
```

## Scope

These are backend building blocks rather than a complete playable RPG. The item catalog currently contains three sample items; more definitions can be added to `ItemConfig` as the game expands.
