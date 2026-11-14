# DiegeticDread - architecture and method

DiegeticDread is a dark ambient layer for S.T.A.L.K.E.R. Anomaly.
It gathers dark and eerie sounds from community soundscape packs, then measures and deduplicates them. It plays them as positioned single sounds.
A runtime director reads where the player stands and who is near.
It does not add engine ambient channels and it does not edit an ambient file. It plays its own sounds, and it mutes the base game's copy of any sound it also ships.

This document is the method and the invariants. A private build tool masters the corpus, and this repo holds only its output: the record (`dd_sound_metadata.script`) and the ogg tree.
No source build is named here. `dd_sound_metadata.script` is the corpus record AND the runtime table, one file. The tool reproduces every number below.

## Model: a single-sound director, no channels

The old model shipped our sounds as engine sound channels (`sound_channels.ltx` sections) and let the ambient system play them. That is gone. DiegeticDread now owns playback end to end:

- Content lives FLAT in our own category directories under `gamedata/sounds/zs/<category>/<name>.ogg`, never in an engine channel. Dread is not a directory.
  It lives per-sound in `dd_spooks_metadata` (the business-metadata override, below). The deploy writes no `sound_channels.ltx` definitions for our content.
- The director (`gamedata/scripts/dd_director.script`) plays each sound as a positioned single play through `xsound.play_at`, the vanilla `play_at_pos` call shape with a RETAINED handle
  that keeps every playing sound stoppable (`xsound.stop_shots`, the dev-tab Stop) and alive across GC. A long horror drone or a psy bed plays as one sound on a long period, with no loop and no bed.
- Placement follows vanilla `sound_ambient.script`. `_compute_spawn_dist()` rolls `random(spawn_min, ch_max) / 2`, with `spawn_min` the row's `ch_min` underground and the band's midpoint above ground.
  `play_sound()` pulls that distance closer by dread (`_compute_pull_dist()`), and `get_placement()` adds a random bearing and the row height. The volume is the sliders alone.
  Only the base-game channels the director replays (`play_base_channel()`) take the vanilla `indoor` volume table.
- The base's own copy of a sound we ship is removed from its ambient channels statically by a DLTX overlay at config load (the exclusion, below), so the base never doubles the director.
  A separate observer owns the vanilla `update_ambient` slot only to replay and log the base ambient.

Because the director is the only playback path, xlibs (`xsound`) is required. Without it the mod is inert and no sound plays. The director's own data is Lua: `dd_sound_metadata`
(generated audio facts), `dd_spooks_metadata` (hand-curated dread), and `dd_location_override` (hand-curated level and coordinate overrides), each an auto-loaded table read once.
`xltx` reads only the engine's own LTX (the base-ambient `sound_channels.ltx`), and `xfs` enumerates on-disk oggs for the review player, which also reads a logs-dir `.txt` playlist through `getFS`.
There is no raw `ini_file` in the mod's own scripts.

The category is the unit of organization and of play. It is a directory of sounds plus two attributes, an `env` set (which enclosure states it may play in) and a `requires` gate (a live precondition),
with no weight and no cooldown. Dread lives per-sound. `dd_spooks_metadata` (the `sounds` table, `["deployed-name"] = { dread = "low"|"med"|"high" }`) is the single source of a sound's dread.
A sound with no override plays at EVERY scene dread (the default), and an overridden one plays ONLY at its dread. Curating a sound is adding one line to that table, keyed by the stable deployed name,
and it survives every intake and refresh (dread lives in the hand-curated table, never in the tree). The director reads the generated sound config (`dd_sound_metadata.script`) that lists each category,
then that category's flat sound list, each row a named-field table: its blob min/max pair, its spawn band (ch_min/ch_max, derived from the reach), its height,
and its measured loudness (`lufs`/`crest`/`bv`). The director also flattens those rows into `xsound.load_meta`, so any consumer can read a sound's delivered loudness.
It applies the dread override at load to build the per-dread pools. The config replaces the channel definitions the director used to read from `sound_channels.ltx`.
Anomaly Lua cannot enumerate a directory at runtime, so the deploy writes the config and the director reads it. See "Categories - the rule table" below.
The category list is the single source of truth.

## Content pipeline (reproducible)

The build tool keeps DiegeticDread as two states: the ogg tree and ONE committed record.
That record, `dd_sound_metadata.script`, is both the tool's store and the file the director loads.
Every command takes a source tree and a target tree and names no mod. Copying a finished target over this repo is a manual step.

The automatic commands:

- intake reads a sound pack, and it is the only command that does. It folds stereo to mono and resamples any off-rate file to 44100.
  It then deduplicates inside the pack and against the tree by audio alone.
  Every net-new sound goes to the `holding` category as `zs/holding/<origname>_<audiohash>.ogg`, mastered at the default reach for the review player.
  Intake decides nothing about where a sound belongs. It plans in its own scratch and commits at the end. A failed intake writes nothing, and a second intake of the same pack changes nothing.
- refresh never reads a pack. It files every ogg under its row's category, measures what is new, and recomputes every blob under the measured model (below).
  It regenerates `dd_sound_metadata.script`, then runs verify and the delivered-loudness audit. A second refresh writes nothing.

The manual steps:

- sort: a holding sound moves to its category when its row moves into that category's block of the record. The next refresh moves the ogg to match.
  The evidence is the analysis report, the sound's original folder and file name, and this mod's own tables (the CATEGORY rules, the level lists).
  The report gives each sound its top tags, an inside score, a quality check and a suggested category. Verify FAILS while any holding row remains.
