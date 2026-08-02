# Bagshui Memory Leak Analysis and Fixes

## Background

Performance profiling via the `perf_monitor` addon revealed that Bagshui's `OnUpdate` handler was
allocating between 548 KB and 1,066 KB per notable call (worst case 3,595 KB), averaging ~15–22 MB
of allocations per 30-second window. This triggered Lua's stop-the-world garbage collector with
pauses as severe as **447 ms** and **364 ms**, causing visible in-game freezes that correlate 1:1
with bag update events (`BAG_UPDATE` fired 44 times per 30-second window in testing).

WoW 1.12 uses **Lua 5.0**, whose garbage collector is a simple mark-and-sweep that pauses execution
entirely. The GC is triggered proportionally to the amount of heap allocated since the last
collection; high string churn from `..` concatenation is the primary driver because:

1. Every `a .. b` expression allocates a new string object for the result.
2. A chain like `a .. b .. c .. d .. e` (right-associative in Lua 5.0) creates 4 intermediate
   string objects, only the final one survives past the expression.
3. Strings are interned (same content → same pointer), but **intermediate temporaries are still
   allocated before being looked up in the intern table** — they still hit the GC.

The fixes below replace hot-path `..` chains with `string.format` calls (single allocation) and
a `table.concat` pattern (zero intermediate strings), reducing estimated temporary string
allocations by 60–70% per cache update cycle.

---

## Fix 1 — `ItemInfo:GetTooltip()` tooltip assembly

**File:** `Components/ItemInfo.lua`
**Function:** `ItemInfo:GetTooltip()` (inner loop, ~line 397)

### Problem

The tooltip string was built by repeated `..` concatenation inside a nested loop
(per tooltip line × per left/right side):

```lua
item.tooltip = item.tooltip .. BS_INVENTORY_TOOLTIP_JOIN_CHARACTERS[lr] .. ttText
```

For an item with 8 tooltip lines and 2 sides per line, this produces **32 intermediate string
objects** per `GetTooltip()` call. Each bag slot update triggers one call per changed item.
At 80 slots with many changed items, this is a major allocation source.

### Fix

A module-level table `_getTooltip_parts` is cleared at the start of each `GetTooltip()` call.
The loop appends segments to this table, and `table.concat` assembles the final string once:

```lua
-- Module level (declared once, reused every call)
local _getTooltip_parts = {}

-- Inside GetTooltip(), before the loop:
BsUtil.TableClear(_getTooltip_parts)

-- Inside the loop, replacing the .. chain:
table.insert(_getTooltip_parts, BS_INVENTORY_TOOLTIP_JOIN_CHARACTERS[lr])
table.insert(_getTooltip_parts, ttText)

-- After the loop, replacing item.tooltip = BsUtil.Trim(item.tooltip):
item.tooltip = BsUtil.Trim(table.concat(_getTooltip_parts))
```

`table.concat` in Lua 5.0 allocates exactly one string for the final result. The reused table
itself is never garbage-collected between calls.

---

## Fix 2 — `ItemInfo:GetUniqueItemId()` unique ID string construction

**File:** `Components/ItemInfo.lua`
**Function:** `ItemInfo:GetUniqueItemId()` (~line 741)

### Problem

The unique item ID was assembled with 4 chained `..` operators:

```lua
return
    (bagNumOverride or item.bagNum or "?") .. ":" ..
    (slotNumOverride or item.slotNum or "?") .. ":" ..
    (item.itemString or "?") .. ":" ..
    (item.count or 0) .. ":" ..
    (item.bagshuiInventoryType or "?")
```

This creates **4 intermediate strings per call**. `GetUniqueItemId()` is called up to 4 times per
bag slot per cache update (once for the current ID, once for shadow tracking, and potentially
twice for the pre/post stock state comparison). At 80 slots: **80 × 4 × 4 = 1,280 intermediate
strings per update cycle.**

### Fix

Replace the chain with `string.format` using a module-level format string constant:

```lua
local _getUniqueItemId_fmt = "%s:%s:%s:%s:%s"  -- module level

function ItemInfo:GetUniqueItemId(item, bagNumOverride, slotNumOverride)
    return string.format(_getUniqueItemId_fmt,
        tostring(bagNumOverride or item.bagNum or "?"),
        tostring(slotNumOverride or item.slotNum or "?"),
        tostring(item.itemString or "?"),
        tostring(item.count or 0),
        tostring(item.bagshuiInventoryType or "?")
    )
end
```

