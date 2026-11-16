# Oddshire — Autonomous TikTok Live RPG & Gacha World Design Document

**Date:** 2026-11-15 (Finalized 2026-11-16 as **Oddshire**)
**Status:** Approved (Ready for Implementation Plan)
**Project Directory:** `C:\projects\oddshire`

---

## 1. Intent and Why

### What We Are Building
**Oddshire** is a **100% hands-off, no-camera autonomous TikTok Live game** set in an *Animal Crossing*-inspired **2.5D tabletop RPG & Gacha world**. Hundreds of viewer avatars inhabit one seamless, curved-horizon 3D world simultaneously—walking around a village plaza, venturing into a non-stop frontier dungeon, roleplaying free-text actions, pulling digital Gacha capsules (`!pull`), rolling visible D20 dice, and competing for ranks and titles—while an AI Storyteller (or a **Gifter who usurped the Sky Master throne**) drives evolving plot chapters.

### Why & Who It Serves
Oddshire is designed to achieve three goals simultaneously without compromising depth or fairness:

1. **Deep Persistent RPG & Gacha Progression:** Every viewer's TikTok username (`uniqueId`) is automatically their persistent character save file (Level, XP, Class, Gold, Gear Tier, Gacha Collectibles, Titles, Stats). Viewers have real reasons to stay for hours and return on future streams.
2. **Strong Gift Monetization ("Live Sky Master" Bidding Loop):** Instead of one-off static gift alerts, the **Highest Gifter** (or qualifying Sky Master Gift tier sender) literally ascends into the sky behind a glowing Storyteller screen as the **Live Sky Master** for a set duration, able to rewrite the live plot and world mutators via chat (`!sm <instruction>` or `!dm <instruction>` alias). Rival gifters can usurp the Sky Master seat at any time by out-gifting the current ruler, sparking exciting hero-vs-villain bidding wars.
3. **Organic Channel Growth & Online Companion Portal:** Joining and following spawn/boost a character immediately in the world, in-game highlight banners celebrate massive rolls and Mythic Gacha pulls right on stream, and **Supabase** syncs world state, live Adventure Logs, collectibles, and leaderboards in real time so viewers can follow what's happening online (e.g., from a link in the TikTok bio) even between streams.

---

## 2. Goals, Non-Goals, and Anti-Goals

### Goals
- **Simultaneous Player Agency (Zero Blocking Queue to Play):** Every viewer who types a command or free-text LARP action immediately sees a reaction on their avatar (e.g., walking toward target + spinning `🎲` icon). An **8–10 second batched AI engine** resolves up to 25–30 players' actions simultaneously in one shot—no viewer waits behind a 1-by-1 spotlight queue just to act.
- **Clean Readability at Crowd Scale (150–300+ Avatars):** Background characters display **no floating text labels** by default—only distinct *Animal Crossing*-style class hats, weapons, outfits, and subtle rank auras. Full nameplates, TikTok avatar badges, and speech bubbles render only for Spotlight targets, the reigning King/Sky Master, and the most recent 3–5 active rollers.
- **Smart Dynamic Camera Director:** Smooth spring-damped 3D camera tracks the main action and executes 2.5–3.5s Spotlight pans for Natural 20s, Natural 1s, PvP duels, Sky Master Takeovers, major highlight moments, and batched Joiner Wagons.
- **Controlled AI Token Cost (~$0.03–$0.05/hr quiet, capped under heavy flood):** Crowd votes (`1`/`2`/`3`) and quick emotes cost 0 AI tokens locally; idle ticks skip the LLM entirely; free-text LARP actions are bundled into a single compact JSON prompt every 8s (stretching dynamically with instant deterministic overflow under peak floods).
- **Lightweight Laptop Execution:** Runs smoothly at 60 FPS alongside OBS Studio / TikTok Live Studio on a standard laptop.
- **Local-First Persistence + Real-Time Supabase Sync:** Local SQLite (`world.db`) powers 0ms gameplay reads/writes on stream, while background synchronization keeps Supabase updated for the public Online Companion Web Page.

