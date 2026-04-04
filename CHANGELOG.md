# SiZ — Suck it, Zombies
## Development Changelog

## v3.1.18a
### MP CURRENCY + INTERACTABLES + MAP FIX
- **[FIX]** MP client lost all money on armory entry
  - _Root cause: _currency not replicated; server always read 0 from remote player node. Fixed: client pushes _currency to server via rpc_save_client_currency before scene change._
- **[FIX]** MP client unable to activate dustbin or wallbuy station
  - _Root cause: same _currency=0 issue caused affordability check to always fail on server. Fixed: client passes its currency with the purchase request; deduction sent back via deduct_currency RPC on PlayerBasics._
- **[FIX]** Armory return loaded wrong map
  - _Root cause: reset() hardcoded selected_map = Map002. Fixed: default_map var set at map select time; reset() restores to default_map instead._
- **[FIX]** Post-game lobby IP input field not editable
  - _Root cause: _ready() unconditionally set ip_input.editable = false on every lobby load. Fixed: field always starts blank and editable; _on_host() locks it when hosting._


## v3.1.17a
### MP SPRITE SELECT + HYPNORAY ALLY PERSISTENCE + BHG + INTERMISSION
- **[ADD]** Separate LAN mode layout for sprite select screen
  - _In MP: background swaps to LANSpriteSelectScreen.png, stats label hidden. Sprite button and confirm button positions adjustable via exported Vector2 vars on sprite_select.gd (inspector-tunable, default to solo positions)._
- **[ADD]** MP sessions start fresh -- all solo armory unlocks ignored
  - _GameManager.is_armory_item_unlocked() returns false for everything in MP except attachments/laser. Solo save file is never read or modified during MP. Laser sight is pre-unlocked for all MP players from spawn._
- **[FIX]** Post-game lobby incorrectly pre-selected previous map and showed "Change Map" button
  - _Root cause: map_confirmed and selected_map were intentionally preserved across reset() for a Play Again loop that does not exist in MP. Both are now cleared in reset(). map_select.gd sets map_confirmed = true on map pick, so the mid-session round-trip (lobby -> MapSelect -> lobby) is unaffected._
- **[ADD]** Intermission bar drains in 2s after armory visit
  - _siz_game._returned_from_armory bool set in _ready() from GameManager.returning_from_armory. BAR_DURATION overridden to 2.0 if true; otherwise normal tiered duration (7s rounds 1-3, 5s round 4+) applies._
- **[FIX]** BHG produced no charge orb after swapping away while orb was travelling or collapsing
  - _Root cause: _on_blackhole_expired() only triggered a new charge if current_weapon == Weapon.BLACKHOLE at expiry time. Swapping away prevented the restart with no recovery path. Fixed: _apply_weapon() now starts a fresh charge when switching to BHG with no valid charged orb and no active singularity._
- **[FIX]** HypnoRay allies wiped on armory visit
  - _Root cause: change_scene_to_file() destroyed all scene nodes including live allies. Fixed: ally state (enemy_type, position, kills_remaining) saved into saved_player_state before scene change via _save_hypno_allies(). On armory return, _spawn_player() respawns each ally at its saved position with remaining kill budget, then calls link_restored_hypno_ally() on the player._
  - _Additional: is_restored_ally flag set before add_child() so _ready() skips add_to_group("enemies") and the nav-snap await that was teleporting restored allies off-screen._
- **[FIX]** HypnoRay allies converted under BHG singularity influence had distorted scale
  - _Root cause: become_hypno_ally() never reset sprite or root scale after BHG pull distortion. Fixed: sprite scale/offset/rotation/position and root scale all reset to defaults at conversion. _in_singularity_pull cleared._
- **[TWEAK]** HypnoRay ally movement speed increased 1.6x
  - _Allies used bare MAX_SPEED which was insufficient to catch enemies fleeing toward the player. _ALLY_SPEED_MULT = 1.6 applied to both engage (nav-path) and flank movement._
- **[FIX]** Sec button allies-active state not restored after armory return
  - _apply_armory_selections() now calls _sec_btn.set_allies_active(true) if _hypno_allies is non-empty after reset. New link_restored_hypno_ally() method on PlayerBasics handles the same for respawned allies._
- **[FIX]** Active BHG singularities not cleared at round end
  - _Added loop in _begin_intermission() over World/BlackHoles children; calls _expire(false) on any valid BHG node._


## v3.1.16a
### ARMORY POLISH + SPEED TUNING + MP HEALTH PICKUP FIX
- **[FIX]** LAN client could never pick up health items
  - _Root cause (primary): GameManager.reset() cleared is_multiplayer_game = false, which ran after _on_connected_to_server() set it to true. Client always loaded solo player stats (player_0001, MAX_HP=80) instead of lan_default (MAX_HP=60). Server used lan_default so MAX_HP mismatched -- server saw hp=70 >= MAX_HP=60 and skipped the client every frame. Fixed: is_multiplayer_game removed from reset(); it now persists until main menu loads._
  - _Root cause (secondary): is_multiplayer_game was never set to true on the client at all. Fixed: GameManager.is_multiplayer_game = true added to _on_connected_to_server() in multiplayer_lobby.gd._
  - _Root cause (tertiary): hp not synced to server via MultiplayerSynchronizer. Fixed: hp added as property 5 (replication_mode=1, on-change) to all four BasicPlayer scene replication configs._
- **[FIX]** restore_from_state() could set hp above MAX_HP on round transitions
  - _Root cause: saved hp captured before _load_player_attributes() had clamped MAX_HP down, then restored unclamped. Fixed: hp = min(saved_hp, MAX_HP) on restore._
- **[TWEAK]** Movement speed increased to 1.5x across all players, enemies, and drone
  - _player_attributes.json: player_0001/0004 145->218, player_0002/0003 90->135, lan_default 115->173. enemies.json: basic 105->158, runner 160->240, tank 95->143, mutant_basic 120->180, mutant_runner 176->264, mutant_tank 114->228. deployed_drone.gd MOVE_SPEED 160->240._
- **[TWEAK]** Intermission bar drain tiered -- rounds 1-3: 7s, round 4+: 5s
  - _BAR_DURATION converted from const to var; set dynamically before drain activates based on GameManager.round_number._
- **[FIX]** Completed-round label removed from intermission sequence
  - _Phase 2 (ROUND X label fade in/out) stripped from _run_intermission_sequence(). Bar drains immediately after white flash._
- **[ADD]** Armory item selection highlight -- red 12px border on first-tap (pending) panel
  - _StyleBoxFlat border applied via _set_pending_highlight(). Clears on tap-away, successful purchase, subcategory change, or scene exit._
- **[ADD]** "Not enough cash" flash -- pending panel border blinks 3x, Unable.mp3 plays
  - _Flash runs in _process via _flash_panel state var. Panel stays highlighted after flash; clears only on tap-away or purchase._