- reject: delete one sound, its ogg and its row. The tool keeps no reject memory. A later intake can bring the audio back, and the curator judges it again.
- cut: slice a long bed into named pieces. This is authoring, never part of intake.
- the base-exclusion overlay, written by hand (below).

Dread curation lives per sound in `dd_spooks_metadata`, outside the tree, where no command touches it.

### No audible sound drops before audition

Measurement only flags candidates. A dead report lists rows at or under -60 LUFS integrated, and the audit lists the faint ones.
An analysis report adds clipping, noise, speech or music contamination, and scene contradictions.
Each list loads into `ui_dd_player` as a playlist. A file leaves the corpus only by a reject.
Intake reports a file whose fold or resample fails and never ships it. That is the one mechanical removal.

### Sorting is manual

The curator picks each sound's category by hand after intake. The tool holds no routing table, folder rule or pack registry.
To adopt a pack, the curator points intake at its folder. Every net-new sound then waits in `holding` until the curator sorts or rejects it.

### Identity and the corpus of record

Each file is named by content: `zs/<category>/<origname>_<audiohash>.ogg`, the hash an md5 of the audio pages only (blob-agnostic), short form. The name is readable and keeps the source name.
The hash disambiguates the generic `sound_NN` names 27 packs share, and a file's name comes from its own content. Adding content never renames anything.
The record row key is `<category>/<name>`, category-qualified because one pack ships identical audio in two folders routed to two categories (the labs/drone twins).

The corpus of record is `dd_sound_metadata.script` ITSELF, the same file the director loads. Its rows follow the uniform sound schema every Diegetic sound shares.
The runtime columns are path, ch_min/ch_max, mn/mx, h and lufs/crest/peak/bv.
Beside them sit the tool's columns the game never reads: the source's own blob, fold and slice lineage, duration, fingerprint, seq and audio hash.
Each row is one sound, keyed by its audio. The record keeps no source pack and no source path. The source blob is an audit column, and the measured master never reads it.
Every refresh derives the band from the category reach. ch_max is the reach and ch_min is 0.6 times it, one pair per category. `h` is the category's height.
A `seq` ordinal keeps every emit byte-stable. Row batches are wrapped in their own Lua functions to stay under LuaJIT's constants-per-prototype cap.
Refresh recomputes everything from the record. That is why a pack is deletable after intake.

- Intake is additive. A new pack dedups against the record by audio alone, and only net-new audio enters, appended with the next `seq`.
  The match is the exact audio hash, then chromaprint candidates decided by PCM cross-correlation at >= 0.90.
- Limits: re-encode detection is heuristic (fp >= 0.88 candidate, xcorr >= 0.90 decide), not exact. A better copy of a shipped sound arriving later never replaces it implicitly.
  Upgrading is a deliberate reject followed by a new intake.

## Deduplication: waveform identity, source side only

Identity is decided by the waveform. There are four stages, cheapest first, so the expensive test runs only on the pairs the cheap ones flag (`dedupe`, `pcm_correlation`).

- md5: byte-identical reships across packs collapse to one.
- audio hash: among the md5 survivors, files identical in AUDIO but differing only in their comment blob collapse here, exact and blob-agnostic.
  A reship with a different volume or distance is the usual case, and the fuzzy stage could miss it.
- Chromaprint fingerprint (`fpcalc`, >= 0.88): stable across bitrate and codec, so it finds the re-encoded copies md5 misses. Its same-versus-distinct ranges overlap,
  so it only proposes candidate pairs and never decides.
- PCM cross-correlation (`DEDUP_XCORR` = 0.90): decode both, line them up by envelope offset, and correlate over the overlap. A re-encode scores near 1.0 and a distinct sound near 0.
  Two files merge only under complete linkage, so a similarity chain never collapses two distinct recordings and variety is never lost.

The survivor of every collapsed group is the HIGHEST-BITRATE copy (`dedupe`: `max(group, key=bitrate)` at the md5, audio-hash, and cross-correlation stages),
and inside one intake a file under 32 kbps is dropped when the pack holds anything at or above it.
A low-bitrate survivor is therefore the best, or the only, copy of that sound across every source we pull. The low-bitrate tail is old SoC-era source recordings that were never released higher,
kept because re-encoding cannot restore detail the source never captured. Modern packs contribute none.

This runs among the source packs only, within a pack and between the packs we pull from. DiegeticDread does not deduplicate against the target modpack.
It never drops a sound because the install already plays it. Doubling with the base is handled by the static DLTX exclusion overlay at config load.

## Byte-for-byte audio, blob rewritten to the measured model

A MONO sound's AUDIO ships byte for byte with no re-encode, the vorbis pages untouched. What refresh writes is only the X-Ray ogg comment blob (min distance, max distance, base_volume),
a lossless header-page rewrite that leaves the audio pages identical while it recomputes the blob from the measured model (below).
A STEREO sound is one exception. Because the engine cannot 3D-position it, intake folds it to mono (re-encoded) before hashing.
An off-rate sound is the other. Intake resamples it, because the engine rejects any rate but 44100.
The source's own blob is captured before the fold and kept as an audit column. The measured master does not read it.

Blob contract (the engine-read fields). The `0x0003` comment struct (`SoundRender_Source_loader.cpp`) holds five fields: `min`, `max`, `base_volume`, `game_type`, and `max_ai_dist`.
The rewrite sets all five, and the engine reads exactly those. Nothing written is ignored, nothing read is left unset.
`min`, `max`, and `base_volume` carry the attenuation and loudness below, the linear rolloff plus the OpenAL inverse keyed on `min` and the `base_volume` gain (`SoundRender_Emitter_FSM.cpp`).
`game_type` keeps the file's own value, and `max_ai_dist` is written equal to `max`, both inert for our content.
The play-time sound type overrides the blob `game_type` (`SoundRender_Core.cpp`).
`max_ai_dist` only sets NPC hearing range, which our `no_sound`-type plays never trigger (`SoundRender_Emitter.cpp`).
`max_ai_dist` is set to `max` only to satisfy the loader's `>= 0.1` assert, the one engine-read lever left deliberately unused.
A `world_ambient` play with an owner would make NPCs hear the sound, which the director does not want.