`string.format` allocates exactly one string. `tostring()` is needed because `string.format` with
`%s` in Lua 5.0 does not automatically coerce numbers.

---

## Fix 3 — `ItemInfo:Get()` item link construction

**File:** `Components/ItemInfo.lua`
**Function:** `ItemInfo:Get()` (~line 245)

### Problem

Item links were constructed with 5 chained `..` operators:

```lua
item.itemLink = itemQualityColor .. "|H" .. itemString .. "|H[" .. (itemName or L.Unknown) .. "]|H|r"
```

This creates **5 intermediate strings per changed item**. Every item whose `itemString` is newly
retrieved (i.e., a cache miss or forced update) triggers this path.

### Fix

Replace with `string.format` and a module-level format string:

```lua
local _itemLink_fmt = "%s|H%s|H[%s]|H|r"  -- module level

item.itemLink = string.format(_itemLink_fmt, itemQualityColor, itemString, (itemName or L.Unknown))
```

---

## Fix 4 — `Inventory:UpdateLayoutLookupTables()` group ID construction

**File:** `Components/Inventory.Layout.lua`
**Function:** `Inventory:UpdateLayoutLookupTables()` (~line 284)

### Problem

The group ID string was reconstructed with `..` on every call to `UpdateLayoutLookupTables()`,
even though the exact same value is stored on `groupDetails.groupId` from the previous call:

```lua
local groupId = rowNum .. ":" .. groupNum
-- ...
groupDetails.groupId = groupId  -- stored here for next call
```

Layout updates happen on every bag update that causes a layout recalculation.

### Fix

Reuse the cached value when available:

```lua
local groupId = groupDetails.groupId or (rowNum .. ":" .. groupNum)
```

The string is still created on first layout build (when `groupDetails.groupId` is nil), but
subsequent calls return the already-interned string with no new allocation.

---

## Fix 5 — `Inventory:UpdateCache()` pre/post item count table copy

**File:** `Components/Inventory.Cache.lua`
**Function:** `Inventory:UpdateCache()` (~line 551)

### Problem

At the end of every cache update cycle, the post-update item counts were copied into the
pre-update table via `BsUtil.TableCopy`:

```lua
BsUtil.TableCopy(self.postUpdateItemCounts, self.preUpdateItemCounts)
BsUtil.TableClear(self.postUpdateItemCounts)
```

`TableCopy` iterates every key/value pair and writes them into the destination table. For
inventories with many item types this is a non-trivial traversal, and `TableCopy` creates a new
destination table when `dest` is nil — though here `dest` is always provided, so no new table
is allocated. However, the copy itself is unnecessary work.

### Fix

Swap the table pointers and wipe the now-redundant one:

```lua
local _temp = self.preUpdateItemCounts
self.preUpdateItemCounts = self.postUpdateItemCounts
self.postUpdateItemCounts = _temp
BsUtil.TableClear(self.postUpdateItemCounts)
```

The post-update table becomes the pre-update table for the next cycle at zero copy cost. The
old pre-update table is cleared and becomes the new post-update accumulator. No allocations.

---

## Summary of Changes

| # | File | Change | Estimated impact |
|---|------|--------|-----------------|
| 1 | `Components/ItemInfo.lua` | `GetTooltip()`: `..` loop → `table.insert` + `table.concat` | ~32 strings → 1 per item per tooltip load |
| 2 | `Components/ItemInfo.lua` | `GetUniqueItemId()`: `..` chain → `string.format` | ~4 strings → 1; ×1,280 calls/update |
| 3 | `Components/ItemInfo.lua` | `Get()` itemLink: `..` chain → `string.format` | ~5 strings → 1 per changed item |
| 4 | `Components/Inventory.Layout.lua` | `UpdateLayoutLookupTables()`: reuse cached `groupId` | ~N strings → 0 on repeat calls |
| 5 | `Components/Inventory.Cache.lua` | `UpdateCache()`: `TableCopy` → pointer swap | O(n) copy → O(1) swap |

Combined effect: estimated **60–70% reduction** in temporary string allocations per bag update
cycle, directly reducing GC trigger frequency and severity of GC-induced frame pauses.