- **[ADD]** Waiting.png on armory back button in MP -- button hides, waiting image pulses brightness while server waits for all peers to confirm exit
- **[ADD]** Pressed textures wired for back buttons on Armory, MapSelect, and SpriteSelect screens
  - _MapSelectGoBackPressed.png used for Armory and MapSelect. SpriteSelectGoBackPressed.png used for SpriteSelect._

## v3.1.14a
### MULTIPLAYER BUGFIX PASS
- **[FIX]** Round number never synced to client -- client always remained on round 0
  - _Root cause: _launch_round() ran server-only with no RPC. _show_round_label_rpc() existed but was never called. Fixed: _sync_round_number_rpc() added; called from _launch_round() on every round start._
- **[FIX]** Weapon holder rotation snapping to default aim on remote screens; weapons firing in two directions simultaneously
  - _Root cause: _physics_process() ran in full on all peers. Non-authority peers wrote weapon_holder.rotation from their local aim_angle (never updated, defaulted to straight up), overwriting the MultiplayerSynchronizer's synced value every frame. Same process also called try_fire() on non-authority peers, sending real projectile spawn RPCs to the server from the wrong aim direction. Fixed: is_multiplayer_authority() guard added at top of _physics_process() after _update_eyes()._
- **[FIX]** Changing ability in armory healed player to full HP
  - _Root cause: apply_armory_selections() unconditionally set hp = MAX_HP at the end on every call, including mid-game ability swaps. Fixed: saved_hp and saved_max_hp captured before JSON reload; if player was at full health going in, hp = MAX_HP (handles first init and reinforced_chassis unlock); otherwise hp = min(saved_hp, MAX_HP)._
- **[FIX]** Sprite select screen carried over previous session's selection for both host and client
  - _Root cause: selected_sprites persisted across reset() by design, but was never cleared on fresh lobby entry. Host fix: selected_sprites.clear() on _on_host_pressed(). Client fix: selected_sprites.erase(my_id) on _on_connected_to_server(), replacing the stale re-send of the old pick. Sprite select screen now always starts with _selected_index = -1 and all buttons enabled regardless of other players' picks._
- **[ADD]** Player self-outline in multiplayer -- thin blue outline appears when your sprite moves more than 200 units from screen centre
  - _player_self_outline.gdshader with enabled uniform toggled per-frame by authority only. GameManager.is_multiplayer_game gate ensures outline never appears in solo. Hook points for occluder-triggered outline (Map002 cabins) planned for a future session._
- **[FIX]** Hypnoray targets killable by other players, weapons, and items during conversion animation
  - _Targets now added to hypno_pending group on server when locked in. GameEffectsManager.enemy_hit() returns early for any hypno_pending enemy -- covers bullets, drone, smash, dash, singularity, and meat bomb in one place. Projectiles physically pass through pending enemies (collision_layer moved to 16 via enter_hypno_pending() on basic_enemy). Group and collision restored on conversion to hypno_allies._


## v3.1.0a
### HYPNORAY ABILITY + BLACKHOLE GUN POLISH
- **[ADD]** Hypnoray secondary ability -- converts up to 2 nearest enemies into temporary allies
  - _Weapon holder animates out with lerp-in/lerp-out. HypnoAnim plays on all peers via RPC. Converted allies attack other enemies for a kill limit then expire. Cooldown starts only after all allies die. Tier upgrades: hypnoray_2 (range +25%, kill budget +3), hypnoray_3 (+1 ally cap). Green outline on pre-targeted enemies._
- **[ADD]** Blackhole Gun -- dustbin-exclusive weapon; charge/travel/singularity lifecycle
  - _Charges at muzzle (CHARGE_DURATION), then fires as a travelling orb. On arrival: pulls all enemies into a singularity, kills on contact. blackhole_expired signal re-triggers charge cycle. Armory upgrades: bhg_charge (charge time -25%), bhg_travel (travel speed +50%), bhg_absorb (singularity radius +30%), bhg_pull (disables player draw force). FMJ on singularity kills._
- **[FIX]** BHG charge orb spawning at muzzle for all weapons, not just Blackhole Gun
  - _Charge spawn triggered too broadly; now gated on current_weapon == Weapon.BLACKHOLE._


## v3.0.6a
### ROUND 0 TUTORIAL + POPUP SYSTEM + LASER SIGHT
- **[ADD]** Round 0 tutorial system for new players -- solo only
  - _First launch: round 0 runs before round 1. Sequential controller popup tips walk through movement, aiming, firing, and the armory. Cash pickup spawned near player. Returning players skip directly to intermission. _has_seen_armory_popup() persists state. MP always starts at round 1._
- **[ADD]** ControllerPopUp tutorial overlay -- in-game tip panels anchored to controller
  - _Four sequential tips keyed to game events: move, aim, fire, armory. Dismissible by interaction; auto-dismissed by trigger. Pulse highlight on relevant button during each tip._
- **[ADD]** Laser sight armory attachment ($10, 1911 only) -- visual-only beam, no gameplay effect
  - _TextureRect child of Muzzle node; visibility toggled by is_armory_item_unlocked() check on weapon swap and armory confirm. Included in armory tutorial flow._
- **[ADD]** Armory popup and main menu popup overlay system
  - _ArmoryPopUp.tscn and MainMenuPopUp.tscn: full-screen dismissible overlays. Used for first-time armory introduction and main menu feature callouts._


## v3.0.3a
### HARD MODE + CREDITS + MUSIC SYSTEM OVERHAUL
- **[ADD]** Hard Mode toggle on map select screen
  - _HardModeButton with confirmation popup (HardModeWarningPopUp). When active: bullet/drone/bomb kill drop chance reduced to 25%, player contact damage +50%, enemy HP scaling unchanged. GameManager.hard_mode persists across reset()._
- **[ADD]** Credits screen -- scrolling credits accessible from main menu
  - _Credits.tscn with back button. credits_button.gd and credits.gd. Power_Restored.mp3 as menu music._
- **[ADD]** Multi-track game music playlist -- 5 tracks, shuffled, auto-advance on track end
  - _MusicManager.play_game_music() shuffles GAME_PLAYLIST and plays sequentially. Menu music switched to Power_Restored.mp3. Tracks: Master_Disorder, Mega_Hyper_Ultrastorm, Noise_Attack, Video_Dungeon_Crawl, Zombie_Chase._
- **[TWEAK]** Music volume tuned -- level 2: -8.0 dB, level 1: -14.0 dB
- **[TWEAK]** Map select no longer pre-selects the previously chosen map -- player must make an explicit choice each session
- **[ADD]** Reset progress option on main menu with confirmation popup (ResetWarningPopUp)