- The blob is the measured model, shared across every sound mod. No author min or max is read.
  The max is the category's reach from the tool's reach table, which follows vanilla's `sound_channels.ltx` reach per content.
  Wind and foliage reach 60. Drip, rats and bats reach 90. Labs, industrial and structural reach 115. The mutant kinds reach 150, and every other category 120.
  The min is the measured crest floor off the farthest placement. A sustained low-crest bed carries at distance, and a sharp high-crest transient stays near-field.
  The crest band was tuned on ffmpeg's linear crest factor. The record holds crest in dB, and the build converts it back for the band.
  base_volume starts at 1.0 and is bounded into the loudness band off the measured LUFS (ffmpeg ebur128).
  A file that delivers quieter than the floor (-36 dB) at its placement is lifted toward it, and one louder than the ceiling (-18 dB) is pulled toward it.
  Each move closes 0.7 of the gap. The lift is capped at the far placement, and the cap is set for the game's effects volume at 0.5.
  Above that setting the engine's 1.0 gain clamp takes part of the lift.
  The same audio and reach give the same blob every run.
- Reach is the one tunable, per content, never per sound. A category that plays too loud or too far has a wrong reach value.
  Fixing it once moves every sound in the category on the next refresh.
- Height is per category and follows vanilla's own channel heights. Birds play 10 m up like vanilla's crows, rats 1 m down, and every other category at 0, labs included.
  A category that mixes heights is split. That is why birds left wildlife.
- Dead files are reported, never culled. Refresh lists every row at or under -60 LUFS as a dead candidate, and the curator rejects it.
  Every quiet sound that carries real content stays. Its faint feel comes from the director's placement.

The config (`dd_sound_metadata.script`, also the corpus record) carries per sound the blob min/max pair (read back from the deployed ogg, trace readout only), the spawn band the director places within (ch_min/ch_max, derived from the reach), the height, and the measured `lufs`/`crest`/`peak`/`bv` fed to `xsound.load_meta` for the delivered-loudness readout.

Fitness: 44100 Hz vorbis only, the X-Ray standard. Intake resamples an off-rate file, and it drops a file under 32 kbps when the pack holds a better copy. The 44100 rule is the engine's own.
`CSoundRender_Source::LoadWave` hard-rejects any other rate (`SoundRender_Source_loader.cpp`, returns false). `xrSound` has NO resampler and decodes to PCM, then plays the file as decoded.
No runtime step downsamples or re-compresses, so shipped audio is exactly its source quality.
A low-bitrate survivor is an old low-bitrate source kept because it is the best copy available (see Deduplication).

## The director

The director owns playback on ONE 100ms loop (`("dd_director","tick")`, separate from the base-ambient observer). Each cycle round-robins ONE producer that writes its sensor into a flat board,
derives dread and the eligible set from the board, and emits at most one positioned sound. The per-cycle cost is the single heaviest scan, and cost never sums across producers,
so the whole board refreshes over the producer count (~0.6s).

    every 100ms -> run one producer (a scan), writing its sensor into the board
    sense       -> the board: environment, time, stalkers{}, monsters{}, anomalies{}
    select      -> eligible = map (dd_location_override.levels) & environment & presence -> two-level shuffle-bag
    apply       -> dread (grounded, additive) -> emission frequency + the dread bucket the sound-bag draws from
                   -> vanilla-band position + play

Smart terrains are deliberately NOT an input. They proved unreliable for filtering (the trader and base cases). Place identity comes from the level, the environment, and a live seller check.

### The board - the world as tokens

The board is the single source of truth. Producers write it, and SELECT, APPLY, and the HUD read it. Values are readable tokens. There are two sensor classes: SCAN (polled on the round-robin) and,
later, EVENT (bumped by callbacks, decayed). A sensor's fields match what is useful. Mobile things carry `online` plus `near`, and static anomalies carry only `near`.

- `environment` - "outdoor" | "indoor" | "underground" | "labs". `GetEvent("underground")` says you are on an underground level, `LAB_LEVELS` splits the sci-fi labs from the tunnel/mine/bunker levels,
  and otherwise the `is_indoor` raycast (cached by cell) gives indoor versus outdoor.
- `time` - "day" | "dusk" | "dawn" | "night" | "deep_night".
- `stalkers` - one pass over the dedicated `db.OnlineStalkers` array (`_collect_stalkers`): `online` (any stalker online, the gunfire gate),
  `enemy_near` and `ally_near` (the strongest near hostile and the strongest near non-hostile, each a power tier,
  where a neutral NPC counts as company because only a per-NPC enemy relation is a threat).
  `service_near` reads none, allied or hostile. The nearest trader, barman, medic or mechanic within 60m marks a base.
  It comes from `xsmart.find_base_owner`, which also sees Sidorovich and the Forester that the array never lists.
- `monsters` - one pass over `xcreature.iterate_online_monsters` (`_collect_monsters`): a demonized `db.OnlineMonsters` id-array when present,
  else a best-effort `is_mutant` walk capped at `MONSTER_ITER_CAP` online objects so the scan stays bounded under soak (a monster past the cap is missed),
  where an engine-side monster registry is the parity fix: `online` (the mutant gate), `enemy_near` (strongest near monster as a power).