### Non-Goals
- Frame-by-frame twitch/platformer movement controls (physically incompatible with TikTok Live's 3–5 second stream delay).
- Heavy photorealistic 3D rigged skeletal meshes or blocky Roblox aesthetics.
- Manual streamer camera/mic intervention during gameplay.

### Anti-Goals (Failure Conditions Even If Technically Working)
- Making viewers wait in a 1-by-1 queue before their typed action executes.
- Calling an LLM API once per incoming chat message (causing runaway token bills and rate limits).
- Rendering 100+ overlapping usernames and chat bubbles on screen simultaneously.
- Placing essential UI inside the bottom ~35% of the 9:16 screen where TikTok's native chat/gift overlay covers it.

---

## 3. Constraints & Platform Boundary

### TikTok Live Inputs (`tiktok-live-connector`)
Only the 5 standard TikTok Live events are used as inputs:
1. `member` (Viewer Joined): Spawns/activates the viewer's avatar via the Arrival Wagon/Dock (batched if many join at once).
2. `chat` (Comment): Quick commands (`1`/`2`/`3`, `!class`, `!plaza`, `!quest`, `!pull`, `!upgrade`, `!throne`, `!duel`, `!dance`), Sky Master directives (`!sm ...` / `!dm ...`), and free-text LARP actions (`!act <text>` or filtered natural RP sentences).
3. `like` (Tap Likes): Charges personal/party "Luck / Blessing" meters and triggers minor healing/buff sparks on the tapper's avatar.
4. `gift` (Gifts with `diamondCount` & `repeatCount`): Drives Rerolls/Blessings, Elite Monster/Chest spawns, Bounty placement, and **Live Sky Master Takeover / Usurping**.
5. `social` (Follow / Share): Grants a persistent "Town Citizen" starter bonus (bonus starting gold + passive XP buff + follow sparkle on arrival).

### Screen Layout Constraint (9:16 Vertical — `1080×1920`)
All vital HUD elements and primary camera framing live in the **Safe Broadcast Band (`Y: 72px–1248px`, top 65% minus native header)**:

```text
+-------------------------------------------------------+  Y: 0
|   [TikTok Native Header Safe Margin — 72px Clear]     |  <- Host badge, viewers, X
+-------------------------------------------------------+  Y: 72
| [FULL-WIDTH COMPACT HUD STACK · 14px+ Stream Legible] |
|   👑 Sky Master Bar (@SM 2:10 | Next Usurp 201💎)     |
|   📜 Chapter & 24s Macro Vote: [1] [2] [3] · 👹 Boss  |
|-------------------------------------------------------|
|          ☁️ GIANT SKY MASTER (Y: ~280 Clear) ☁️       |
|                                                       |
|      🌳 GRAND PLAZA (Safe) <---> ⚔️ FRONTIER PATH     |
|      (Blacksmith, Gacha, Throne) (Monsters, Boss)     |
|                                                       |
|   [📜 AGGREGATED BATCH BANNER & ROLLING LOG]          |
|   [💬 5-SEC ONBOARDING: "Type 1/2/3 or !act to play!"]|
|-------------------------------------------------------|  Y: 1248 (65%)
|   (Background terrain & ambient trees only — NO UI)   |
|         TikTok Native Chat / Gift Overlay Zone        |
+-------------------------------------------------------+  Y: 1920 (100%)
```

### Hardware & Stream Latency Constraint
- Target: Standard laptop running Node.js backend + GPU-batched Three.js `InstancedMesh` / PixiJS sprite-sheet rendering alongside OBS Studio at 60 FPS with 150–300 visible puppets.
- Stream delay (3–5s): **Macro-Turn Choice Clock (`24–30s`) is decoupled from Micro-RP Batches (`8s`)** so viewers always have comfortable time to read and vote despite RTMP/HLS delay.

---

## 4. Approaches Considered

| Approach | Summary | Trade-offs | Decision |
|---|---|---|---|
| **1. Three.js 2.5D Diorama + Node.js + SQLite & Supabase Sync** | Curved *Animal Crossing* 3D terrain, 3D camera swoops, 3D tumbling D20s, lightweight layered 2.5D chibi puppets, Node.js TikTok/AI/SQLite backend, and Supabase real-time sync for the web portal. | Characters are stylized 2.5D layered puppets (like *Paper Mario* / *Cult of the Lamb* meets *Animal Crossing*) rather than full rigged 3D characters, but runs 200+ avatars at 60 FPS on a laptop, plugs directly into OBS as a Browser Source, and shares UI/data models with the online web companion. | **Chosen** |
| **2. Godot 4 Engine + Node.js Companion** | 3D world in Godot connected via WebSocket to Node.js. | Visual 3D editor, but splits runtime into two separate desktop apps, requires manual 3D asset rigging, and cannot run inside an OBS Browser Source or share web viewer code. | Rejected |
| **3. Pure 2D Top-Down Canvas (Phaser/PixiJS)** | Flat 2D top-down RPG. | Lowest GPU load, but loses the signature *Animal Crossing* curved 3D horizon, physical 3D D20 rolls, and dramatic upward camera swoop to the giant Sky Master. | Rejected |

---

## 5. Detailed System Design

### 5.1 Seamless World Map & Zones
One continuous curved-horizon 3D map with zero loading screens, divided into two connected areas:

1. **The Grand Village Plaza (Safe Hub):**
   - **Arrival Dock / Wagon Stop:** Where new joiners arrive.
   - **Village Throne:** Seat of the reigning **Village King/Queen**.
   - **Gacha Capsule Shrine (`!pull`):** Plaza shrine where players crack open capsules for hats, weapons, pets, and auras.
   - **Blacksmith / Class Shrine (`!upgrade`, `!class`):** Where players switch classes or forge persistent gear tiers.
   - **Pillory & Duel Ring:** Designated circle where `!duel @user` and `!throne` challenges resolve visually, and caught thieves serve their timeout.
2. **The Whispering Frontier & Raid Arena (Non-Stop Adventure Zone):**
   - Winding dirt road leading out through the Town Gates into forests, ruins, shrines, and the **Boss Clearing**.
   - Characters who join a quest/crusade (`!quest` or auto-marching during *Monster Crusade* chapters) walk out as a living crowd fighting roaming enemies and encountering AI events.

### 5.2 Rotating Plot Chapters & Live Gifter Sky Master

#### Rotating 10-Minute Plot Chapters (Default AI Storyteller)
Every ~10 minutes (when not overridden by a Gifter Sky Master), the AI Storyteller smoothly shifts the world's active chapter across three rotating themes so every playstyle gets spotlighted:
1. **Monster Crusade:** An active threat marches toward the village or lurks in the Frontier; players group up to raid bosses, disarm traps, and loot rare gear.
2. **Village Chaos & Heists:** A shady festival or intrigue hits the Plaza—encouraging pickpocketing, tavern schemes, mystery potions, and PvP bounties.
3. **Throne & Politics:** Trials, royal decrees, elections, tax revolts, and challenges to dethrone the current Village King.

Each active plot presents:
- A short **Plot Banner** at the top of the screen.
- **3 Crowd Choices (`1`, `2`, `3`)** updated each macro-turn/encounter (where `1` and `2` are AI-authored paths, and `3` can optionally adopt a clever viewer's suggestion as a Wildcard).

#### Live Gifter Sky Master Takeover (Monetization Core — Simulation-Refined)
- **Entry Floor & 3-Minute Reign:** Requires a minimum **200 diamonds** (configurable) to claim an empty Sky Master Throne. Reign lasts **3 minutes (180s)** to match TikTok Live attention spans and double bidding-war velocity.
- **True Windowed Cumulative Usurping & 30s Contested Overtime:**
  - Every gift adds to a **Windowed Gift Ledger (`Map<username, cumulativeDiamonds>`)** scoped to the current Sky Master cycle, so gift combos and repeat streaks accumulate toward usurping.
  - A high-visibility **Sky Master Auction HUD Bar** displays current ruler, time left, current total, top contender progress, and exact diamonds needed to usurp.
  - When a challenger exceeds the reigning Sky Master's window total, a **30-second "⚡ THRONE CONTESTED!" Overtime** triggers—allowing the reigning Sky Master to drop gifts (`≥50💎`) to **defend & extend once (`+2 minutes`, `hasDefended = true`)**.
- **Visual Transformation & Persistent Whale Prestige:** The gifter's oversized chibi avatar rises into the sky at `Y: ~280` (clear of the top HUD) behind a glowing Storyteller Screen. Top donors also unlock a persistent **Plaza Hall of Fame Statue / Donor Aura & Wings** saved across streams so their spend never vanishes when the timer ends.
- **`!sm` / `!dm` Mechanical Authority (Cumulative Per-Reign Budget):**
  - `!sm` (or `!dm`) can spawn custom bosses, change plot choices, and trigger session mutators (*2x Gold Rain*, *Frog Curse*, *Floor is Lava* requiring `!jump`/`!dance` within 8s).
  - Enforces a **cumulative per-reign budget** (max `±1,000g` / `±300 XP` distributed per reign, max `±150g`/`±50xp` per recipient, **no self-targeting**). Sky Masters cannot alter persistent Levels, Gear Tiers, or `Village King`.
- **Non-DM Gift Tiers (Refined):**
  - **Micro Gifts (1–19 diamonds):** *3-Minute Party/Personal Luck Blessing* (`+3` rolls for 3 mins + mini-pet) — **does NOT hijack the camera**.
  - **Mid Gifts (20–199 diamonds):** *Queue Elite Boss / Gold Piñata* — queues alongside the active boss without erasing current raid HP, granting the gifter **50% bounty share + Beastmaster title**.
  - **Sky Master Tier (`200+` diamonds):** *Live Sky Master Takeover* (3 minutes).
- **Free-to-Play Path ("Mini-Master Scroll"):** Double-Nat-20 rollers or Boss MVPs earn an active **60-second Mini-Master Scroll** usable when no paid Gifter holds the Sky Throne.

### 5.3 Simultaneous Action Loop, Decoupled Macro Clock & Two-Tier Batch Engine

1. **Instant Local Feedback (`0ms` server receipt + `!me` Beacon):**
   - **Zero-Token Commands:** `1`, `2`, `3`, `!class`, `!plaza`, `!quest`, `!pull`, `!upgrade`, `!throne`, `!steal`, `!dance`, `!jump`, and **`!me` (pulses a bright personal Spotlight Ring around your avatar)** cost 0 LLM tokens.
   - **Action-Verb RP Filter & Cooldown:** Explicit `!act <text>` gets top priority. Un-prefixed chat enters only if it matches an **RPG action-verb/intent lexicon** (allowing crisp 2-word commands like `"stab troll"` or `"cast heal"`, while rejecting conversational noise like `"i love this stream"`), subject to a per-user cooldown (e.g., 16–24s). All user strings are sanitized against XSS & prompt injection.
2. **Decoupled Clocks & Two-Tier Same-Tick Resolution (No Backlog Cascades):**
   - **Macro-Turn Choice Clock (`24–30s`):** World vote windows (`1`/`2`/`3`), chapter transitions, and boss mechanics run on a 24–30s round clock synced comfortably with TikTok's 3–5s video delay.
   - **Micro-RP Batch (`8s` Fixed Cadence):** Caps LLM narrative generation at **12–15 spotlighted RP actions per 8s tick**. Any overflow beyond 15 actions is resolved **in the exact same 8s tick** via fast deterministic stat templates (0 extra LLM tokens, zero multi-tick backlog).
   - **Autonomous Local Ticks:** If 0 RP actions are queued, the server skips the LLM (0 tokens) **and still advances** crowd vote tallies, local boss damage, and boss counter-attacks so the world never stalls.
3. **Aggregated Batch Summary + Rolling Spotlight Carousel:**
   - Every player gets their server-rolled D20 + Class Modifier outcome simultaneously, plus an **Aggregated Batch Summary Card** in the Adventure Log (`⚔️ 18 Heroes struck! (290 dmg) · 🌟 Crits: @Maya, @You`) and a rolling label carousel so nobody's result vanishes unread in 1 frame.

### 5.4 Crowd Readability & Camera Priority State Machine

#### Readability & 5-Second Onboarding
- **5-Second Onboarding Banner:** High-contrast ticker anchored right above the 65% line (`💬 Type 1, 2, or 3 or !act <anything> to play! · !me spots you`).
- **Minimal Text Policy + Rotating Highlight Labels:** Default crowd puppets render zero text; labels appear for the King, Sky Master, `!me` pingers, and a rotating carousel of recent rollers (never overwriting all 20 labels in 1 frame).
- **GPU-Batched Rendering:** Production renderer uses Three.js `InstancedMesh` / PixiJS sprite-sheet batching instead of raw per-puppet Canvas2D `measureText` loops for cool laptop thermals alongside OBS.

#### Camera Director Priority State Machine (No Whiplash & Safe Clamping)
- **Strict 3.0-Second Minimum Cut Cooldown:** Once the camera switches targets, lower/equal priority events cannot interrupt until `3.0s` expires.
- **Priority Hierarchy:** `1. Sky Master Takeover / Contested Overtime` > `2. Nat 20 / Nat 1 / Boss Slain / Mythic Gacha` > `3. Contested Throne Duel` > `4. Batched Arrival Wagon` (1-rose gifts & failed `!throne` attempts NEVER hijack the camera).
- **65% Safe-Zone Frustum Clamp:** Camera zoom and Y-offset strictly clamp spotlight targets above `Y <= 60%` of the frame so nameplates and D20 badges never slide under TikTok's native bottom-35% chat overlay.

### 5.5 Classes, Bounded Accuracy, Gacha Capsule Shrine & Anti-Griefing

#### Bounded Accuracy & Exponential Progression
- **4 Core Classes (`!class <name>`):** `+3` roll modifier genuinely applied to matching actions (`Knight`: combat/duels, `Wizard`: magic/rituals, `Rogue`: stealth/heists, `Bard`: charm/chaos/buffs).
- **Exponential XP Curve & Proficiency Cap:** `XP_needed = 100 * Level^1.5` (no 40-second level exploits); level modifier follows 5e proficiency (`+2` to `+6` cap) against tiered Encounter DCs (`DC 11 + Chapter Tier`) so rolls stay tense.
- **Dynamic Boss HP & Contribution-Only Loot:** Boss HP scales with active population (`HP = 450 + ActivePlayers * 25`) so bosses never melt in 1 tick; kill loot goes **only to active contributors** in that chapter (no idle-bot inflation), plus **1 Free Boss Capsule** token awarded to each active contributor.
- **Plaza Blacksmith Gold Sinks (`!upgrade`):** Persistent Gear Tiers (**Wooden → Iron [250g] → Steel [1,000g] → Mithril [5,000g] → Dragonforged [25,000g]**) granting visual weapon/aura upgrades and `+1` to `+4` gear bonuses.

#### Oddshire Gacha Capsule Shrine (`!pull` — 100% Digital Launch, ToS-Compliant)
- **Cost & Drops:** Viewers spend **250 Gold** (or redeem a free **Boss Capsule** drop) at the Plaza Gacha Shrine via `!pull`:
  - ⚪ **Common (60%):** Cozy villager hats & basic tools (`+1` roll bonus)
  - 🔵 **Rare (28%):** Glowing class weapons, knight crests, mini-pets (`+2` roll bonus)
  - 🟣 **Epic (10%):** Floating wings & elemental auras (`+3` roll bonus)
  - 🌟 **Mythic (2%):** Triggers a **Priority-2 Camera Spotlight + Golden Capsule Cutscene** (`"🥚 MYTHIC PULL! @User unboxed the Golden Frog Crown!"`, `+4` roll bonus)
- **Duplicate Star-Up Fusion:** Pulling a duplicate collectible automatically increments its star rating (`★1 → ★2 → ★3`, granting an extra `+1` bonus at `★3`) so every pull advances progression.
- **TikTok ToS Guardrail:** Digital `!pull` capsules strictly consume earned in-game Gold or Boss Kill drops—never real-money gambling, cash-out value, or "NFT/crypto" terminology.
- **Sky Master Alias (`!sm` / `!dm`):** Both `!sm <instruction>` and `!dm <instruction>` are supported for controlling the Sky Master throne.

#### Anti-Griefing PvP, Pickpocketing & Contested Throne Duels
- **Plaza Sanctuary & Guarded Immunity:** The **Grand Village Plaza is a safe sanctuary**; pickpocketing (`!steal`) is restricted to the Tavern/Frontier (or *Chaos & Heists* chapters). Robbed victims gain **15-minute Guarded immunity**; caught thieves suffer a **50g fine + 90s Plaza Pillory/Jail lockout** (or an open PvP bounty if broke, stopping 0g spam).
- **Contested Throne Duels (`!throne`):** Challenging the **👑 Village King/Queen** requires a **100g ante** into the Royal Treasury and resolves as a **true Contested Roll** (`Challenger D20 + Mod + Gear/Collectible` vs. `King D20 + Mod + Gear/Collectible`). Newly crowned Kings gain a **10-minute Coronation Grace** immunity period.

### 5.6 Local-First SQLite (WAL) + Optimistic Supabase Sync

1. **Primary Stream Store — Local SQLite (`world.db` with `WAL` mode + `busy_timeout = 5000`):** 0ms local read/write latency (`players`, `collectibles`, `world_state`, `adventure_log`, `gift_ledger`, `highlight_clips`).
2. **Future-Ready `collectibles` Table Schema:** Tracks `id`, `item_id`, `rarity`, `name`, `slot` (`hat` | `weapon` | `pet` | `aura`), `roll_bonus`, `owner_username`, `stars`, and nullable `nfc_tag_uid` / `claim_code` columns so physical 3D-printed NFC blind boxes can plug into Supabase later with zero schema rewrite.
3. **Versioned Optimistic Sync (`sync_version`) — Supabase:** Background worker reads dirty rows with their `sync_version` integer and clears dirty flags only when `sync_version` still matches after the HTTP roundtrip—eliminating race-condition progress loss.
4. **Public Online Companion Web App (TikTok Bio Link):** Live stream status, active Sky Master auction state, real-time Adventure Log, Top 50 Leaderboards (RPG Power + Top Donor Hall of Fame), and `@username` Hero + Collectible Lookup.

---

## 6. Architecture & Data Flow (High-Level)

```text
TikTok Live Room ──(tiktok-live-connector)──▶ Node.js Game Server
                                                 │
   ┌─────────────────────────────────────────────┼────────────────────┐
   ▼                                             ▼                    ▼
Local SQLite (world.db)                 8s Batched AI Engine     WebSocket Broadcast
   │                                    (Gemini Flash / Mini)         │
   ▼ (Background Sync Worker)                                         ▼
Supabase Cloud DB ──(Realtime / REST)──▶ Online Bio Companion   Three.js 9:16 Client
                                         (Public Leaderboard    (OBS Browser Source /
                                          & Hero Lookup)         Local Simulator UI)
```

---

## 7. What's Left to the Builder (Builder's Calls)

- Procedural Three.js geometry, custom curved-world shader math, and 2.5D *Animal Crossing*-style chibi character construction.
- Synthesized Web Audio API sound effects (D20 rattle, *Animal Crossing*-style gibberish Storyteller blips, crit fanfare, gift chimes).
- Exact XP leveling curve, cooldown durations, and PvP duel balance constants.
- Built-in **Offline Simulator / Dev Control Panel** for testing 100+ simulated viewers, chat floods, Nat 20/1 camera pans, and Gift Sky Master Takeovers without being live.
- **TikTok Gift Streak Deduplication (*Builder's Call*):** Track `gift` events via incremental `repeatCount` deltas and `repeatEnd` commits so combo streaks are never double-counted.
- **Chat Filtering & Idle Tick Skipping (*Builder's Call*):** Prioritize `!act` over RP-filtered chat, and skip LLM calls entirely on idle ticks.