## v3.0a
### THE ARMORY
> Major version bump. The Armory is the second fundamental architecture change after LAN multiplayer -- persistent cross-round progression replaces the flat starting loadout.

- **[ADD]** The Armory -- full between-round upgrade screen with three top-level categories
  - _Categories: Attachments (per-weapon: Mosin, Shotgun, 1911, BHG), Abilities (Smash, Pushback, Hypnoray -- single active slot), Accoutrement (Undercarriage, Chassis, Pickups, Armament cross-weapon upgrades). Subcategory button rows. Item cards with name, description, cost, and lock state. Persistent unlocks to user://armory_unlocks.json._
- **[ADD]** Armory accessible during intermission via hold-to-open on WeaponSwapButton
  - _2-second hold triggers armory open. Hold progress bar and pulse bar shown during intermission window. 120-second global armory timer; all players must confirm exit before game resumes. MP: server coordinates peer-ready tracking via GameManager._armory_peers_ready._
- **[ADD]** apply_armory_selections() on PlayerBasics -- re-reads JSON bases then stamps all unlocked multipliers
  - _Called on armory confirm and on _ready(). Preserves live weapon slots, BHG charge state, and HP across calls. Stamping order: Accoutrement (Armament) -> per-weapon Attachments -> Accoutrement (rest) -> Abilities._
- **[ADD]** Armory pricing reworked by weapon tier -- all item descriptions rewritten for accessibility
- **[ADD]** $1 per-kill reward -- add_kill_reward() RPC credited silently on every enemy death regardless of method
- **[ADD]** touch_button.gd and touch_texture_button.gd -- shared base scripts for all interactive buttons
  - _Replaces per-button _gui_input boilerplate. Handles ScreenTouch only (no MouseButton), _is_pressed double-fire guard, and consistent pressed signal emission._
- **[TWEAK]** MeatBomb reworked to HP damage -- base 20 HP, Mk.I armory upgrade doubles to 40 HP
  - _Previously an instant kill. Now scales with armory tier. METHOD_MEAT_BOMB routed through take_hit() HP path._
- **[TWEAK]** Drone Mk.II armory upgrade -- 2x damage multiplier; METHOD_DRONE constant added to GameEffectsManager
- **[TWEAK]** Drop system overhauled -- 45% drop chance (25% hard mode); bullet kills: 50/50 cash-or-item roll; bomb/drone kills: 50% cash, 30% health, 20% nothing
- **[ADD]** GameManager.rpc_sync_full_armory_unlocks -- host pushes complete unlock dict to all clients on armory open so all peers share the same state


## v2.5.14a
### INTERACT BUTTON REWRITE + UNABLE SOUND + CONTROLLER POLYGON FIX
- **[ADD]** WeaponSwapButton rewritten as explicit state machine -- BLANK / SWAP / INTERACT / LOCKED
  - _BLANK: no second weapon, out of range. Press: Unable sound. SWAP: two weapons, out of range. Press: swap weapons. INTERACT: near interactable, affordable. Press: purchase. LOCKED: near interactable, can't afford. Press: Unable sound. Button owns all sound decisions; server-side denial paths are silent._
- **[TWEAK]** Unable.mp3 sound ownership moved entirely to WeaponSwapButton and MeatBombButton
  - _Wallbuy and dustbin denial paths are now silent -- the button fires Unable before the press reaches the server. Eliminates the spurious Unable-on-successful-purchase bug caused by server-side timing._
- **[TWEAK]** MeatBombButton plays Unable when pressed while disabled (blank texture)
- **[TWEAK]** WeaponSwapButton _ready() calls _refresh_texture() -- prevents stale flip_h/flip_v from .tscn rendering on first frame
- **[TWEAK]** Blank texture correctly flipped (flip_h + flip_v) in BLANK state only; all other states explicitly reset flips
- **[TWEAK]** Already-owns check removed from affordability display -- button shows INTERACT vs LOCKED based purely on currency vs cost
  - _Already-owns is still a server-side silent denial; the button entering LOCKED on an already-owned wallbuy was the original source of the spurious Unable sound._
- **[TWEAK]** swap_weapon() uses closest wallbuy station rather than first in scene-tree order
- **[ADD]** MeatBombButton: touch index tracking added -- button only claims a touch that originates inside its polygon; releases only on the owning touch index
  - _Prevents thumbstick drag events bleeding into the button._
- **[FIX]** RightThumb and LeftThumb circle detection replaced with octagon polygon detection
  - _Circle radius of 300px bled into adjacent button regions. Octagon inscribed within the same bounds; Geometry2D.is_point_in_polygon used consistently across all buttons and thumbsticks._
- **[FIX]** MeatBombButton and DashButton polygons retraced to match actual teardrop button shapes
  - _Previous polygons were axis-aligned rectangles that overlapped neighboring buttons and thumbstick areas. Polygons now traced from live touch coordinates on device._
- **[TWEAK]** Intermission sequence revised -- bar drains fully over 7s then fades out, then round label crossfade plays
  - _"ROUND 1" fades in, "1" crossfades to "2" (word stays), "ROUND 2" holds then fades. _start_round fires after full sequence completes. _cleanup_labels() helper added._
- **[FIX]** _show_round_label_rpc missing @rpc decorator -- caused ERR_INVALID_PARAMETER on round transition
  - _@rpc("authority", "reliable", "call_local") restored._


## v2.4a
### INTERACTION FEEDBACK + DRONE WARNING + ITEM PICKUP REWORK + INTERMISSION UI
- **[ADD]** Unable.mp3 added to Audio/Interactables/ -- plays for all denied interaction attempts
  - _Fires on: wallbuy can't afford, wallbuy already owns weapon, dustbin can't afford, dustbin already spinning. Server-authoritative; RPC'd to the requesting peer only. Does not fire for full-HP health pickups (handled silently by skipping pickup)._
- **[TWEAK]** Health pickup skips player at full HP -- pickup persists for other players who need it
  - _Server checks player.hp >= player.MAX_HP before calling _give_health_to_player(). Loop continues to next player._
- **[ADD]** Drone warning outline -- pulsating orange outline activates at kill 7, speeds up with each subsequent kill
  - _enemy_outline.gdshader applied to drone AnimatedSprite2D. Pulse frequency: 2 Hz at kill 7, +1.5 Hz per kill thereafter. Alpha driven by sin wave each physics frame._
- **[ADD]** Drone expiry sound -- DroneTakeoff.mp3 replayed at pitch_scale 0.55 on expiration
  - _Plays locally on the authority peer only via on_drone_expired()._
- **[TWEAK]** Item button sequence reworked -- blank when empty, item texture when held, blank again while deployed
  - _set_item_type("none") called immediately on bomb activation or drone deployment. Restores to item texture if item is still in slot but not yet active (e.g. bomb count still > 0). CountLabel removed._