- `anomalies` - `near` (a hazard anomaly within range, `xsmart.is_hazard_anomaly_near`), the drone gate.

### Power - man and monster on one scale

Every hostile near thing is graded "none"|"low"|"med"|"high" by `get_power_tier(rank)`. A stalker's `character_rank()` (0-27000, cuts 9000/14500) and a monster's `se:rank()` (1-20,
cuts 6/17) collapse to the SAME token. So `stalkers.enemy_near` and `monsters.enemy_near` are the same type, and threat reads both with one rule (`max`).
The cuts sit on the engine's own rank scales, a stalker's `character_rank()` and a monster's `se:rank()`.

### The loop and the emission gate

The loop runs a fixed 100ms and never stops, so a rising dread is always caught, and the board refreshes over the producer rotation. Emission is separate:
the gap between plays scales with the CURRENT dread, from a calm mean of 40s (`SPACE_CALM_MS`, silence carries the dread) to a peak mean of about 8s (`SPACE_PEAK_MS`),
power-eased off the calm end (`SPACE_EASE`). At dread 0.5 the gap is about 19s (drops fast then fine-grades toward peak), with a `SPACE_JITTER` spread.
The gap is re-evaluated every 100ms against the live dread.
A rising dread tightens it at once, and a threat that appears mid-calm-gap fires promptly even during the long calm interval.
This targets the base's COMBINED felt rate (vanilla horror channels average 45-70s each but run ~5 concurrently, ~14s combined, and the modern packs ~5s combined) with a single stream,
kept sparser at calm because horror pacing builds dread through silence and keeps strong scares rare. Nothing emits only at dread 0 (a safe hub or a fully-calmed place).
Any dread above 0 plays, and it grows sparser the calmer it is, while the loop keeps sensing and the gap keeps advancing so a resumed dread emits at once.
There are no per-category cooldowns and no weights.

### SELECT - which categories can play

A category is eligible only if all three checks pass, in order (`is_eligible`, reading the board):

- **map** - the level's list in `dd_location_override.levels` names it.
  `default` holds the universal cues on every level, and each level's list adds its terrain flavor, its interior or facility kinds, and the `dark_signal` lore placement.
  A lab level lists `labs`, a swamp lists `mutant_ambient_swamp`, and a wild forest never lists `dark_signal`.
- **environment** - the category's `env` set contains the current `board.environment` (labs counts as underground for the gate). Outdoor never plays the inside kinds (structural, labs, drip, rats),
  and indoor never plays the outside kinds (foliage, wind, wildlife, birds, urban, the zones).
- **need** - a live gate: `mutant` needs `monsters.online`, `gunfire` needs `stalkers.online`, and `drone` needs `anomalies.near`. The rest need none.

The base is NOT a select filter. A friendly base is silenced by APPLY, which drives dread to 0, and SELECT does not gate it. Selection is a **symmetrical two-level shuffle-bag**, no weights:
a category-bag cycles every eligible category once before repeats (a 2-sound category can never be hammered while others wait), and a per-category sound-bag cycles every sound once.
The sound-bag's POOL is the category's precomputed DREAD POOL for the current scene. `_get_dread_bucket(board.dread)` maps the scene dread to `low` (< 0.40), `med` (< 0.70), or `high` (>= 0.70),
the `DREAD_LIMIT_LOW`/`DREAD_LIMIT_MED` cut points in `dd_director`, and `_select_sound` draws from `_dread_pools[category][bucket]`.
The pools are built once at load (`_build_dread_pools`) from the category's flat sound list plus the per-sound override (below):
a sound with no override lands in ALL three pools (it plays at every dread), and an overridden one only in its pool. The bag is keyed per category+bucket,
so a scene-dread shift draws from that bucket's pool. Rarity emerges from rotation rather than from any limiter.

### The dread override - per-sound dread selection

`dd_spooks_metadata` (the `sounds` table, `["deployed-name"] = { dread = "low"|"med"|"high" }`) is the single source of a sound's dread, read once at load (`_load_dread`).
It is the per-sound business metadata, distinct from the generated audio facts in `dd_sound_metadata`,
and the SELECT-stage counterpart to the per-level and coordinate overrides in `dd_location_override`.
A sound with no entry plays at every dread. A dedicated entry supersedes that, playing ONLY at its dread.
The table is EMPTY by default, so there are no overrides and every sound plays everywhere, exactly as before curation.
Curation adds one line keyed by the deployed name (the stable `<origname>_<hash>`).
It survives every intake and refresh because it is hand-curated data, never generated.
`verify` flags any entry whose name matches no shipped sound (a stale hash after an upstream re-encode silently reverts that sound to play-everywhere).
This coexists with the no-repeat shuffle-bag by construction: the pools are precomputed and stable, so `_select_sound` still draws each pool once before repeating,
and the override decides only pool membership, leaving the rotation intact.

### APPLY - dread drives frequency and a placement pull

Dread is a scalar 0..1, additive, with **no baseline constant**, the sum of whatever grounded conditions hold now:

    dread = lore + environment + time + threat + company + anomaly + service

- **lore** - the level's own baseline (grim in the psi north and the labs, mundane in the fields).
  A hand-marked position override replaces this baseline where one is set (see Position overrides below).
- **environment** - outdoor +0, indoor +small, underground +med, labs +big.
- **time** - day +0, dusk/dawn +small, night +med, deep night +big.
- **threat** - the single scariest near thing, `max(stalkers.enemy_near, monsters.enemy_near)`, scaled by its power tier (man and monster the same). If no living thing is near at all,
  loneliness adds a small term.
- **company** - a near non-hostile stalker calms (neutral or friendly, since any human presence breaks the isolation), scaled by its power (a veteran calms more).
- **anomaly** - an anomaly near adds a little.

