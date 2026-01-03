# Marvel Rivals Lua Scripting API Reference

Use this context to write Lua scripts for the Marvel Rivals cheat framework.

## Script Structure

```lua
-- Optional: Define configurable UI settings
function on_config_ui(builder)
    builder:AddBoolean("name", "Display Name", default_bool, "tooltip")
    builder:AddSlider("name", "Display Name", default_val, min, max, "tooltip")
    builder:AddHotkey("name", "Display Name", default_vk_code, "tooltip")
end

-- Optional: React to config changes
function on_config_changed(config_name)
end

-- Required: Main game loop (called every frame)
function on_game_tick(ctx, helpers, config)
    return true
end
```

## ctx (GameContext) Fields

```lua
ctx.local_player.pawn           -- int: player pawn address
ctx.local_player.hero_id        -- int: current hero ID
ctx.local_player.team_id        -- int: team ID
ctx.local_player.camera_location -- {X, Y, Z}: camera world position
ctx.local_player.camera_rotation -- {X, Y, Z}: camera rotation (pitch, yaw, roll)
ctx.local_player.camera_fov     -- float: camera FOV
ctx.local_player.buffs           -- array of BuffDebuffInfo (1-indexed)

ctx.screen_width, ctx.screen_height
ctx.screen_center_x, ctx.screen_center_y
ctx.aimbot_key_pressed          -- bool
ctx.trigger_key_pressed         -- bool
ctx.has_aimbot_target           -- bool
ctx.aimbot_target_actor_state   -- int: current aimbot target
ctx.aimbot_target_fov           -- float: FOV distance to target

ctx.actors                      -- array of ActorContext (1-indexed)
ctx.abilities[bind_index]       -- sparse array, abilities[15][1] = first ability at bind 15
```

## ActorContext Fields

```lua
actor.actor_state               -- int: unique actor identifier (use with helpers)
actor.pawn                      -- int: pawn address
actor.hero_id                   -- int
actor.team_id                   -- int
actor.is_alive                  -- bool
actor.is_same_team              -- bool
actor.is_visible                -- bool
actor.current_health, actor.max_health, actor.health_percentage
actor.world_position            -- {X, Y, Z}
actor.velocity                  -- {X, Y, Z}
actor.distance_world            -- float: raw distance
actor.distance_meters           -- float: distance in meters

-- Screen-space bones (2D after projection)
actor.bone_head, actor.bone_spine_01, actor.bone_pelvis
actor.bone_upperarm_l, actor.bone_upperarm_r
actor.bone_lowerarm_l, actor.bone_lowerarm_r
actor.bone_hand_l, actor.bone_hand_r
actor.bone_thigh_l, actor.bone_thigh_r
actor.bone_calf_l, actor.bone_calf_r
actor.bone_foot_l, actor.bone_foot_r

-- World-space bones (3D positions) - same names with _world suffix
actor.bone_head_world, actor.bone_spine_01_world, etc.

-- Buffs/Debuffs and abilities
actor.buffs                      -- array of BuffDebuffInfo (1-indexed)
actor.abilities[bind_index]      -- sparse array, same as ctx.abilities
```

## AbilityInfo Fields

```lua
ability.can_activate            -- bool: can use now
ability.can_preactivate         -- bool
ability.is_activated            -- bool: currently active
ability.is_blocked              -- bool: blocked by game state
ability.is_on_cooldown          -- bool
ability.energy_percent          -- float: 0-100 for ult charge, etc.
```

## BuffDebuffInfo Fields

```lua
buff.hash                       -- uint64: FNV-1a hash of tag name (for fast comparison)
buff.name                       -- string: tag name (e.g., "GameplayEffect.Buff.Shield")
buff.count                      -- int: stack count (always >0 for active buffs)
```

**Example: Check for specific buff using hash (fastest)**