- **[TWEAK]** Item pickup no longer blocked by active item -- players can pick up a bomb or drone while one is deployed
  - _Pickup blocking guards removed from add_meat_bomb() and add_drone(). Players still can only hold one item and activate one at a time. Drop rates may need tuning._
- **[ADD]** Intermission UI sequence -- replaces silent 5-second gap between rounds
  - _1s post-kill delay. Completed round label fades in then out. Red drain bar fades in, fills to 100%, pops (scale punch), fades out over ~5s. Next round label fades in. All peers run sequence via _begin_intermission_rpc(). Round 1 label still fires via _show_round_label_rpc (no intermission on round 1)._


## v2.3a
### DRONE + DUSTBIN INTEGRATION
- **[ADD]** Drone item -- full implementation: item slot system, orbit/follow mode, engage/attack mode, kill-limit expiration (10 kills), solo + MP code paths
  - _MeatBombButton repurposed as generic item slot button; pop_ready() animation on pickup; DroneTakeoff.mp3 on deployment._
- **[ADD]** Dustbin: MeatBomb and Drone added to reel pool
  - _Reel textures and scales driven by items.json dustbin_texture / dustbin_scale fields. Pool order: mosin, shotgun, 1911, meat_bomb, drone._
- **[ADD]** receive_dustbin_item() added to PlayerBasics -- routes item_index to add_meat_bomb() or add_drone()
- **[ADD]** Drone world drop -- drone_pickup.tscn / drone_pickup.gd / DronePickup spawner in siz_game; drops from bullet-kill pool
- **[ADD]** Drone fire sound -- Basic_Mosin_Fire.mp3 as placeholder; plays positionally via play_world_sound_rpc on all peers
- **[TWEAK]** Cash drop rate increased -- 20% to 35% bullet-kill chance; cash weight 1.0 to 2.5 (~40% of drops)
- **[TWEAK]** Music bleed fix -- play_menu_music() / play_track() routed through MusicManager


## v2.2a
### ROUND SYSTEM + SCORING + ECONOMY
- **[ADD]** Round system -- quota formula: 8 + round*4 + floor(round^1.4); 5s intermission; round kill tracking server-side; clients notified via RPC
- **[ADD]** Runner enemy (last kill) -- from round 2, last surviving enemy converts to runner in-place; must be killed to end round
  - _convert_to_runner() applies runner stats and shader from enemies.json._
- **[ADD]** Enemy HP scaling -- +15% HP per round above 1, no cap
- **[ADD]** Dustbin (mystery box) -- server-authoritative; animated reel; $50; config from interactables.json; owned weapons excluded from pool
- **[ADD]** M1911 pistol -- starter weapon; 10-shot burst, 0.3s fire rate, 25 damage; wallbuy station on Map001
- **[ADD]** World item drop system -- 35% bullet-kill chance; weighted pool: MeatBomb, Drone, Health, Cash; round_end/Dash/Smash kills: no drops; MeatBomb kills: 5% cash-only
- **[ADD]** MeatBomb world drop -- meat_bomb_pickup.tscn; pulsing glow shader; max 3 carried
- **[ADD]** Health pickup world drop -- heals 20 HP flat; plays MiscPickup SFX
- **[ADD]** Cash pickup world drop -- grants $10; plays MoneyPickup SFX
- **[ADD]** Two-slot weapon inventory -- _weapon_slot_a / _weapon_slot_b; swap disabled until second distinct weapon acquired
- **[ADD]** Offensive item drop cooldown -- after bomb or drone drops, offensive item weight halved for next 2 drops
- **[ADD]** Score system -- GameManager tracks S/E/P/N per peer; solo leaderboard persisted to user://scores.json; LAN summary on game over
- **[FIX]** GameManager.reset() clearing selected_sprites / selected_map / map_confirmed on Play Again
  - _Fixed: session-level selections excluded from reset()._


## v2.1a
### MEATBOMB ITEM + MAP002 + MUSIC
- **[ADD]** MeatBomb item -- attracts all enemies to bomb position; explodes after fuse; kills in radius; up to 3 carried
  - _meat_bomb.tscn / meat_bomb.gd; MeatBombBeep.mp3 / MeatBombBoom.mp3 SFX; server-authoritative spawn; MP: client requests via _request_meat_bomb RPC._
- **[ADD]** Item slot system -- MeatBombButton introduced as dedicated item button; item_btn wired in PlayerBasics._ready(); pop_ready() on pickup
- **[ADD]** Map select screen -- maps.json drives map list; GameManager stores selected_map and map_confirmed
- **[ADD]** Map002 added -- multi-layer scene (ground, shed, cabin, wall, fence sprites); distinct world bounds from Map001
- **[ADD]** MusicManager autoload -- play_track() / play_menu_music() / ensure_buses(); per-map music track field in maps.json
- **[ADD]** enemies.json / items.json / interactables.json data files; GameManager loads all at startup
- **[FIX]** Shotgun triple MeatBomb drop -- die() called once per pellet hit
  - _Fixed: drop roll moved to basic_enemy.die(); is_dying guard prevents re-entry._


## v2.0a
### ATTRIBUTES, VARIANT ENEMIES + RESTRUCTURE
> First fundamental architecture change since v1.0. JSON-driven attributes, dynamic map loading, variant enemy types, and a full project restructure.

- **[ADD]** Dynamic map loading — map world no longer baked into SiZ_Game.tscn
  - _overall_test._ready() loads GameManager.selected_map at runtime and adds it to $World before any spawning occurs. NavigationRegion2D is in the tree before enemies are created. Adding a new map requires only a new .tscn and a GameManager.MAPS entry._
- **[ADD]** player_attributes.json — all player stats driven by JSON
  - _Stats: max_speed, max_hp, dash_cooldown, secondary_cooldown (displayed), reload_duration, smash_range, dash_distance, max_fire_distance, meat_bomb_max (Armory), plus flat tuning values. Four player entries plus lan_default (tier 3 across the board for all LAN players)._
- **[ADD]** Tier system for player stats — 1 to 5 stars in 20% increments of max value
  - _Each vacuum has a fixed loadout: one 1-star stat (clear weakness), two 4-star stats (clear strengths), one 2-star stat. Tier values map to actual game constants loaded at runtime. Star ratings displayed on SpriteSelect screen._
- **[ADD]** weapons.json wired into PlayerBasics at runtime
  - __load_weapon_data() called at _ready(). All weapon consts (fire_rate, burst_max, spread_degrees, aim_threshold_degrees, pellets) converted to vars and overwritten from JSON. Falls back to hardcoded defaults if JSON missing. Debug print confirms load on startup._
- **[ADD]** enemies.json — enemy stats and visual properties driven by JSON
  - _Keys: basic, runner. Fields: max_speed, damage_interval, contact_range, sound_range, shader, hue_shift, color_mod. basic has no shader. runner is faster, damages more frequently, and has a teal hue shift._