A `service_near` of "allied" (a safe hub) REPLACES the sum with 0, silent. "hostile" (an enemy-held base) adds. The base is detected by a live service NPC (trader/medic/mechanic) within 60m,
with per-NPC relation deciding allied versus hostile. Per-NPC relation is warfare-correct, and the over-assigned `is_base` prop is not used. Every term is grounded, so there is no unearned +X.
Dread never gates the CATEGORY (SELECT stays map & env & presence). It drives **frequency** (a shorter gap between plays as dread rises),
the **dread pool** the per-category sound-bag draws from (the scene bucket, filtered by the per-sound override, above),
and a **placement pull** (`play_sound` moves the rolled spawn distance up to 20% closer at peak, `DREAD_PULL`). It never touches the volume table.

The base game's `update_ambient` code places each emitted sound. `get_placement` clones its placement and volume math (`sound_ambient.script`), fed the sound's spawn band from the config.
Under the measured model the category's reach sets the spawn band. ch_max is the reach and ch_min is 0.6 times it, one pair for every sound in the category.
The blob falloff (min/max) and the spawn band come from that one reach and always agree. Every sound has a band whether or not a pack ever wired it.
Placement goes through the vanilla transform: `random(spawn_min, ch_max)/2` (spawn_min = (ch_max+ch_min)/2 outdoors, ch_min underground), a random bearing, and `pos.y + height`.
The play volume is the game ambient slider times the MCM master `vol_global` (1.0 = untouched).
X-Ray uses that volume as the upper bound of the linear falloff, not as a gain (`SoundRender_Emitter_FSM.cpp`).
Below 1 it quiets the near plays. The MCM slider runs 0 to 1, and a saved value above 1 reads as 1.
`is_eligible` already matched the category to the environment, and xray's occlusion raycast muffles walls (`SoundRender_Emitter_FSM.cpp`).
Only the base replay applies the vanilla indoor/outdoor/underground table.
The heard loudness at that distance is the engine's attenuation on the MEASURED blob, and the director decides only WHEN and WHAT. The two distance pairs are never conflated: the blob min/max is the
FADE curve, the spawn band ch_min/ch_max is the SPAWN radius, and both come from the reach. The blob min also drives OpenAL's inverse rolloff.
Height is the category's elevation from the tool's height table, carried as `snd.h`. Birds play 10 m up, rats 1 m down, and everything else at the player's level.

### Position overrides - hand-marked static positions

Some places the live sensors cannot read, an empty bloodsucker village or a surface machinery factory, read mundane because no NPC, anomaly, or level baseline marks their reputation.
`dd_location_override.positions` fixes them by hand: one entry per position with `pos` {x,y,z}, `radius`, an optional `dread`, `add`, and `categories`/`select`.
The director reads the table once into a flat list (`_load_positions`), and the `position` producer resolves the active one each rotation with `xmath.is_in_range` (flat XZ, squared,
no sqrt) over the current level's few and writes it to `board.position`.

Where the actor stands inside a position's radius it ALWAYS wins:

- **dread** replaces the `lore` baseline term (the live terms, threat, company, env, time, and anomaly, still stack on top),
  so the empty village never reads below its set dread but a real bloodsucker still lifts it.
- **add** is a flat modifier on the whole sum.
- The override wins over a safe hub: a position with a `dread` suppresses the allied-service silence (`_resolve_dread`), so a hand-marked place is never zeroed by a passing trader.
- **categories** override SELECT: `select = only` plays ONLY the listed categories there, and `select = add` adds them to the level's map (`is_eligible`), still gated by env and presence.

This layer needs no MCM knob. It is data, edited in the LTX and captured via the dev-tab snapshot. The debug HUD carries an `OVERRIDE` section (the active position's name, dread, add,
select) and an `[OVERRIDE]` trace fires at DEBUG on entering or leaving a position.

### Emission model - how the final loudness is set (engine-grounded)

The engine computes the audible gain per play (`SoundRender_Emitter_FSM.cpp`):

    gain = base_volume x volume_att x effect_volume x occlusion x fade

- **base_volume** - the per-file value in the ogg comment blob (X-Ray native, `SoundRender_Source_loader.cpp`). A direct linear multiplier, applied to mono AND stereo alike.
  The measured master starts it at 1.0 and bounds it into the loudness band off the measured LUFS (floor -36, ceiling -18). An in-band file stays at 1.0.
- **volume_att** - LINEAR distance attenuation `(max_dist - dist)/(max_dist - min_dist)` (`SoundRender_Emitter_FSM.cpp`): full at `min_distance`, silent at `max_distance`, NOT inverse-square.
  On top of it OpenAL applies its OWN inverse-distance rolloff keyed on the blob min (`AL_REFERENCE_DISTANCE`, never disabled by the engine), and the two multiply.
  Measured min/max sit in the blob, where the engine reads them, and in the config as the trace readout.
  The spawn band sits within the same reach. The fade curve and the placement form one geometry.
- **stereo** - the engine force-2Ds any 2-channel buffer (`Core.cpp`: `channels==2 -> switch_to_2D`), and the 2D path skips BOTH attenuation and occlusion (`Emitter_FSM.cpp`:
  `volume_att = p_source.volume`, `occ = 1`), so a stereo sound plays at-ear at full loudness wherever it is placed, the "one sound too loud" blare. Only MONO is spatialised,
  so intake FOLDS every stereo file to mono: sum (L+R)/2, or drop one channel for an anti-phase pair that would sum to silence. Proven from the source.

### Coexistence with a base soundscape - the loudness balance