```lua
-- Pre-compute hash outside of on_game_tick for efficiency
local SHIELD_BUFF_HASH = 0xABCDEF1234567890  -- Replace with actual hash

function on_game_tick(ctx, helpers, config)
    -- Fast hash comparison (O(1))
    for _, buff in ipairs(ctx.local_player.buffs) do
        if buff.hash == SHIELD_BUFF_HASH then
            -- You have shield buff active
        end
    end

    return true
end
```

**Example: Check for specific buff using string (slower but flexible)**

```lua
function on_game_tick(ctx, helpers, config)
    -- String pattern matching (slower)
    for _, actor in ipairs(ctx.actors) do
        if not actor.is_same_team then
            for _, buff in ipairs(actor.buffs) do
                if buff.name:find("Vulnerable") and buff.count > 0 then
                    -- Enemy is vulnerable, good time to attack
                end
            end
        end
    end

    return true
end
```

## helpers Functions

```lua
helpers.CanDoAction()                           -- bool: 50ms cooldown check
helpers.UseAbility(ability_type)                -- bool: send ability input
helpers.SendSkill(vk_code)                      -- bool: raw key input
helpers.IsLookingAt(actor_state, max_fov)       -- bool: crosshair on target
helpers.IsKeyHeld(vk_code)                      -- bool: check if key held

helpers.GetClosestEnemy()                       -- ActorContext or nil
helpers.GetClosestVisibleEnemy()                -- ActorContext or nil
helpers.GetAimbotTarget()                       -- ActorContext or nil

helpers.CountEnemiesInRange(meters)             -- int
helpers.CountVisibleEnemiesInRange(meters)      -- int
helpers.CountEnemiesInFOV(fov_pixels, max_dist) -- int (max_dist optional)
helpers.CountVisibleEnemiesInFOV(fov, max_dist) -- int
helpers.CountAlliesInFOV(fov, max_dist)         -- int
helpers.CountVisibleAlliesInFOV(fov, max_dist)  -- int
```

## Ability Type Constants

```lua
helpers.PRIMARY_ATTACK      -- 0 (left click)
helpers.SECONDARY_ATTACK    -- 1 (right click)
helpers.RELOAD              -- 2
helpers.MELEE_ATTACK        -- 3
helpers.ABILITY_1           -- 4 (Shift)
helpers.ABILITY_2           -- 5 (E)
helpers.ABILITY_3           -- 6 (Q)
helpers.ULTIMATE            -- 7
helpers.TEAMUP_ABILITY_1    -- 8
helpers.TEAMUP_ABILITY_2    -- 9
helpers.TEAMUP_ABILITY_3    -- 10
```

## Common VK Codes (for AddHotkey/IsKeyHeld)

```lua
0x01 = Left Mouse, 0x02 = Right Mouse, 0x04 = Middle Mouse
0x10 = Shift, 0x11 = Ctrl, 0x12 = Alt
0x41-0x5A = A-Z keys (0x41='A', 0x58='X', etc.)
0x70-0x7B = F1-F12
```

## Example: Auto-ability on visible enemies

```lua
function on_config_ui(builder)
    builder:AddBoolean("enabled", "Enable", true, "Toggle feature")
    builder:AddSlider("fov", "FOV", 150, 50, 500, "Detection FOV")
    builder:AddSlider("range", "Range", 15, 5, 50, "Max range in meters")
    builder:AddHotkey("hotkey", "Activation Key", 0x58, "Hold to activate")
end

function on_game_tick(ctx, helpers, config)
    if not config.enabled then return true end
    if not helpers.IsKeyHeld(config.hotkey) then return true end
    if not helpers.CanDoAction() then return true end

    local enemies = helpers.CountVisibleEnemiesInFOV(config.fov, config.range)
    if enemies > 0 then
        helpers.UseAbility(helpers.ABILITY_2)
    end
    return true
end
```

## Performance Rules

- Avoid `string.format`, `print`, and string concatenation in on_game_tick
- Minimize table allocations per frame
- Use early returns to skip unnecessary work
- Config values are accessed directly: `config.your_setting_name`