- **[ADD]** Variant enemy types — runner introduced as second enemy type
  - _enemy_spawner.gd picks type via weighted random draw (70% basic, 30% runner) and passes enemy_type through spawn data. enemy_type synced via MultiplayerSynchronizer so clients apply correct visuals._
- **[ADD]** enemy_hueshift.gdshader — canvas item shader for variant enemy skins
  - _hue_shift (0.0-1.0) and color_mod (RGB multiplier) uniforms. Applied at spawn via _apply_visuals() reading from enemies.json._
- **[ADD]** enemy_hueshift_outline.gdshader — combined hue shift + smash outline in a single pass
  - _Solves shader layering: hue shift and smash highlight previously overwrote each other since ShaderMaterial only supports one material slot. Combined shader applies hue shift to opaque pixels and outline to transparent border pixels simultaneously. Hue/color_mod params read from enemies.json per enemy type on highlight._
- **[TWEAK]** PUSHBACK_COOLDOWN and SMASH_COOLDOWN merged into single SECONDARY_COOLDOWN
  - _Both abilities share one cooldown value driven by secondary_cooldown in player_attributes.json. Simplifies tuning and display on SpriteSelect screen._
- **[TWEAK]** Smash highlight saves and restores original enemy material
  - __smash_saved_materials dictionary stores each enemy sprite's pre-highlight material. On unhighlight, original is restored. Ensures hue-shifted enemies return to their variant skin after leaving smash range._
- **[TWEAK]** Aim distance (max_fire_distance) raised from tier 3 (540) to tier 4 (720) for all players
- **[TWEAK]** Dash distance raised 30% across all players and lan_default (300 -> 390)
- **[ADD]** Full project directory restructure
  - _Scripts/, Audio/, Images/, Shaders/ at root level, each grouped by scene. Scenes/ is flat — all .tscn files at one level with no subfolders. Autoloads moved to Scripts/Autoloads/. Overall_Test renamed to SiZ_Game throughout._
- **[TWEAK]** Dead files pruned — PlayerBasics.gd.bak, SiZ_sounds.m4a, TestZombieText.png, duplicate WeaponButton images from Weapons/Images/, Level001/ folder
- **[FIX]** GDScript warnings cleaned up — unused vars and params across GameManager, GameEffectsManager, VolumeButton, MusicButton, SecondaryAbilityButton
  - __compute_score p param prefixed _p. meta var in GameEffectsManager prefixed _meta. layout_mode = 1 line removed from volume button layer builder (redundant with anchors_preset). set_smash_textures pressed param renamed pressed_tex. Dead var scale removed from LeftThumb and RightThumb._
- **[ADD]** Lobby flow rebuilt — host, sprite select, map select, and return all handled correctly
  - _lan_player_data persisted in GameManager across scene changes. _restore_as_host() and _restore_as_client() repopulate lobby on return from SpriteSelect/MapSelect. Clients request state resync from server on lobby load._
- **[ADD]** sprites_updated signal on GameManager — lobby refreshes on all peers whenever sprite dict changes
  - _Emitted inside rpc_sync_sprites (call_local). Lobby connects in _ready() and calls refresh_player_list() + _update_start_button() on every update._
- **[ADD]** Start Game gated on map confirmed AND all connected players having sprites
- **[ADD]** Pulsing gold highlight on Pick Sprite and Choose Map buttons when required action is pending
- **[ADD]** Pick Sprite button switches to Change Sprite after selection
- **[ADD]** Lobby dot and Vac label now reflect vacuum body color keyed by sprite index, not join slot
- **[ADD]** SpriteSelect buttons now display the vacuum Middle animation frame instead of raw body texture
- **[ADD]** Random player spawn positions — solo spawns anywhere valid in world bounds; LAN clients spawn within 250 units of host
  - _GameManager.get_player_spawn_position() and get_player_spawn_near() use PhysicsDirectSpaceState2D shape queries. Fixed spawn array retained as fallback only._
- **[ADD]** rpc_sync_game_over_data RPC — server pushes final_scores and lan_player_data to all clients before scene change
  - _Fixes blank leaderboard on client game over screen. Called from both trigger_game_over and _on_quit_to_menu._
- **[TWEAK]** LANGameOver leaderboard uses lan_player_data names instead of raw ENet peer IDs
- **[TWEAK]** Play Again removed from LAN game over screen pending replay vote system
- **[TWEAK]** LANGameOver panel removed — results render directly over the painted leaderboard area in the background image
- **[FIX]** Index out of bounds crash on player spawn
  - _Root cause: removing contact_time replication property left a gap in SceneReplicationConfig index sequence (0,1,2,3,5). Renumbered all four player scenes 0-4._
- **[FIX]** Instant death on first enemy touch in multiplayer
  - _Root cause: _damage_timer was a single float shared across all players per enemy. Loop fired damage for every player in the same frame. Fixed with _damage_timers: Dictionary keyed by peer ID._
- **[FIX]** Projectile physically dragging players
  - _Player collision_mask included layer 4 (projectile layer). Both CharacterBody2D nodes caused mutual physics interaction via move_and_slide. Removed layer 4 from player mask._
- **[FIX]** RPC-to-self errors on host for rpc_report_pull, rpc_report_pickup, and set_my_sprite
  - _All used rpc_id(1, ...) unconditionally. Host (peer 1) cannot RPC to itself with call_remote mode. Fixed with get_unique_id() != 1 guard._
- **[FIX]** Stale contact_time property causing constant error spam
  - _Property removed from PlayerBasics.gd but remained in replication config of all four player scenes. Removed from all scenes._
- **[FIX]** Invalid UID warnings on every project load
  - _UIDs are machine-specific. All ext_resource uid= attributes stripped from .tscn/.tres, and project.godot/export_presets.cfg UID references replaced with res:// text paths._
- **[ADD]** Player damage sounds
  - _4 damage sound files (PlayerDamaged01-04.mp3) added to Audio/Player/Damaged/. take_damage() in PlayerBasics.gd picks one at random and plays it locally via a self-freeing AudioStreamPlayer on the SFX bus._
- **[ADD]** Map002 music
  - _DangerOnTheHorizon added to Map002.tscn with same settings as Map001 (volume -15db, pitch 0.7, looping)._
- **[TWEAK]** overall_test.gd renamed to siz_game.gd
  - _All references updated across PlayerBasics.gd, basic_enemy.gd, meat_bomb.gd, meat_bomb_pickup.gd, health_pickup.gd, WeaponSwapButton.gd, WeaponDoll.gd, GameManager.gd, and SiZ_Game.tscn._
