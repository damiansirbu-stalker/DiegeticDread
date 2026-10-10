# DiegeticDread - architecture and method

DiegeticDread is a dark ambient layer for S.T.A.L.K.E.R. Anomaly.
It gathers dark and eerie sounds from community soundscape packs, then measures and deduplicates them. It plays them as positioned single sounds.
A runtime director reads where the player stands and who is near.
It does not add engine ambient channels and it does not edit an ambient file. It plays its own sounds, and it mutes the base game's copy of any sound it also ships.

This document is the method and the invariants. The build tool is `diegetic-manager` in stalker-dev, and this repo holds only data. `tools/sources.yaml` is the authored gather config,
`tools/manifest.json` the corpus record, and `tools/measure_cache.json` plus the committed proofs round it out. The tool reproduces every number below.

## Model: a single-sound director, no channels

The old model shipped our sounds as engine sound channels (`sound_channels.ltx` sections) and let the ambient system play them. That is gone. DiegeticDread now owns playback end to end:

- Content lives FLAT in our own category directories under `gamedata/sounds/zs/<category>/<name>.ogg`, never in an engine channel. Dread is not a directory.
  It lives per-sound in `dd_spooks_metadata` (the business-metadata override, below). The deploy writes no `sound_channels.ltx` definitions for our content.
- The director (`gamedata/scripts/dd_director.script`) plays each sound as a positioned single play through `xsound.play_at`, the vanilla `play_at_pos` call shape with a RETAINED handle
  that keeps every playing sound stoppable (`xsound.stop_shots`, the dev-tab Stop) and alive across GC. A long horror drone or a psy bed plays as one sound on a long period, with no loop and no bed.
- The base's own copy of a sound we ship is removed from its ambient channels statically by a DLTX overlay at config load (the exclusion, below), so the base never doubles the director.
  A separate observer owns the vanilla `update_ambient` slot only to replay and log the base ambient.

Because the director is the only playback path, xlibs (`xsound`) is required. Without it the mod is inert and no sound plays. The director's own data is Lua: `dd_sound_metadata`
(generated audio facts), `dd_spooks_metadata` (hand-curated dread), and `dd_location_override` (hand-curated level and coordinate overrides), each an auto-loaded table read once.
`xltx` reads only the engine's own LTX (the base-ambient `sound_channels.ltx`), and `xfs` enumerates on-disk oggs for the review player, which also reads a logs-dir `.txt` playlist through `getFS`.
There is no raw `ini_file` in the mod's own scripts.