Every sound is leveled into one loudness band by the measured model (floor -36, ceiling -18, off the measured LUFS at the reach/2 placement), so the corpus is even without a per-sound target and
nothing screams or drops out. The director places each sound within its category reach through the vanilla placement code, so the horror sits IN the mix at a controlled level rather than on top of it.
The one runtime control that balances the two is the MCM master `vol_global` (multiplied into the play volume in `play_sound` alongside the game ambient slider), for a base that runs unusually loud or
quiet. Nothing about the mix depends on the player leveling anything.

So every per-file blob value is the measured model's: `base_volume` bounded into the loudness band, min and max from the reach and the measured crest, the spawn band from the reach, and
`height` taken from the category. The director reads no `indoor` flag. No source min, max, base_volume or spawn band reaches the blob.

### Visual layer

Only when dread is at its peak (>= 0.80) a short distortion pulse fires occasionally through xlibs `xpp`, dwell-gated so a momentary spike never flashes, on a cooldown.
This is the one place a threshold on the continuous dread still matters, and there is no grade ladder otherwise.

### Debug HUD (`dd_hud`, off by default)

A three-column readout (MCM `hud_position`), built lazily on read so it costs nothing on the loop, grouped by stage: PLAYING (the director's current sound plus the base ambient the observer replays,
each bright while sounding, gray once stopped), SELECT (the available category list plus the DREAD number, tinted gray, amber, or red by value, no rainbow), APPLY (each dread term's contribution),
OVERRIDE (the active hand-marked position and what it forces, when one is in range), and SENSORS (every board field by its exact name, `stalkers.enemy_near`, `monsters.online`, worded tokens).
Players never see it, and it feeds off `dd_director.get_hud_rows`.

## Categories - the rule table

A category is atomic, one coherent thing (one dread kind, one zone). The category is the unit of organization, the shipped folder (`zs/<name>/`, a flat directory of oggs,
with dread the per-sound `dd_spooks_metadata` override) and the config key.
The tool's category list holds each name with its reach and height. Routing and play rules live elsewhere.
A category's runtime attributes (its `env` set, `requires` gate, per-map eligibility, and presence checks) live in the director (`dd_director`) and the per-map LTX, keyed by the category name.
The config carries sound paths and per-sound values only, and the category NAME is the entire contract between the pipeline and the runtime.

The 21 categories:

- creature - `mutant`: one pooled bag, gated at runtime on any real mutant being present (a single boolean, not per-species). Distant mutant cries and growls, combat sounds left out.
- zone ambience - `mutant_ambient_forest` / `_swamp` / `_urban` / `_field`: per-zone horror-mutant atmosphere, map-selected per level, no time and no species.
- ambience - `spook`, `scream`, `drone`, `dark_signal`, `industrial`, `structural`, `labs`, `drip`, `wind`, `foliage`, `wildlife`, `birds`, `urban`, `gunfire`, `rats`, `bats`.

The packs' own folders are evidence for the sort. Surface spooks lean to the outdoor kinds, underground spooks to `structural`, `labs` and `drip`, and machinery to `industrial`.
`underground` is the enclosure STATE alone. A category is split only along an axis the runtime acts on: env, per-map zone, presence or placement height.
For that reason the four zones are separate categories. Tree and flying birds split off `wildlife` into `birds` for their height.

## The base-exclusion: static DLTX removal, plus a logging observer

The base game's System B (Lua `sound_ambient.update_ambient`) plays the rotating dread and atmosphere sounds, the vanilla "fake" spooks, drones, and distant-mutant growls.
If the player also runs a soundscape pack the mod drew from, the base plays the same sounds the director does, so they double.
DiegeticDread removes its own sounds from the base's ambient channels STATICALLY, at config load. It runs no muting loop at runtime.

### Static removal (the exclusion)

The overlay `configs/environment/mod_sound_channels_diegeticdread.ltx` is HAND-AUTHORED and committed, not emitted by refresh. For every channel that lists one of our sounds it carries `![channel]`,
a `<sounds = <path>` per removed item (`Xr_ini.cpp`, the Remove op), and `>sounds = ambient\no_sound`. The composed `sound_channels.ltx` the game loads no longer lists our sounds, so vanilla's
`update_ambient` and the engine bed never play them. It is engine-native and survives anything at runtime, since there is no slot to lose.

The removal set is decided by AUDIO, never by path. A channel entry is ours when its ogg is the same recording as a corpus sound: the same audio hash (md5 of the vorbis pages, blob-agnostic), or,
for a re-encode the hash misses, a chromaprint candidate at >= 0.88 confirmed by PCM cross-correlation at >= 0.90, the same identity dedup uses (I3).
A re-leveled or re-encoded copy is still caught, and the generic `sound_NN` names many packs share are told apart by audio, not by their shared path.
The comparison runs against the real channel layouts of the installed packs, which catches a pack that re-homed our sound to another channel.
A human writes the overlay from that comparison. The tool never emits it.

The overlay targets the UNION across all the layouts: a path is removed where it is ours in any mod. DLTX removes by exact string and warn-discards an absent channel or path (`Xr_ini.cpp`), so the
one overlay is correct on any install, each seeing only its own subset apply. Union is chosen over intersection because its failure is over-removal, a distinct base clip going silent, rare and
inaudible under the director's own dread, rather than doubling, the defect the exclusion exists to kill. The residual is the generic `sound_NN` collision: a path carrying our audio in one pack and a
distinct recording in another is removed on both, a handful of clips, accepted.

Bed-empty guard: every touched channel carries `>sounds = ambient\no_sound` alongside its removals.
A full removal then never empties a System A bed, which asserts on empty `sounds` (`Environment_misc.cpp`). `verify` simulates the overlay against every channel layout it is given.
It FAILS if any touched channel ends empty without the guard, and the CTD invariant cannot regress.
It also reports any removal that matches no base entry, because DLTX removes by exact string and a miss leaves the sound playing twice.
The overlay owns only what the director ships: a base sound the mod never captured stays in its channel and plays, and a channel goes silent only when every sound in it is ours.

The overlay is TESTED BY HAND against the mods, all of them, not the one install: load each, confirm the base no longer doubles the director and no channel the overlay touches falls silent on load.
The standalone sound-only builds ship no channel config, so a sound sourced only from them has no wiring to remove; if the same audio also came from an Anomaly pack, that pack's sibling path carries
the removal. The match is by audio signature, not by folder or mod name, and there is no runtime lookup.

### The observer hook (owns the vanilla scheduler slot)

A time-event on the vanilla `update_ambient` slot (`update_base_ambient`, installed by `register_base_observer`) REPLACES vanilla `sound_ambient.update_ambient`, so it runs for EVERY player,
not only at DEBUG. It is a clone of the vanilla channel rotation, timing, and volume rule with added nil-guards, and it has two deltas. It replays through `xsound.play` (an engine-owned single play),
so a base sound is not cut on channel re-fire the way vanilla's retained-handle GC cut it, and it LOGS each base fire at DEBUG (`[BASE]` lines and the HUD BASE row).
It does no muting (the composed config it reads already has our sounds removed) and no injection. It is not there only to log. It owns the base ambience for everyone.
The log plus the no-cut are what it adds over leaving vanilla in place. If another ambient-scheduler mod wins the slot back, only the trace and the no-cut are lost.
The exclusion still holds because it is the static overlay, independent of this hook.
This slot (`sound_channels`/`update_ambient`) is separate from the director's own 100ms loop slot (`dd_director`/`tick`), so the two never share.

## Preservation and proof

- The audio is unchanged, proven. `verify` checks every shipped file's audio hash against its row, blob-agnostic. A rewritten blob never counts as a change, and a hand-edited ogg fails.
- Volume and distance sit in the X-Ray blob, rewritten to the measured model (min/max from the reach and the measured crest, base_volume into the loudness band).
  base_volume starts at 1.0 for every file, a blob-less source included.
- The record keeps no source. A sound's identity is its audio (the content-hash name), and the repo holds no provenance file.
- Every sound is accounted for. `verify` fails on a row without a file, a file without a row, and any row still in `holding`.

## Invariants

- Performance first. Performance is the top priority and outranks features. A feature that cannot meet the budget moves its cost into the engine where the engine can carry it,
  otherwise we rework or remove it. We never keep it at the cost of the budget. Only correctness and never breaking base gameplay rank above it.
- Use the engine, don't work around it. Every capability comes from the engine and the Anomaly layer first, always through xlibs. Our own code enters only where stock behavior falls short.
- I1 Play once, no loops. The director fires every sound once through `xsound.play_at` (retained handle, stoppable).
  There is no loop layer and no continuous bed. A long sound plays once, like any other.
- I2 No channels for our content. Sounds live FLAT in category directories (`zs/<category>/<name>.ogg`) and are named by the sound config (`dd_sound_metadata.script`),
  with dread the per-sound `dd_spooks_metadata` override. The deploy defines no `sound_channels.ltx` channels for our content. The one config that touches base channels is the hand-authored DLTX exclusion overlay,
  which REMOVES our sounds from existing base channels and never adds one. The engine ambient bed and its asserted channels stay intact, so nothing can cause a missing-channel crash.
- I3 Deduplicate by the waveform, source side only. md5 then Chromaprint fingerprint then PCM cross-correlation, complete linkage at 0.90. Distinct variety is never merged.
  Deduplication runs among the source packs, never against the target modpack.
- I4 Fitness is codec plus sample rate: 44100 Hz vorbis. Intake resamples an off-rate file.
- I5 Ship byte for byte where nothing must change, and transform where the engine forces it and book it. A MONO file's audio pages are its own, untouched,
  and only the X-Ray blob is rewritten losslessly to the measured model.
  The category reach and the measured crest set min and max. base_volume is bounded into the floor/ceiling loudness band off the measured LUFS.
  No source min, max, or base_volume is preserved in the blob. See "Byte-for-byte audio".
  Two steps re-encode the audio. Intake folds every STEREO file to mono, because the engine 3D-positions mono only, and it resamples an off-rate file.
  Both are mechanical and deterministic. The source blob is captured before it as an audit column the master ignores.
  Cutting a long bed into pieces makes new sounds. It is authoring, done by hand and logged, never in the intake.
- I6 Take every sound a pack holds, wired or not. Intake reads the pack's whole sound tree, because the packs ship far more than their channel configs list.
- I7 Sorting is manual and per sound. Intake decides no category. Every net-new sound waits in `holding`, and verify refuses the tree until each one is sorted or rejected.
- I8 Remove, do not inject. The static DLTX overlay removes a base sound the mod ships from its channel, matched by audio identity, per item.
  It never adds a channel and never injects a sound into the base ambient. The base's other sounds are untouched.
- I9 Dark scope only. Keep spook, horror, underground, eerie, and oppressive weather. Leave generic daytime life and the base weather bed to the base ambience.
- I10 Leave emission alone. Blowout and psi-storm are their own system and are never touched.
- I11 Reproducible. refresh regenerates the blobs and the runtime display columns from the measured facts (lufs/crest) and the category reach and height.
  It reads no pack and no author value. The same inputs give the same bytes, and a double refresh writes nothing.
  The base-exclusion overlay is the one committed artifact refresh does NOT emit. It is authored by hand, and verify simulates it against the installed channel layouts.
- I12 One record, no source kept. A shipped sound's identity is its audio (the content-hash name).
  The record, `dd_sound_metadata.script`, is the same file the director loads. It keeps no source pack or path, and it is the only state file.
  The readme credits sources collectively per the attribution policy, which never names a build on a public surface.
- I13 The director owns its own play slot only while active. Without xlibs the director is inert (no play), but the exclusion still holds because it is the static DLTX overlay, independent of xlibs.
  The observer clone resets on hour, level, or weather change, so it never replays a channel for the wrong level, and it guards every value an engine call needs.

## MCM and trace

Scripts add control, an in-game trace, and the MCM, mirroring the alife-family pattern (`dd_mcm`, `dd_debug`, `xmcm`, `xlog`). All are guarded. Without xlibs they degrade to no-ops.

- `dd_director.script` owns the director (its own `dd_director`/`tick` slot), the score, the pick and pace, the positioned play, and the base-ambient observer (the separate `update_ambient` slot,
  with the exclusion itself the hand-authored static DLTX overlay).
- `dd_hud.script` is the debug HUD (off by default), a three-column readout built from `dd_director.get_hud_rows`.
- `ui_dd_player.script` is the review and curation tool (gated by the MCM `sound_player` toggle): a standalone keyboard-owning 2D `CUIScriptWnd` modal opened by PageDown (not a PDA tab),
  reusing `ui_dd_player.xml`.
  It browses `zs/<category>/` off disk (via `xfs`) and auditions each sound through the director's own `play_sound` at the director's exact placement (fed the live scene dread), a fixed distance,
  or at-ear. It curates by LOGGING only and never moves or deletes a file. Fifteen one-word tag buttons in five labelled zones - type (outside, inside), level (low, med, high), vol (inaudible, faint, loud, painful), dist (close, far), fate (keep, cut, release, dupe) - and a custom free-text field write `[SOUND]` note lines,
  and a separate PROBE NOTE field writes a `[PROBE]` block with the full ordered sensor dump. Both go to `diegeticdread_notes.txt` (a clean, human-read notes file,
  distinct from the diagnostic `diegeticdread_probe.log`), written by `dd_test.log_sound_note` / `dd_test.log_probe_note`.
  Nothing parses the notes. Curation is hand-edited code a human writes after reading them.
  Its loudness line reads a director mode at the farthest play, which sits at half the band max, pulled in by the mode's dread and lifted by the row height.
  The window owns the keyboard only so its text fields capture typing (a PDA-subdialog editbox never could). There are no command shortcuts,
  and a focused edit box is detected by the parent `OnKeyboard` return so it never doubles as a command. While the window is open the director is auto-silenced (`dd_director.set_muted`),
  restored on close. It replaces the old PDA tab (`ui_dd_player_tab` / `pda_dynamic_tabs`) and the INSERT note popup (`ui_dd_note`), all retired to `.deleted/`.
- `dd_debug.script` is the trace facade. At DEBUG it records every sound played and every term of the dread score to `diegeticdread.log`, so the soundscape is checked by observation.
  Below DEBUG the off path marshals nothing and crosses no luabind bridge.
- `dd_mcm.script` is one MCM page tree. Atmosphere holds a single master volume for our sounds (no per-category sliders), also the balance control between the horror and the player's base ambience,
  since the mod levels only its own corpus (see "Coexistence with a base soundscape"). Visuals toggles the peak-dread screen distortion. Development holds the trace level, a log flush,
  the debug HUD position, the sound-player dev toggle (the review player), and a reset-to-defaults button. Every control is neutral at its default. Labels are in English.

## Tools and data artifacts

- The measured surface, the build tool's shared audio core. One `ffmpeg` `astats`+`ebur128` pass reads `lufs` (integrated loudness),
  `crest` (peak minus RMS), and `peak` (sample peak). `ffprobe` reads `dur`, `sample_rate`, `channels`, and `bit_rate`.
  Chromaprint `fpcalc` reads the fingerprint `fp`, and the fold reads mid/side RMS.
- Where each value is used. Pipeline-time, each drives one step: `lufs`/`crest` the base_volume loudness floor/ceiling band and (`crest`) the min-distance ratio off the reach, `lufs` the dead report,
  `dur` the dedup duration bucket, `sample_rate` the 44100 resample, `channels` the stereo fold, `bit_rate` the dedup quality-winner pick, `fp` the dedup candidate stage
  (decided by PCM cross-correlation), mid/side RMS the fold sum-vs-drop split. Read at RUNTIME, both sets frozen in `dd_sound_metadata` at refresh: `lufs`/`crest`/`peak`/`bv`/`mn`/`mx` feed
  `xsound.compute_delivered_loudness` for the `[PLAY]`/`[BASE]` trace and the review player, and `ch_min`/`ch_max`/`h` feed the director's `get_placement`.
- Committed data in this repo: `dd_sound_metadata.script` (the corpus record AND the runtime table, one file), the ogg tree, and the hand-authored exclusion overlay.
  The measured facts live on each record row, and a sound's identity is its audio. The repo carries no other data file.
- The build tool is private, and this repo holds data only. intake is the once-per-pack ingest into `holding`, and refresh is the forever path (blobs, metadata, verify, audit) with no packs required.

A new pack is adopted by pointing intake at its folder. Its net-new sounds wait in `holding` until the curator sorts or rejects each one.
Refresh then masters them, and verify passes once none is left. The pack is deletable after intake, and the record is committed when the finished tree is copied into this repo.

## Deploy

A gamedata overlay distributed as a GitHub release and moddb addon. The repo holds the buildable source, the tool, the docs, and the audio.
It is wired for local sync and the gamma-redux install through `stalker-manager`.