- **[ADD]** maps.json -- map config refactored out of GameManager hardcode
  - _world_min, world_max, scene path, preview image, and display name now loaded from Data/maps.json. GameManager.MAPS is now a runtime var populated by _load_maps(). world bounds applied in siz_game._ready() when the selected map loads._
- **[ADD]** Map select screen selection visibility improved
  - _Selected map thumbnail gets gold outline and gold label. Unselected maps dimmed to 60%. All driven from map_select.gd via StyleBoxFlat, no scene changes needed._
- **[ADD]** Proximity-based marker spawning in enemy_spawner.gd
  - _Enemies spawn at Marker2D nodes in the "enemy_spawner" group when players are within 2000 units. Weight scales with proximity. Normal viewport-edge spawn baseline weight 0.3 when markers are active._


## v1.3.12a
### VOLUME CONTROLS + PICKUP FLASH FIX
- **[ADD]** SFX and Music volume controls on main menu
  - _Speaker and music note icon buttons. Three-tier cycling: mute / 50% / 100%. Each button uses layered image overlays (base + level indicators) rather than swapped textures. Settings persisted to user://settings.json and restored on startup._
- **[ADD]** Dedicated SFX and Music audio buses created at runtime by MusicManager
  - _SFX bus controls all non-music audio. Music bus is independent. Master bus never touched by either control. Bus names stamped in code at _ready() on all AudioStreamPlayer nodes project-wide to fix Godot bus name resolution timing._
- **[ADD]** ButtonClickPlayer nodes added to SoloGameOver and LANGameOver — click sounds on Play Again, Quit, and Submit
- **[FIX]** Pickup sprite flash not visible during despawn window
  - _Root cause: modulate.a on the Node2D parent does not propagate through a child ShaderMaterial — the shader renders at full alpha independently of the parent modulate hierarchy. Fixed by driving a flash_alpha uniform directly on the shader material each frame. Both the sprite pixels and the glow outline now fade together._
- **[FIX]** layout_mode integer-to-enum warnings in VolumeButton and MusicButton
  - _Control.LAYOUT_MODE_ANCHORS does not exist in Godot 4.6. Redundant layout_mode assignment removed — anchors_preset already sets it implicitly._


## v1.3.11a
### SPRITE SELECT + MAP SELECT + GAME FLOW
- **[ADD]** SpriteSelect scene — 4 vacuum portrait buttons, confirm/back navigation, stats display
  - _Stats shown as asterisk star ratings (e.g. *** = 3/5) for Speed, HP, Dash, and Secondary. In LAN mode, sprites claimed by other peers are greyed out and disabled._
- **[ADD]** MapSelect scene — map thumbnail buttons with name labels, confirm/back navigation
  - _All buttons are real scene nodes — positions and sizes editable in the 2D editor. Adding a new map requires a MapButtonN node, a connection, and a GameManager.MAPS entry._
- **[ADD]** Solo flow: Main Menu -> SpriteSelect -> MapSelect -> game
- **[ADD]** LAN flow: Main Menu -> Lobby -> Pick Sprite (per-player button in lobby row) -> SpriteSelect -> Lobby -> host picks map -> Start Game
  - _Start Game button only appears after host confirms a map. Map choice broadcast to all peers via GameManager.rpc_sync_map(). Sprite selections synced via GameManager.rpc_sync_sprites(). Each peer's sprite shown in their lobby row._
- **[ADD]** BasicPlayer0002-0004.tscn created — full clones of 0001 with unique scene UIDs and node IDs
- **[ADD]** GameManager.game_scene const separated from selected_map
  - _selected_map holds the world geometry path. game_scene always points to SiZ_Game.tscn. Map select sets selected_map; confirm navigates to game_scene. Fixes map select loading bare world geometry instead of the full game scene._
- **[TWEAK]** selected_sprites, selected_map, and map_confirmed excluded from GameManager.reset()
  - _Choices survive the game -> game over -> play again loop without revisiting select screens._
- **[FIX]** Sprite selection not applying to player in-game
  - _Root cause: GameManager.reset() called in overall_test._ready() cleared selected_sprites before _do_spawn() could read it. Fixed by excluding selected_sprites from reset()._
- **[FIX]** can_process error on player spawn
  - _Root cause: player_index setter called _apply_skin() before _ready() had run, accessing null @onready vars. Fixed with is_node_ready() guard in the setter. _ready() already calls _apply_skin() after a 0.3s delay once player_index is correct._
- **[FIX]** SpriteSelecGoBack.png filename typo corrected to SpriteSelectGoBack.png
  - _Old typo'd file removed to resolve UID duplicate warning._


## v1.3.10a
### SCORING + GAME OVER SCREENS
- **[ADD]** Per-peer score tracking in GameManager
  - _Four counters per peer: S (seconds survived), E (enemies killed), P (trigger pulls), N (pickups collected). Formula: (E*5) + (N*3) + S. Trigger pulls reported via RPC in MP. build_final_scores() computes all peers at end of game._
- **[ADD]** SoloGameOver scene — score display, initials entry, top-10 leaderboard
  - _Current run score shown large. If score qualifies, initials panel appears with LineEdit (max 3 chars). Leaderboard renders as Labels positioned to align with the baked background image slots. Current run entry highlighted in gold. Scores persisted to user://scores.json._
- **[ADD]** LANGameOver scene — per-peer score summary sorted descending
  - _All peers listed with rank, player label, and score. First place highlighted in gold._
- **[ADD]** Play Again and Quit buttons on both game over screens with click sounds
- **[FIX]** _flashing unused parameter warning in AnimatedSprite._on_health_changed()
  - _health_changed signal introduced flashing param in v1.3.9a; AnimatedSprite received it but never used it. Prefixed with _ to suppress warning._
- **[FIX]** _compute_score unused p (trigger pulls) parameter warning
  - _Pulls tracked for future formula tuning but not yet factored into score. Prefixed with _p._


## v1.3.9a
### HEALTH PICKUP SYSTEM + PICKUP DESPAWN TIMERS
- **[ADD]** Map002 (Da Cabean) wired into game
  - _Map002.tscn added with Ground/Cabin/Shed/Fence sprites, collision polygons, navmesh, and 7 spawn markers. Added to GameManager.MAPS and MapSelect screen. Enemy spawning disabled temporarily; world_min/max set to placeholder -3100/3100 pending bounds calibration. Player spawns at 0,0 for testing._
- **[ADD]** New discrete HP health system replacing contact_time regen
  - _Player now has 100 HP (MAX_HP) with HIT_DAMAGE=10 per hit. No passive regen. Health only restored by HealthPickup._
- **[ADD]** Per-enemy damage interval system in basic_enemy.gd
  - _Each enemy tracks its own _damage_timer. On first contact: immediate hit. Timer resets to DAMAGE_INTERVAL (1.0s) and counts down while in range. Re-contact after leaving range also triggers an immediate hit._