The category is the unit of organization and of play. It is a directory of sounds plus two attributes, an `env` set (which enclosure states it may play in) and a `requires` gate (a live precondition),
with no weight and no cooldown. Dread lives per-sound. `dd_spooks_metadata` (the `sounds` table, `["deployed-name"] = { dread = "low"|"med"|"high" }`) is the single source of a sound's dread.
A sound with no override plays at EVERY scene dread (the default), and an overridden one plays ONLY at its dread. Curating a sound is adding one line to that table, keyed by the stable deployed name,
and it survives every gather and master (dread lives in the hand-curated table, never in the tree). The director reads the generated sound config (`dd_sound_metadata.script`) that lists each category,
then that category's flat sound list, each row a named-field table: its blob attenuation pair, its source channel's SPAWN band and `indoor` flag (the author's placement), its height,
and its measured loudness (`lufs`/`crest`/`bv`). The director also flattens those rows into `xsound.load_meta`, so any consumer can read a sound's delivered loudness.
It applies the dread override at load to build the per-dread pools. The config replaces the channel definitions the director used to read from `sound_channels.ltx`.
Anomaly Lua cannot enumerate a directory at runtime, so the deploy writes the config and the director reads it. See "Categories - the rule table" below.
The category list is the single source of truth.

## Content pipeline (reproducible)

`diegetic-manager` runs two phases with a hard boundary. GATHER (`gather DiegeticDread <Source>`) is the only phase that reads a source pack. It executes the authored `tools/sources.yaml`
(registry + route rows), routes, gates 44100, folds stereo, slices the long `dark_signal` beds, culls dead files, and dedups by waveform against itself and the manifest. It freezes everything the
pack can ever tell us into `tools/manifest.json` - author blob, spawn band, height, indoor, exclusion wiring, origin, every collapsed duplicate's path. Its per-pack coverage proof (UNUSED-DARK = 0) and
provenance rows commit as the record, and after they pass the pack is deletable. Gather is idempotent, so a re-run of an ingested pack changes nothing. MASTER (`master DiegeticDread`) never touches a
pack. It recomputes every blob from the manifest's AUTHOR values under the current floors (deterministic, tunable both directions), emits `dd_sound_metadata` and the exclusion overlay, and runs verify
(closure, integrity, schema) plus the reach audit. Dread curation is per-sound in `dd_spooks_metadata` rather than in the tree, so neither phase touches it.

- gather: walk the source pack's sound tree and route each FILE to a category by its folder path (the `routes` rows of `sources.yaml`, a structural per-file allowlist).
  The packs ship far more dark content than they wire into a channel, so the folder trees are the source of truth over any config list. Gate on sample rate, fold stereo to mono
  (the engine 3D-positions mono only, and the author blob is captured BEFORE the fold strips it), drop dead files (true silence after the fold), and run the long-file pass.
  A sound whose ACTIVE (silence-removed) length exceeds the max emission window is culled, except `dark_signal`, which is sliced into short desilenced pieces.
  Deduplicate by waveform against the pack and the manifest. Each duplicate's origin and wiring merges into the surviving row, net-new rows append, and the per-pack coverage proof then runs.
  Every dark file is shipped, booked (dead, long, off-rate, re-encode), or the gather FAILS with the repo untouched. Plan runs entirely in scratch.
  The tree, the manifest, and the proof rows commit only after the proof passes.
- master: recompute each file's blob from its manifest AUTHOR values (attenuation min/max and base_volume unchanged) under the two lift-only floors, then write only the files whose target differs.
  Emit `dd_sound_metadata.script` and the base-exclusion DLTX overlay from the manifest exclusion rows.
  The metadata row carries the category's flat sound list, and per sound the blob pair, spawn band, `indoor`, height, and measured `lufs`/`crest`/`peak`/`bv`.
  Verify covers manifest-tree closure both ways, id-vs-audio integrity (a hand-edited ogg trips it), the manifest schema gate, the dangling-dread report, and then the reach audit.

### No audible sound drops before audition

No audible sound is dropped before the user auditions it.
Measurement only FLAGS a drop candidate (too long, off-character, past a spectral or loudness bound). It never excludes an audible file on its own.
The flagged list is loaded into `ui_dd_player` as a playlist, the user auditions it, and only then does a file get excluded.
The sole mechanical removals are files that cannot be auditioned: dead-silent (below the LUFS floor), off sample rate, corrupt, or an anti-phase pair that folds to silence.

### Selection is manual, pulling is mechanical

Which packs and which folders contribute is a per-pack judgment, made by hand before any pull. Each new pack is opened, then assessed for what dark content it actually holds,
and its folders are mapped to categories by adding rows to `ROUTE` (the structural per-file table `route` walks). The folder-path match is only the mechanical pull that runs after that decision,
and anything unmatched is dropped (dark scope). There is no keyword classifier that decides scope on its own.
The `UNUSED-DARK = 0` ledger invariant then confirms the hand-written rules captured every dark file the pack holds.

The source list is the registry in `tools/sources.yaml`: one declarative entry per pack with `name`, local `path`, reference `url`, and `licence`, order preserved (dedup and band-provenance
rank are order-sensitive). Entries are never deleted after gather - the path may go stale, the licence/credit record stays. Sources are ALWAYS pulled locally by hand, and the pipeline never
downloads. The `url` is a credit and provenance reference only. A licence gate stops a gather FROM any source not cleared (`licence: pending`).
The `diegetic-manager provision DiegeticDread` command reports each source present or MISSING, and a missing pack blocks only ITS gather - master never needs one.

### Identity and the corpus of record

Each file is named by content: `zs/<category>/<origname>_<audiohash>.ogg`, the hash an md5 of the audio pages only (blob-agnostic), short form. The name is readable and keeps the source name.
The hash disambiguates the generic `sound_NN` names 27 packs share, and identical audio always maps to the same name, so adding content never renames anything.
The manifest row key is `<category>/<name>`, category-qualified because one pack ships identical audio in two folders routed to two categories (the labs/drone twins).

The corpus of record is `tools/manifest.json`.
Each committed row is one per sound ever gathered. It holds its origin pack and path, every collapsed duplicate's (pack, path), and the pre-fold AUTHOR blob.
The row also carries the spawn band and its provenance (same-author / dup-pack / other-pack / unwired), height, indoor, the exclusion strings, fold and slice lineage, duration, and fingerprint.
A `seq` ordinal keeps every emit byte-stable. Rejected sounds stay as rows (status plus fingerprint), so the next gather flags a re-encode of a reject and keeps it out.
The manifest is why the packs are deletable. Master recomputes everything from it.

- Gathering is ALWAYS additive: the published corpus is frozen, a new pack dedups against the manifest (recorded paths first, then exact audio hash, then chromaprint
  candidates decided by PCM cross-correlation at >= 0.90), origins and wiring MERGE into existing rows, and only net-new audio enters, appended with the next `seq`.
  A re-gather of an ingested pack is byte-idempotent. The manifest does not change.
- Limits: re-encode detection is heuristic (fp >= 0.88 candidate, xcorr >= 0.90 decide), not exact. A quality upgrade (a better copy of a shipped sound arriving later)
  never happens implicitly - the frozen copy wins. Upgrading is a deliberate reject-and-regather.

## Deduplication: waveform identity, source side only

Identity is decided by the waveform. There are four stages, cheapest first, so the expensive test runs only on the pairs the cheap ones flag (`dedupe`, `pcm_correlation`).

- md5: byte-identical reships across packs collapse to one.
- audio hash (n124): among the md5 survivors, files identical in AUDIO but differing only in their comment blob (a reship with a different volume or distance) collapse here (`_hash_audio`,
  exact and blob-agnostic), before the fuzzy stage that can miss them. Every collapsed path goes into the survivor's `dups` for the exclusion.
- Chromaprint fingerprint (`fpcalc`, >= 0.88): stable across bitrate and codec, so it finds the re-encoded copies md5 misses. Its same-versus-distinct ranges overlap,
  so it only proposes candidate pairs and never decides.
- PCM cross-correlation (`DEDUP_XCORR` = 0.90): decode both, line them up by envelope offset, and correlate over the overlap. A re-encode scores near 1.0 and a distinct sound near 0.
  Two files merge only under complete linkage, so a similarity chain never collapses two distinct recordings and variety is never lost.

The survivor of every collapsed group is the HIGHEST-BITRATE copy (`dedupe`: `max(group, key=bitrate)` at the md5, audio-hash, and cross-correlation stages),
and a sub-`LOWQ_BITRATE` (32 kbps) file is dropped only when its category keeps something better (`good if good else chosen`). So a low-bitrate survivor is the best, or the only,
copy of that sound across every source we pull. The low-bitrate tail is old SoC-lineage source recordings (SoP, NLC, OGSE, Solyanka, DeadAir) that were never released higher,
kept because re-encoding cannot restore detail the source never captured. Modern packs contribute none.

This runs among the source packs only, within a pack and between the packs we pull from. DiegeticDread does not deduplicate against the target modpack.
It never drops a sound because the install already plays it. Doubling with the base is handled by the static DLTX exclusion overlay at config load.

## Byte-for-byte audio, author's blob unchanged

A MONO sound's AUDIO ships byte for byte with no re-encode, the vorbis pages untouched. What the deploy writes is only the X-Ray ogg comment blob (min distance, max distance, base_volume),
a lossless header-page rewrite (`_write_blob`), so the audio pages stay byte-identical. A STEREO sound is the exception. The engine cannot 3D-position it,
so the deploy folds it to mono (`_masterize_channels`, re-encoded, booked "cut"), then writes the blob. Its author's blob is captured from the source BEFORE the fold strips it (`src_blob`),
so the fold does not lose the author's values.

Blob contract (the engine-read fields). The `0x0003` comment struct (`SoundRender_Source_loader.cpp`) holds five fields: `min`, `max`, `base_volume`, `game_type`, and `max_ai_dist`.
`_write_blob` writes all five, and the engine reads exactly those. Nothing written is ignored, nothing read is left unset.
`min`, `max`, and `base_volume` carry the attenuation and loudness below, the linear rolloff plus the OpenAL inverse keyed on `min` and the `base_volume` gain (`SoundRender_Emitter_FSM.cpp`).
`game_type` is written 0, and `max_ai_dist` is written equal to `max`, both inert for our content.
The play-time sound type overrides the blob `game_type` (`SoundRender_Core.cpp`).
`max_ai_dist` only sets NPC hearing range, which our `no_sound`-type plays never trigger (`SoundRender_Emitter.cpp`).
`max_ai_dist` is set to `max` only to satisfy the loader's `>= 0.1` assert, the one engine-read lever left deliberately unused.
A `world_ambient` play with an owner would make NPCs hear the sound, which the director does not want.

- **Attenuation** (min/max distance) is the AUTHOR's, with ONE uniform, declared transform: the min_distance FLOOR (`_normalize_blobs`). 61% of the corpus carries min 1-2, the UNSET tool default.
  OpenAL's inverse rolloff keys on min (`AL_REFERENCE_DISTANCE`). Those files therefore lost -25..-33 dB at their felt placement and played silent.
  The floor raises each file's written min to `ratio x its felt-far distance` (band_max/2). It never lowers an authored min,
  so the distance-baked 300-10000m sounds and every deliberately-authored range stay untouched. max is kept unchanged, and the guard `max > min` keeps the engine divide safe.
  - The ratio is NOT flat. PRINCIPLE (measured, mild-moderate, automated): placement loudness follows CREST, INVERTED (`_crest_ratio`). A sustained low-crest tone carries in air,
    so a higher ratio keeps it present at distance. A sharp high-crest transient is a near-field detail, so a lower ratio keeps it intimate. Crest is measured per file (the measurement cache).
    Verified across the corpus: `drip` (24 dB), `rats`, and `foliage` are the sharp near-field sounds, and the sustained dread (`drone`/`scream`/`mutant`/`spook`, ~6-7 dB) carries.
    Span 0.40-0.60 (`RATIO_LO`/`RATIO_HI`) centers on the old flat 0.5, with ~3 dB of far-edge spread, so scares and beds separate without anything dropping to silence.
    This restores the near/far loudness depth a single flat ratio had removed. A blob-less file gets its category-folder median min/max, then the same crest floor.
- **Loudness is the AUTHOR's base_volume with ONE lift-only correction: the loudness FLOOR** (`_loudness_floor`).
  base_volume is a linear multiplier the engine applies on every play (see `sound-source-and-emitter.md`). The deploy writes each file's authored base_volume UNCHANGED,
  except where its measured DELIVERED loudness at the felt-far placement (`content_LUFS + 20log10(base_volume x volume_att x al_gain)`, LUFS from `_measure_corpus_lufs`,
  ffmpeg ebur128 on the deployed ogg, cached) falls below the faint-audible level (`LOUD_FLOOR` -36 dB). Then base_volume is lifted toward the floor,
  PARTIAL (`LOUD_FRAC` 0.7 of the deficit) and CAPPED (`LOUD_MAXBV` 6 plus a far-gain cap `LOUD_FARCAP` that keeps the sound below the engine's 1.0 clamp so distance falloff survives). LIFT-ONLY:
  it never lowers an author value, so loud files and their dynamics are untouched. It fixes quiet CONTENT the min floor cannot (a -40 LUFS recording plays inaudible even at flat full gain,
  for example drip, min-floored to 80 = the loudest placement, still inaudible). It is NOT the corpus-median RMS leveling (n126,
  REMOVED for blowing the tight 0.5-2.0 out to 0.1-28 and deafening the transient scares). A floor raises only the too-quiet toward faint-audible and stops. ~37% of the corpus is lifted,
  and the loud or reference sounds are untouched. A file with no authored value (or <=0) starts at base_volume 1.0, then the floor.
- **Dead-file cull** (`_cull_dead`): after the fold, a file whose audio is unmeasurable or true silence (peak -inf) is DROPPED.
  The fold can cancel an anti-phase pair to silence that `_drop_silent` (pre-fold) could not see. There is NO quiet-cull: a quiet-but-real sound ships at its author's loudness,
  and a faint feel comes from the director's placement rather than from removing content.

The config (`dd_sound_metadata.script`) carries, per sound, the blob attenuation pair (read back from the deployed ogg, trace readout only),
the source channel's spawn band and `indoor` flag (the author's placement, which `play_sound` feeds to the vanilla formula), the source-channel height,
and the measured loudness (`lufs`/`crest`/`bv`) fed to `xsound.load_meta` for the delivered-loudness readout.

Fitness gate: 44100 Hz vorbis only, the X-Ray standard. Off-rate and junk-bitrate files are dropped and accounted, never silently. The 44100 rule is the engine's own.
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
later (n118), EVENT (bumped by callbacks, decayed). A sensor's fields match what is useful. Mobile things carry `online` plus `near`, and static anomalies carry only `near`.

- `environment` - "outdoor" | "indoor" | "underground" | "labs". `GetEvent("underground")` says you are on an underground level, `LAB_LEVELS` splits the sci-fi labs from the tunnel/mine/bunker levels,
  and otherwise the `is_indoor` raycast (cached by cell) gives indoor versus outdoor.
- `time` - "day" | "dusk" | "dawn" | "night" | "deep_night".
- `stalkers` - one pass over the dedicated `db.OnlineStalkers` array (`_collect_stalkers`): `online` (any stalker online, the gunfire gate),
  `enemy_near` and `ally_near` (the strongest near hostile and the strongest near non-hostile, each a power tier,
  where a neutral NPC counts as company because only a per-NPC enemy relation is a threat),
  and `service_near` ("none"|"allied"|"hostile", a trader/medic/mechanic within 60m = a base).
- `monsters` - one pass over `xcreature.iterate_online_monsters` (`_collect_monsters`): a demonized `db.OnlineMonsters` id-array when present,
  else a best-effort `is_mutant` walk capped at `MONSTER_ITER_CAP` online objects so the scan stays bounded under soak (a monster past the cap is missed),
  where the registry PR is the parity fix (n119): `online` (the mutant gate), `enemy_near` (strongest near monster as a power).
- `anomalies` - `near` (an anomaly within range, `xsmart.is_anomaly_near`), the drone gate.

### Power - man and monster on one scale

Every hostile near thing is graded "none"|"low"|"med"|"high" by `get_power_tier(rank)`. A stalker's `character_rank()` (0-27000, cuts 9000/14500) and a monster's `se:rank()` (1-20,
cuts 5/16) collapse to the SAME token. So `stalkers.enemy_near` and `monsters.enemy_near` are the same type, and threat reads both with one rule (`max`).
The engine facts and the probe-verified monster rank table are in `doc/library/modding/npc-strength-evaluation.md`.

### The loop and the emission gate

The loop runs a fixed 100ms and never stops, so a rising dread is always caught, and the board refreshes over the producer rotation. Emission is separate:
the gap between plays scales with the CURRENT dread, from a calm mean of ~32s (`SPACE_CALM_MS`, silence carries the dread) to a peak mean of ~6.5s (`SPACE_PEAK_MS`),
power-eased off the calm end (`SPACE_EASE`) so mid-dread lands ~15s (drops fast then fine-grades toward peak), with a `SPACE_JITTER` spread. The gap is re-evaluated every 100ms against the live dread,
so a rising dread tightens it at once and a threat that appears mid-calm-gap fires promptly even during the long calm interval.
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
  and indoor never plays the outside kinds (foliage, wind, wildlife, urban, the zones).
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
It survives every gather and master because it is hand-curated data, never generated.
`_report_dangling_dread` at deploy flags any entry whose name matches no shipped sound (a stale hash after an upstream re-encode silently reverts that sound to play-everywhere).
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

The base game's `update_ambient` code places each emitted sound.
`get_placement` clones its placement and volume math (`sound_ambient.script`), fed the sound's OWN source-channel values from the config.
gather harvests the channel SPAWN band (`ch_min`/`ch_max`) from the same channel files the exclusion reads, and freezes it in the manifest, SAME-AUTHOR.
The band comes from the same pack as the shipped copy's blob. Blob and placement must be one author's pair, or the combination reproduces nobody's mix.
Fallbacks run in order: a collapsed duplicate's own pack, then any pack wiring the path, then the UNWIRED fallback.
The UNWIRED fallback gives the sound its CATEGORY CENTER, the median of that category's wired bands, or of its own blobs when nothing in it is wired.
It adds a deterministic +/-25% jitter (seeded by the deployed name so a re-master never reshuffles), capped to the sound's own blob max so it is never placed past its silence point.
This replaces an earlier own-blob fallback that flung a default 1-300 blob out to 150m.
The category center places a folder-only sound where that category actually sits. Per-category plus the blob-max cap means no cross-category leak.
Every cross-pack comparison follows the sources.yaml registry order, Shrike's latest Amplified line first, the same preference dedup uses to pick the winning copy,
and within one pack the author's largest-max wiring wins.
This goes through the vanilla transform, with min lifted to the band midpoint outdoors, then `random(min, max)/2`, a random bearing, `pos.y + height`,
and the vanilla indoor/outdoor/underground volume table times the game ambient slider sets the play volume, with the MCM master `vol_global` on top (1.0 = untouched).
The heard loudness at that distance is then the engine's attenuation on the AUTHOR's blob (min/max plus base_volume). The author placed it, the author leveled it,
and the director decides only WHEN and WHAT. The two distance pairs are never conflated. The blob pair is the FADE curve, and the channel pair is the SPAWN band (see `sound-source-and-emitter.md`,
attenuation range versus spawn radius). The blob min also drives OpenAL's second rolloff, which is exactly why the author's spawn band must be used and no invented distance is substituted.
Height is the sound's ORIGINAL source-channel elevation (harvested at gather, highest non-zero wins), and the manifest and config carry it as `snd.h`.
An overhead sound (bird, vent, thunder) stays overhead.

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
  The deploy writes the AUTHOR's base_volume, lifted only by the loudness floor for the too-quiet (`_loudness_floor`, see "Byte-for-byte audio"). It never re-levels the corpus.
  The authored values are tight (~0.5-2.0) and field-tested, so the corpus is even without imposing a target, and a blob-less file gets 1.0.
- **volume_att** - LINEAR distance attenuation `(max_dist - dist)/(max_dist - min_dist)` (`SoundRender_Emitter_FSM.cpp`): full at `min_distance`, silent at `max_distance`, NOT inverse-square.
  On top of it OpenAL applies its OWN inverse-distance rolloff keyed on the blob min (`AL_REFERENCE_DISTANCE`, never disabled by the engine), and the two multiply (see `sound-source-and-emitter.md`,
  "The OpenAL layer"). This is why placement must use the author's channel band. The whole stack is what the author leveled against.
  The AUTHOR's min/max sit in the blob (the engine reads them) and in the config (the trace readout).
- **stereo** - the engine force-2Ds any 2-channel buffer (`Core.cpp`: `channels==2 -> switch_to_2D`), and the 2D path skips BOTH attenuation and occlusion (`Emitter_FSM.cpp`:
  `volume_att = p_source.volume`, `occ = 1`), so a stereo sound plays at-ear at full loudness wherever it is placed, the "one sound too loud" blare. Only MONO is spatialised,
  so the deploy FOLDS every stereo file to mono (`_masterize_channels` -> `sp.to_mono`: sum (L+R)/2, or drop a channel for an anti-phase pair). Proven from the source.

### Coexistence with a base soundscape - the loudness balance

DiegeticDread does NOT RE-LEVEL its corpus. Each sound keeps its author's own base_volume (~0.5-2.0), except where the loudness floor lifts a too-quiet file toward faint-audible (lift-only, above).
The director plays each sound from its author's own channel spawn band through the vanilla placement code, so its PLAYED loudness is the source mod's,
minus the floor's correction for the un-audibly quiet. The authored levels are field-tested and sit in the range a typical base bed occupies, so the horror sits IN the mix rather than on top of it.
The one runtime control that balances the two is the MCM master `vol_global` (multiplied into the play volume in `play_sound` alongside the game ambient slider),
for a base that runs unusually loud or quiet. Nothing about the mix depends on the player leveling anything.

So everything per-file is the AUTHOR's: `base_volume` (lifted only by the loudness floor for the too-quiet), `max` unchanged, the spawn band, `indoor` flag, and `height` from the source channel,
with two corrected fields the pack tooling never authored, the min_distance floor and the base_volume loudness floor (above).

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
**The pipeline category list carries only the name and the folder routing** (the `routes` rows of `tools/sources.yaml`, executed by the tool's dread seam) and holds no play rules.
A category's runtime attributes (its `env` set, `requires` gate, per-map eligibility, and presence checks) live in the director (`dd_director`) and the per-map LTX, keyed by the category name.
The config carries sound paths and per-sound values only, and the category NAME is the entire contract between the pipeline and the runtime.

The 20 categories:

- creature - `mutant`: one pooled bag, gated at runtime on any real mutant being present (a single boolean, not per-species). Fed by `monsters/<species>` (combat filtered out), `soundscape/mutants`,
  and the flat `spooks_above/mutants` trees. Species is preserved in provenance only.
- zone ambience - `mutant_ambient_forest` / `_swamp` / `_urban` / `_field`: per-zone horror-mutant atmosphere, map-selected per level, no time and no species.
  Fed by the terrain-split `trx/spooks_above/<zone>{day,night}mutants` trees.
- ambience - `spook`, `scream`, `drone`, `dark_signal`, `industrial`, `structural`, `labs`, `drip`, `wind`, `foliage`, `wildlife`, `urban`, `gunfire`, `rats`, `bats`.

Categories are de-mixed from the source structure by folder path (`spooks_above` = surface, `spooks_below` = underground): the old `machine` bucket becomes surface `industrial`,
the underground mega-bucket becomes `labs` (`underground` is now only the enclosure STATE, no longer a category), terrain mutants split off into the four `mutant_ambient_<zone>` zones,
`creak` becomes `foliage`, and vermin split into `rats` and `bats`. A category is split only along a filter axis the runtime acts on (env, per-map zone, presence),
which is why the four zones are separate categories but the underground kinds collapse into `labs`.

## The base-exclusion: static DLTX removal, plus a logging observer

The base game's System B (Lua `sound_ambient.update_ambient`) plays the rotating dread and atmosphere sounds, the vanilla "fake" spooks, drones, and distant-mutant growls.
If the player also runs a soundscape pack the mod drew from, the base plays the same sounds the director does, so they double.
DiegeticDread removes its own sounds from the base's ambient channels STATICALLY, at config load. It runs no muting loop at runtime.

### Static removal (the exclusion)

`master` generates a DLTX overlay, `configs/environment/mod_sound_channels_diegeticdread.ltx`, from the manifest's exclusion rows. It is derived from the pipeline's OWN record, the chosen corpus,
rather than from any installed pack. Every shipped sound was captured from a registry source (`tools/sources.yaml`) at a known path, and a source wires that path to a channel only in its own config,
the same file a user running that pack loads. So for each shipped sound the generator reads its origin pack's channel files and emits, for every channel that lists the path,
`![channel]` plus `<sounds = <path>`, a per-item DLTX removal (`Xr_ini.cpp`, the Remove op) that strips exactly that sound from the channel's `sounds` list and leaves the channel's other sounds.
The composed `sound_channels.ltx` the game loads no longer lists our sounds, so vanilla's own `update_ambient` (and the engine bed) never plays them. It is deterministic, engine-native,
and it survives anything at runtime, since there is no slot to lose.

- Complete by construction, install-independent. The overlay excludes every sound we ship at every source path we drew it from.
  A user running one of our source packs has that pack's identical channels, so our removal applies. A pack we never sourced holds none of our audio, so there is nothing of ours to double there.
  Coverage does not depend on the build machine's modlist. The wiring is harvested once per pack at gather (the exact strings, per pack and channel),
  and the overlay is regenerated from the manifest on every `master`, like every other emitted artifact (I11).
- Identity is the SOURCE PATH rather than a runtime file hash. The generator matches each chosen sound's recorded path against its origin pack's channel entries.
  It never scans an install or hashes a played file. The same recording often ships in several source packs, byte-identical or a re-encode,
  and dedup collapses those copies to one while keeping the (pool, source_path) of every collapsed copy on the survivor (`dups`, set in `dedupe`, folded across categories by `_fold_dups`).
  Those copies are the SAME recording, confirmed by the PCM cross-correlation decider under complete linkage (I3). The exclusion removes each copy at its own pack's path,
  so whichever source pack the player runs, that pack's copy of the sound is taken out. Coverage is per sound across every pack we drew it from.
- Every ambient channel file is read per source: `sound_channels.ltx`, `ambient_channels/backgrounds.ltx`, and `ambient_channels/blowout_channels.ltx` (`_source_channels_raw`). The bed files matter.
  Packs file our captured `whisper_*` and `underground_*` into CONTINUOUS beds in `backgrounds.ltx`, which would double under the director if only `sound_channels.ltx` were read.
- Per-SOUND by design, never per-channel. A base spook the mod did NOT capture (dropped by dedup or the loudness cull, so it was never shipped) stays in its channel and still plays.
  The exclusion owns only what the director ships, and the base keeps the rest, so a channel goes silent only when every sound in it is ours.
  This is deliberate. The base's own uncaptured atmosphere is not ours to remove. Verified: `out_screams` removes 24 of its 25 base screams (exactly the captured ones),
  leaving `sound_13` (uncaptured).
- A shipped sound with no removal is not a gap. Structural capture pulls whole folder trees, not the channel-wired files alone (I6),
  so a sound the pack never lists in any channel is still captured and shipped, with nothing to remove. The base plays it through no channel.
  The overlay header reports the split (source paths wired-and-removed versus folder-only) so coverage is visible.
- Bed-empty guard: every removed-from channel also gets `>sounds = ambient\no_sound`, so a full removal never empties a System A bed (it asserts on empty `sounds`, `Environment_misc.cpp`).
  no_sound is silent, so a partially-removed channel is only marginally diluted.
- One-time cost, none at runtime. DLTX composes the overlay into `sound_channels.ltx` once at config load and caches the merged result (`Xr_ini.cpp`). Nothing re-applies it per frame,
  the base observer caches the parsed channels per level/hour/weather reset, and the ambient reads the shorter composed list, so the removals add no runtime cost however many there are.
  An absent channel or path on the player's install is warn-and-discard at load.
- An absent channel is safely ignored by DLTX (warn-and-discard, no CTD, `Xr_ini.cpp`). The standalone builds under `stalker_anomaly_mods/game_builds_for_sound` ship no Anomaly channel config,
  so a sound sourced only from them has no wiring to remove. If the same audio also came from an Anomaly pack, that pack's sibling path carries the removal.

The removal is on the sound's own source path, emitted as the source config wrote it (original case and backslashes) so it matches the base list item exactly.
The engine removes by exact string (`Xr_ini.cpp`), and because the overlay is built from the source pack's own config, that string is byte-identical to what a user running the pack loads.
There is no folder blocking, no mod names, no runtime lookup, and no `provenance.tsv`.

### The observer hook (owns the vanilla scheduler slot)

A time-event on the vanilla `update_ambient` slot (`update_base_ambient`, installed by `register_base_observer`) REPLACES vanilla `sound_ambient.update_ambient`, so it runs for EVERY player,
not only at DEBUG. It is a clone of the vanilla channel rotation, timing, and volume rule with added nil-guards, and it has two deltas. It replays through `xsound.play` (an engine-owned single play),
so a base sound is not cut on channel re-fire the way vanilla's retained-handle GC cut it, and it LOGS each base fire at DEBUG (`[BASE]` lines and the HUD BASE row).
It does no muting (the composed config it reads already has our sounds removed) and no injection. It is not there only to log. It owns the base ambience for everyone.
The log plus the no-cut are what it adds over leaving vanilla in place. If another ambient-scheduler mod wins the slot back, only the trace and the no-cut are lost.
The exclusion still holds because it is the static overlay, independent of this hook.
This slot (`sound_channels`/`update_ambient`) is separate from the director's own 100ms loop slot (`dd_director`/`tick`), so the two never share.

## Preservation and proof

- Audio is byte for byte, proven. Each gather self-verifies every shipped file's audio hash against its source before the proof rows commit. The record stands at every file matched, no mismatch,
  comment-blob-agnostic so a written blob does not count as a change.
- Volume and distance sit in the X-Ray blob. A source file that shipped with a blob keeps it exact. A blob-less file gets the category-folder median, base_volume 1.0,
  which is an approximation and is booked as one.
- `provenance.tsv` maps every shipped sound (its deployed `zs/<category>/<name>`) to its origin mod, source directory, and filename, plus the deployed base_volume,
  and self-verifies each by audio hash against the source. Categories are not channels, so there are no channel/period/section columns. Nothing loses its origin under the content-hash rename.
- `ledger.tsv` (the historical full-coverage proof) books every source dark sound: shipped, held, or excluded with a reason (emission-domain, intra-corpus re-encode, dead-silent, off-rate, off-scope).
  Each gather since runs the same proof per pack and fails on any uncaptured dark file.
  The invariant is `UNUSED-DARK = 0`. No net-new dark sound is left uncaptured.

## Invariants

- Performance first. Performance is the top priority and outranks features. A feature that cannot meet the budget is reworked or removed with an X-Ray engine modification,
  never kept at the cost of the budget. Only correctness and never breaking base gameplay rank above it. See `doc/standards/stalker-code.md`.
- Use the engine, don't work around it. Every capability comes from the engine and the Anomaly layer first, always through xlibs. Our own code enters only where stock behavior falls short.
- I1 Play once, no loops. The director fires every sound once through `xsound.play_at` (retained handle, stoppable). There is no loop layer and no continuous bed. A long sound plays on a long period,
  tuned to its measured duration.
- I2 No channels for our content. Sounds live FLAT in category directories (`zs/<category>/<name>.ogg`) and are named by the sound config (`dd_sound_metadata.script`),
  with dread the per-sound `dd_spooks_metadata` override. The deploy defines no `sound_channels.ltx` channels for our content, and the only config it writes is the DLTX exclusion overlay,
  which REMOVES our sounds from existing base channels and never adds one. The engine ambient bed and its asserted channels stay intact, so nothing can cause a missing-channel crash.
- I3 Deduplicate by the waveform, source side only. md5 then Chromaprint fingerprint then PCM cross-correlation, complete linkage at 0.90. Distinct variety is never merged.
  Deduplication runs among the source packs, never against the target modpack.
- I4 Fitness is codec plus sample rate: 44100 Hz vorbis. Off-spec files are dropped and accounted.
- I5 Ship byte for byte where nothing must change, and transform where the engine forces it and book it. A MONO file's audio pages are its own, untouched,
  and only the X-Ray blob is rewritten losslessly, carrying the AUTHOR's own base_volume and max (`_normalize_blobs`) with no corpus re-level, plus TWO declared LIFT-ONLY corrections:
  the min_distance floor (fixing the field the pack tooling left at the unset default) and the base_volume loudness floor (lifting content too quiet to be audible). See "Byte-for-byte audio".
  A file that never carried a blob gets its category-folder median, then the same floors. Two steps re-encode and are booked "cut":
  a `dark_signal` bed too long for the emission window is sliced into desilenced pieces,
  and every STEREO file is folded to mono (`_masterize_channels`) because the engine 3D-positions mono only (its author's blob is captured pre-fold and re-written,
  so the fold does not lose the author's values).
- I6 Capture from folder trees, not the wired files alone. The ledger proof is what drives UNUSED-DARK to 0.
- I7 Selection is manual and per-pack. A pack's folders are mapped to categories by hand in `ROUTE` after the pack is assessed. The folder-path match only pulls.
- I8 Remove, do not inject. The static DLTX overlay removes a base sound the mod ships from its channel, matched by audio identity, per item.
  It never adds a channel and never injects a sound into the base ambient. The base's other sounds are untouched.
- I9 Dark scope only. Keep spook, horror, underground, eerie, and oppressive weather. Leave generic daytime life and the base weather bed to the base ambience.
- I10 Leave emission alone. Blowout and psi-storm are their own system and are never touched.
- I11 Reproducible. master regenerates every emitted artifact (blobs, `dd_sound_metadata`, the exclusion overlay) from the manifest deterministically - same inputs, same bytes, no packs.
- I12 Traceable. Every shipped sound resolves to its origin via `provenance.tsv`. Every source file resolves to a ledger category. Credit every source pack, author, and link in the readme.
- I13 The director owns its own play slot only while active. Without xlibs the director is inert (no play), but the exclusion still holds because it is the static DLTX overlay, independent of xlibs.
  The observer clone resets on hour, level, or weather change, so it never replays a channel for the wrong level, and it guards every value an engine call needs.

## MCM and trace

Scripts add control, an in-game trace, and the MCM, mirroring the alife-family pattern (`dd_mcm`, `dd_debug`, `xmcm`, `xlog`). All are guarded. Without xlibs they degrade to no-ops.

- `dd_director.script` owns the director (its own `dd_director`/`tick` slot), the score, the pick and pace, the positioned play, and the base-ambient observer (the separate `update_ambient` slot,
  with the exclusion itself the static DLTX overlay the deploy generates).
- `dd_hud.script` is the debug HUD (off by default), a three-column readout built from `dd_director.get_hud_rows`.
- `ui_dd_player.script` is the review and curation tool (gated by the MCM `sound_player` toggle): a standalone keyboard-owning 2D `CUIScriptWnd` modal opened by PageDown (not a PDA tab),
  reusing `ui_dd_player.xml`.
  It browses `zs/<category>/` off disk (via `xfs`) and auditions each sound through the director's own `play_sound` at the director's exact placement (fed the live scene dread), a fixed distance,
  or at-ear. It curates by LOGGING only and never moves or deletes a file. Five premade one-word buttons (inaudible, faint, loud, unfit, good) and a custom free-text field write `[SOUND]` note lines,
  and a separate PROBE NOTE field writes a `[PROBE]` block with the full ordered sensor dump. Both go to `diegeticdread_notes.txt` (a clean, human-read notes file,
  distinct from the diagnostic `diegeticdread_probe.log`), written by `dd_test.log_sound_note` / `dd_test.log_probe_note`.
  The window owns the keyboard only so its text fields capture typing (a PDA-subdialog editbox never could). There are no command shortcuts,
  and a focused edit box is detected by the parent `OnKeyboard` return so it never doubles as a command. While the window is open the director is auto-silenced (`dd_director.set_muted`),
  restored on close. It replaces the old PDA tab (`ui_dd_player_tab` / `pda_dynamic_tabs`) and the INSERT note popup (`ui_dd_note`), all retired to `.deleted/`.
- `dd_debug.script` is the trace facade. At DEBUG it records every sound played and every term of the dread score to `diegeticdread.log`, so the soundscape is checked by observation.
  Below DEBUG the off path marshals nothing and crosses no luabind bridge.
- `dd_mcm.script` is one MCM page tree. Atmosphere holds a single master volume for our sounds (no per-category sliders), also the balance control between the horror and the player's base ambience,
  since the mod levels only its own corpus (see "Coexistence with a base soundscape"). Visuals toggles the peak-dread screen distortion. Development holds the trace level, a log flush,
  the debug HUD position, the sound-player dev toggle (the review player), and a reset-to-defaults button. Every control is neutral at its default. Labels are in English and Russian.

## Tools and data artifacts

- The measured surface, the shared audio core in `diegetic-manager/core/` (stalker-dev), tools from `$PORTX_ROOT/packages`. One `ffmpeg` `astats`+`ebur128` pass reads `lufs` (integrated loudness),
  `crest` (peak minus RMS), and `peak` (true peak). `ffprobe` reads `dur`, `sample_rate`, `channels`, and `bit_rate`. Chromaprint `fpcalc` reads the fingerprint `fp`, and the fold reads mid/side RMS.
- Where each value is used. Pipeline-time, each drives one step: `lufs`/`crest` the base_volume loudness floor and (`crest`) the min-distance ratio, `peak` the dead-file decode-artifact clamp,
  `dur` the long-file cull and the `dark_signal` slice, `sample_rate` the 44100 fitness gate, `channels` the stereo fold, `bit_rate` the dedup quality-winner pick, `fp` the dedup candidate stage
  (decided by PCM cross-correlation), mid/side RMS the fold sum-vs-drop split. Read at RUNTIME, both sets frozen in `dd_sound_metadata` at master: `lufs`/`crest`/`peak`/`bv`/`mn`/`mx` feed
  `xsound.compute_delivered_loudness` for the `[PLAY]`/`[BASE]` trace and the review player, and `ch_min`/`ch_max`/`h`/`indoor` feed the director's `get_placement`.
- Committed data: `tools/sources.yaml` (the authored gather config: registry, routes, excludes), `tools/manifest.json` (the corpus record), `tools/measure_cache.json`
  (audio-hash-keyed lufs/crest/peak/dur/fingerprint), `ledger.tsv` (the historical full-coverage proof), `provenance.tsv` (origin of every shipped sound, appended per gather).
- The tool is `diegetic-manager` in stalker-dev, and this repo holds data only. `gather DiegeticDread <Source>` is the once-per-pack ingest, and `master DiegeticDread` is the forever path
  (blobs, metadata, exclusion, verify, audit) with no packs required.

A new pack is adopted by hand first. Audition it, author its `sources.yaml` rows (registry entry + route rows), then run `gather DiegeticDread <Source>`.
The coverage proof (UNUSED-DARK = 0) audits the authoring, and the pack is deletable once the committed proof rows land.

## Deploy

A gamedata overlay distributed as a GitHub release and moddb addon. The repo holds the buildable source, the tool, the docs, and the audio.
It is wired for local sync and the gamma-redux install through `stalker-manager`.