- **[ADD]** HealthPickup item — heals max(5, ceil(missing * 0.5)) HP on contact
  - _Dropped by enemies on death at 5% rate (excludes smash/dash/bomb kills). heal_pickup() RPC on PlayerBasics. HealthPickupSpawner added to SiZ_Game.tscn._
- **[TWEAK]** Vignette reworked: persistent damage scaling + hit flash
  - _Vignette invisible at full HP. Scales 0->1 as HP drops 100->0. On each hit: immediate spike that decays at 4x/sec back to damage baseline._
- **[ADD]** Pickup despawn timers on MeatBomb and Health pickups
  - _Items despawn after 10s. Fading begins at 5s remaining: pulse between transparent and visible, ramping from 2Hz to 8Hz toward end._
- **[TWEAK]** health_changed signal updated to (hp_ratio: float, flashing: bool)
  - _AnimatedSprite._on_health_changed and overall_test._on_vignette_health_changed updated to match._


## v1.3.8a
### AUDIO PASS + CODE AUDIT + DEAD CODE CLEANUP
- **[ADD]** Sound effects wired — ButtonClick, Dash, Pushback, Reload, CooldownComplete
  - _ButtonClickPlayer nodes added to Controller.tscn and CanvasLayer in SiZ_Game.tscn (process_mode=ALWAYS). Also added to MultiplayerLobby.tscn. ReloadAudio node added to BasicPlayer0001.tscn._
- **[ADD]** Reload reworked — timed instant refill replaces continuous ammo regen
  - _2.04s RELOAD_DURATION matches ReloadAudio.mp3 length. _start_reload_mosin/_shotgun() set reloading flag, play audio, create timer. _finish_reload_*() restores full ammo instantly._
- **[TWEAK]** CooldownComplete sound plays on all ability button pop animations — SecondaryAbilityButton, DashButton, MeatBombButton
- **[TWEAK]** Dash and Pushback sounds wired in PlayerBasics — authority-only, routed through Controller node
- **[FIX]** Pause-state audio not playing on menu/inventory open
  - _Fixed by setting process_mode=PROCESS_MODE_ALWAYS on both ButtonClickPlayer nodes._
- **[FIX]** Inventory close playing double click sound
  - _Redundant _play_click() in _on_inventory_button_pressed() removed._
- **[FIX]** Scene-transition click sounds cutting short — fixed with 0.27s await before change_scene_to_file()
- **[FIX]** Shotgun could call die() on the same enemy multiple times in one physics frame
  - _Fixed with is_dying bool flag in basic_enemy.gd._
- **[TWEAK]** Dead code removed — min_spawn_distance export was unused; max_spawn_distance renamed spawn_margin
- **[NOTE]** Multiplayer code audit — architecture sound; untested paths noted (dash, MeatBomb, death in live MP)


## v1.3.7a
### AUDIO GROUNDWORK + SQUISH SOUND + RELOAD AUDIO
- **[ADD]** AudioStreamPlayer nodes added to Controller.tscn — ButtonClickPlayer, CooldownCompletePlayer, DashSoundPlayer, PushbackSoundPlayer
- **[ADD]** Squish sound on smash kill — SquishSound reparented to scene root on death so audio survives queue_free
  - __play_squish_rpc() called on all peers; only plays if local player within SOUND_RANGE (1500px) of death position._
- **[ADD]** ReloadAudio node added to BasicPlayer0001.tscn
- **[ADD]** Controller button click sounds wired on all ability buttons
- **[ADD]** Menu/pause/inventory button clicks wired through overall_test.gd _play_click() helper
- **[ADD]** main.gd and multiplayer_lobby.gd button clicks wired to local ButtonClickPlayer nodes


## v1.3.6a
### SMASH HIGHLIGHT + MEATBOMB PICKUP + DASH KILL
- **[TWEAK]** Smash highlight changed from full red tint to red outline shader
  - _enemy_outline.gdshader applied via ShaderMaterial to AnimatedBasicEnemy sprite._
- **[ADD]** MeatBomb pickup system — 5% enemy drop chance on death (excludes MeatBomb/Dash kills)
  - _Pulsing orange glow shader. Spawned via MeatBombPickupSpawner. Max 3 carried._
- **[TWEAK]** MeatBomb button disabled at 0 bombs — players start with none and must find pickups
- **[ADD]** METHOD_MEAT_BOMB and METHOD_DASH added to GameEffectsManager kill method registry
- **[ADD]** Dash kills — enemies within 60px during a dash are killed with METHOD_DASH


## v1.3.5a
### ENEMY OVERHAUL
- **[ADD]** Enemy spawn logic reworked — spawns in rectangular annulus outside visible play area
  - _Logic moved into GameManager.get_valid_spawn_position()._
- **[TWEAK]** Enemy collision polygon reshaped — aids lateral movement around players and obstacles
- **[TWEAK]** Enemy sprite texture updated
- **[TWEAK]** Enemy face direction changed from destination-based to velocity-based


## v1.3.4a
### ENEMY NAVIGATION
- **[ADD]** NavigationRegion2D added to map_001.tscn — baked navmesh for X-shaped walkable area
- **[ADD]** NavigationAgent2D added to BasicEnemy.tscn — enemies path around obstacles
- **[FIX]** NavigationRegion2D inheriting scale 3.118 from map sprite parent — fixed by keeping region at root level
- **[FIX]** Obstacle holes rendering as walkable area — fixed by winding inner polygons clockwise
- **[NOTE]** Enemy piling at chokepoints — avoidance attempted and removed due to stationary bug; accepted as-is


## v1.3.3a
### DASH + HUD POLISH
- **[ADD]** WeaponDoll HUD moved into WeaponDoll.tscn as scene nodes
- **[TWEAK]** Dash world boundary handling — per-axis velocity cancellation; lateral momentum preserved
- **[FIX]** Mosin-Nagant name typo fixed in PlayerBasics.gd


## v1.3.2a
### DASH ABILITY + CODE AUDIT
- **[ADD]** Dash ability — 500-unit slide in facing direction, 10s cooldown, collision disabled during dash
- **[TWEAK]** World boundary bounce on dash — push-back 15 units on world_min/world_max contact
- **[TWEAK]** Dead code removed: _on_smash_cooldown_finished() in PlayerBasics.gd


## v1.3.1a
### SMASH CLEANUP
- **[ADD]** SmashButton and SmashButtonPressed textures created and assigned to SecondaryAbilityButton
- **[FIX]** Old tap-to-smash still active alongside button-based smash — _input(), _try_melee_smash(), _request_melee_kill() removed


## v1.3.0a
### DIFFICULTY + ABILITY REDESIGN
> MP suffix dropped — multiplayer is now standard.

- **[ADD]** Burst counter system — Mosin: 5 shots, Shotgun: 3 shots before lockout
- **[ADD]** Weapon HUD — weapon name, ammo count, progress bar
- **[ADD]** Smash redesigned — up to 3 nearest enemies highlighted automatically, button press executes kill
- **[ADD]** Cooldown fade animation on SecondaryAbilityButton and MeatBombButton
- **[ADD]** Data/weapons.json created — groundwork for JSON-driven weapon attributes
- **[FIX]** receive_push RPC rejected in simultaneous pushback — changed from "authority" to "any_peer"
- **[FIX]** ERR_UNAUTHORIZED despawn errors after MeatBomb explosion — server-only queue_free() used for spawner-managed nodes
- **[FIX]** MeatBomb button stayed disabled forever for P2 after throwing — fixed with rpc_id() to authority client


## v1.2.9a_MP
### WEAPON SYNC
- **[ADD]** current_weapon added to MultiplayerSynchronizer — remote players see correct weapon model on swap
- **[ADD]** _apply_weapon() extracted from swap_weapon() — handles texture, muzzle position, fire_rate, WeaponDoll HUD
- **[FIX]** Invalid assignment on muzzle (Nil) at spawn — setter now guarded; _apply_weapon() called at top of _ready()


## v1.2.8a_MP
### SHOTGUN MUZZLE TUNING
- **[FIX]** Shotgun pellets spawning far behind the muzzle — removed spawn_shotgun_rpc; all pellets rerouted through spawn_projectile()
- **[TWEAK]** Shotgun muzzle position tuned to Vector2(0, -355); Mosin remains at Vector2(0, -373)


## v1.2.7a_MP
### PROJECTILE COLLISION FIX
- **[FIX]** Shotgun pellets ejecting ~350px backward on fire
  - _Root cause: collision_layer=4 was inside collision_mask=5; self-collision triggered physics overlap resolution. Fixed: collision_layer changed to 8._


## v1.2.6a_MP
### WEAPON SWAP + SHOTGUN + WEAPONDOLL
- **[ADD]** Weapon swap — Mosin and Shotgun selectable via dedicated controller button
- **[ADD]** Shotgun — 3 pellets, 30 degree spread, 1.0s fire rate
- **[ADD]** WeaponDoll HUD — shows current weapon sprite; updates on swap
- **[FIX]** Swap double-firing on PC — fixed with elif branches and not _is_pressed guard


## v1.2.5a_MP
### SECONDARY ABILITY TUNING
- **[TWEAK]** Melee smash range increased +10%
- **[TWEAK]** Melee smash blocked during active pushback


## v1.2.4a_MP
### SECONDARY ABILITY LOGIC
- **[ADD]** Melee smash — tap-to-kill enemies within range
- **[ADD]** Pushback — radial impulse pushes all enemies within 500 units
- **[ADD]** Visual cooldown via start_cooldown() on SecondaryAbilityButton


## v1.2.3a_MP
### INVENTORY MENU + SECONDARY SELECTION
- **[ADD]** Inventory menu — pause-overlay panel for selecting secondary ability (Smash or Pushback)
- **[ADD]** _refresh_inventory_ui() highlights active secondary and updates SecondaryAbilityButton skin


## v1.2.2a_MP
### STABILITY PASS
- **[TWEAK]** Tab indentation enforced project-wide
- **[TWEAK]** Additional dead code removal pass


## v1.2.1a_MP
### PLAYER SKIN VARIANTS
- **[ADD]** Per-player animated sprite color variants — 4 colors, 5-frame walk cycle each
- **[ADD]** Four SpriteFrames .tres resources (Player001_Frames.tres through Player004_Frames.tres)
- **[FIX]** All 4 MP instances showing Player 3 color — fixed with is_multiplayer_authority() guard in _apply_skin()


## v1.2.0a_MP
### ARCHITECTURE REFACTOR + DISPLAY FIXES
- **[ADD]** GameManager autoload singleton introduced — centralizes spawn positions, scene paths, world bounds, JSON loading
- **[ADD]** MusicManager autoload
- **[ADD]** Android IP input field repositions when soft keyboard opens
- **[FIX]** Game not scaling on secondary device — stretch mode had been changed from viewport; reverted
- **[FIX]** Controller scene broken during scaling investigation — anchors restored


## v1.1.0a_MP
### COMBAT POLISH + ENEMY CONTACT
- **[ADD]** Enemy contact damage — grace timer before damage begins
- **[ADD]** Slow health regen after breaking enemy contact
- **[ADD]** Weapon sounds broadcast via play_weapon_sound_rpc — all peers hear each other's shots
- **[FIX]** Remote players showing wrong color at spawn — is_inside_tree() guards added to _apply_skin()


## v1.0.0a_MP
### LAN MULTIPLAYER FOUNDATION
> First fundamental change: retrofitting LAN multiplayer before additional systems to avoid painful refactors later.

- **[ADD]** LAN multiplayer via Godot 4 high-level API — host/join lobby, up to 4 players
- **[ADD]** MultiplayerSpawner for enemies, projectiles, and players
- **[ADD]** MultiplayerSynchronizer on player scenes — position and rotation synced
- **[ADD]** Android manifest permissions: INTERNET, ACCESS_NETWORK_STATE, ACCESS_WIFI_STATE
- **[FIX]** Enemies heading northwest — fixed by finding nearest player on host
- **[FIX]** Client projectiles invisible — MultiplayerSpawner added
- **[FIX]** Client enemy kills causing desync — projectile physics guarded with is_server()
- **[FIX]** APK host button silently failing — Android permissions were missing from export manifest


### v0.75a
### ENEMIES, PROJECTILES + CAMERA PIVOT
- **[ADD]** Basic enemy with pathfinding, dot-product facing detection, and spawner
- **[ADD]** Projectile system — basic_projectile.tscn with direction, speed, lifetime
- **[ADD]** Weapon firing with muzzle position node
- **[TWEAK]** Swapped to Camera2D — velocity-driven map physics removed
  - _Incompatible with enemy/projectile world-space positioning._


### v0.5a
### CONTROLS + PLAYER + MAP PHYSICS
- **[ADD]** Player scene with movement, rotation, and velocity-driven map physics
- **[ADD]** Touch/virtual controls for Android input


### v0.0a
### CONCEPTUALIZATION
- **[ADD]** Core concept — sentient AI-powered vacuum, top-down zombie horde
  - _Influences: CoD Zombies, Zombie Estate 2, Escape from Tarkov_
- **[ADD]** Godot 4 project scaffolded — GDScript, Android target
- **[ADD]** Sprite-based map with StaticBody2D collision
- **[ADD]** Main menu with basic navigation

---
_SiZ (Suck it, Zombies) — Godot 4.6 / GDScript / Android LAN Multiplayer_

---
_Changelog updated 2026-04-04 (v3.1.18a)_
