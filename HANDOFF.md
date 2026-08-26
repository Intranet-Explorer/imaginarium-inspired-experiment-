# Imagineverse — Project Handoff

Local AI roleplay / visual-novel engine. Modeled on imaginarium.rocks
after reverse-engineering it, but with a deliberately different rendering
architecture. Runs entirely local on an M3 MacBook Pro, 64GB, via Ollama.

**Status: v0 works, and the first voice test has been run.** Text-only.
Characters talk, you can write their lines or generate them. No images yet,
by design. The first `/auto 20` exposed a convergence failure that has since
been addressed in the creation layer — see *What the first voice test showed*
below before changing anything in `creation.py` or `play.py`.

---

## Environment

- **Machine:** M3 MacBook Pro, 64GB unified memory
- **Model:** `qwen3.8:27b-mlx` via Ollama (18GB). Also installed:
  `qwen2.5:14b-instruct`, `qwen2.5:7b-instruct`, `mistral-nemo:12b`,
  `qwen3:8b`, `qwen3-coder:latest`
- **Code lives at:** `~/Documents/imaginarium` (git; private remote
  `Intranet-Explorer/imaginarium`)
- **Python 3.14**, stdlib only — no pip dependencies
- **Also available:** a loaned NVIDIA GB10 Grace-Blackwell workstation,
  currently used for a separate Chatterbox TTS voice-cloning pipeline.
  Relevant later — sprite batch generation and per-character TTS.

## Running it

```bash
cd ~/Documents/imaginarium
export IMAGINARIUM_CTX=16384

python3 test_offline.py    # schema, prompts, summary, loop-breaker — no model
python3 test_stream.py     # stream_line buffering — no model

python3 cli.py models                                   # smoke test
python3 cli.py char new --world "Between the Stations"  # twice
python3 cli.py loc new --world "Between the Stations"
python3 cli.py session new --world "Between the Stations"
python3 cli.py play 1
```

In the REPL: `/auto 8`, `/ai <name>`, `/n <text>`, `<name> <text>` to
write a line yourself, `/add`, `/drop`, `/undo`, `/t`, `/temp`, `/model`,
`/export`, `/?`.

---

## Files

| File | Purpose |
|---|---|
| `schema.sql` | world, character, outfit, location, session, participant, turn |
| `db.py` | SQLite access layer |
| `llm.py` | Ollama client — streaming, model validation, reasoning suppression |
| `creation.py` | Character/location generation from one description → JSON |
| `play.py` | Prompt assembly, turn generation, terminal rendering |
| `cli.py` | Subcommands + interactive play REPL |
| `test_offline.py` | Migration, relationships, prompt assembly, summary, loop-breaker — stubbed llm, no Ollama |
| `test_stream.py` | `stream_line` buffering against scripted events |
| `README.md` | Setup and design notes |

---

## Design decisions that are load-bearing

**Prompt block order: system → transcript → persona.** The persona sits
LAST. Ollama's runner prefix-matches against the previous request's KV
cache, so alternating speakers only re-prefills the short persona tail
instead of the whole context. Reordering these blocks silently costs a
full re-prefill every turn. This is the thing most likely to get broken
by a well-meaning refactor.

**`bio` and `persona_prompt` are separate fields.** Bio is prose the user
reads. Persona is second-person behavioral instruction: what they want,
what they conceal, how they act under pressure. Bios written for reading
make bland personas. If characters sound alike, this field is the cause.

**Markup contract, enforced from turn one:**
```
Speaker: <action> dialogue text
```
Angle brackets for actions, `*bold*`, `_italic_`. `clean_line()` rewrites
asterisk-actions into angle brackets and strips a leading `Name:` if the
model adds one. Keep strict — the Stage Manager parses this later.

**Visual fields are written but unread.** `appearance`,
`prompt_fragment`, `renderer`, `style_tags` cost one LLM call now and a
migration later.

**`IMAGINARIUM_CTX` matters.** Ollama defaults to 4096 tokens and
silently truncates — a long transcript loses its head with no error. The
rolling summary now caps growth, but the ceiling still has to be set.

**Characters are never generated in isolation.** `make_character` takes the
existing cast and must produce someone who collides with one of them.
Relationships are a first-class table: each pair gets two directional rows
(`wants` / `withholds` per side, symmetric `history` / `friction`). Generating
a character alone produces someone written against an implied absent human,
who then has nothing to want from the people actually in the room.

**Actions earn their place.** The rule is no longer a quota ("most lines
should have none" — unenforceable mid-generation) but a test the model can
apply per line: include an action only when the body does something the words
do not say. `<looks away> I'm fine.` keeps; `<smiles> That's funny.` cuts.

**The transcript is sent as a rolling summary plus the last `WINDOW` turns.**
Not the whole log. This caps context and, more importantly, stops the
transcript out-weighing the persona tail by an ever-growing margin — which is
what drives voice convergence. The summary sits at the top of the user block
and only changes every `SUMMARIZE_EVERY` turns, so the cache still holds most
turns.

---

## Bugs already found and fixed (don't reintroduce)

1. **Server-side `stop: ["\n"]`** — models open their turn with a
   newline, so generation ended before one visible character arrived.
   Now buffers client-side and cuts at the first newline *after* content.
2. **Fuzzy model-name matching in `load_model`** — let a wrong tag pass
   validation, then 404 at generation with a bare stack trace. Now exact
   match, or a bare stem resolving to exactly one tag.
3. **`/no_think` prompt injection** — wrong approach for this Ollama
   build. Reasoning arrives as a separate `thinking` JSON field; the
   correct control is `"think": false` in the request body. Falls back to
   the prompt directive only if the server rejects the parameter.
4. **Session picker renumbered after each selection** — dropped the
   second character silently. Now stable numbering with ✓ toggles.
5. **Generation errors crashed the whole REPL** — now caught, printed in
   red, session continues.
6. **`/model <tag>` discarded the resolved name** — validated a bare stem via
   `load_model`, then set `MODEL` to the stem instead of the full tag it had
   resolved. Passed validation, 404'd at generation. Same shape as bug 2,
   through a different door. `cmd_play`'s startup always did this correctly.
7. **`clean_line` ate leading emphasis** — `*Never* again.` became
   `<Never> again.`, fabricating an action tag. Since transcripts are the
   Stage Manager's training corpus, a false tag is corpus poison. The rewrite
   now requires two words and a lowercase opening verb.
8. **`stream_line` tracked emitted *length*, not text** — `_visible()` could
   return a shorter string than the previous event (a `<think>` opening
   collapsed it to `""`), leaving the counter stranded above it and silently
   swallowing the real line. `_visible` is now monotonic: it strips think
   blocks while keeping text on both sides.

---

## What the first voice test showed

`/auto 20`, session 1, two characters both described as robots.

**The test was confounded** — both source descriptions were machines, so the
run cannot separate *the engine blurs voices* from *these two were specified
as the same character*. It has to be rerun with two people who want
incompatible things.

**The failure mode is template lock, not voice blur.** All ten of Pip's turns
opened `I observe ...`; all ten of Echo's opened `Analyzing:` or `Logging:`.
Locked by turn 2, never broken, at temp 0.85. `I observe` was not even in
Pip's tics list — it emerged. The characters were trivially distinguishable
and neither was a character. Covering the names and answering "yes, I can
tell them apart" would have been the wrong conclusion: the differentiation
was purely mechanical.

**The personas were good and still did not save it.** Both were specific and
genuinely opposed in want. Two reasons they failed anyway:

1. `CHARACTER_SYSTEM`'s free "how they behave under pressure" field produced
   the *same stock answer* in both — freeze and repeat the last input in a
   loop. Turns 3–5 are literally that loop, both characters executing their
   personas correctly, in unison. Correlated personas, not a sampler problem.
2. Pip's richest material was all about its "primary companion", a human not
   in the scene. Its one line covering the actual situation said it treats
   other robots with *clinical indifference*. A persona written alone is
   written against an absent partner.

**Tic bleed takes two exposures.** Echo's `Logging:/Requesting:` grammar was
in Pip's mouth by turn 4. Action tags mirrored too — turn 5's
`<tilts head, optical sensors narrowing slightly>` is turn 4's
`<tilts head, optical lenses narrowing slightly>` with one word changed.

**Action inflation was 100%, not partial.** 20 of 20 turns carried a tag; 19
of 20 were compound (2–3 clauses). Rough classification: ~14 transient,
~4 positional, ~2 affective, ~1 postural.

**What worked:** the markup contract held perfectly. Zero leading-name leaks,
zero asterisk-actions, zero multi-line output, no narrator confusion.

## What we learned reverse-engineering imaginarium.rocks

Observed from screenshots and fetches. Relevant because it's the design
being deliberately diverged from.

- Auth-gated SPA, solo dev, static host + separate asset CDN, aggressive
  service-worker caching (has a hand-written stale-cache apology page)
- Data model is a **world** containing Characters, Locations, Sessions as
  separate reusable entities — not a chat log. We copied this.
- Character creation: one description → LLM writes bio, outfits, and an
  avatar prompt. We copied this.
- Markup format `Speaker: <action> dialogue` with a `Narrator:` line as a
  separate scene-state channel. We copied this.
- "Renderer: Anima" + Danbooru artist tags (`@kantoku`, `@fugtrup`) in
  the Style field ⇒ an Illustrious/NoobAI-family SDXL checkpoint.
  Renderer = checkpoint enum, Style = prompt prefix.
- **The key divergence:** their cinematic view generates the *entire
  scene* — characters included — as one T2I image per visual state. That
  is why it's slow, why consistency drifts, and why it needs a reroll
  button. We are not doing this.
- Turn control: "AI Turn" advances the other side, "Write for me"
  ghostwrites the line for the character you hold, "Unsend" rewinds. The
  user occupies a slot and can delegate it — better than an autonomous
  two-agent loop. We copied this.

---

## The planned architecture (designed, not yet built)

### Rendering: composited sprites, not full-scene generation

Pre-render transparent character sprites and static backgrounds; layer
them in the browser. Turn latency → zero. Consistency is perfect by
construction rather than by conditioning. Cost model inverts: pay per
asset, not per turn.

This is forced by the hardware anyway — SDXL on MPS runs ~1.5–2.5
s/iteration, so even at 6 steps with a Lightning/DMD2 LoRA you're 10–20s
per image. Disqualifying per-turn, fine as an overnight batch job.

**Run the image worker as a separate process behind a queue.** MLX and
PyTorch-MPS share the same unified memory pool with no partitioning; a
diffusion batch mid-conversation will stall token generation.

### Anchors solve the floating-sprite problem

Sprites are generated per **pose class** ("seated, chair height,
three-quarter view"), not per location. Each location gets 2–3 anchor
slots supplying position, scale, and facing:

```
location: closed_office
  anchors:
    - id: desk_near, pose_class: seated,   x: 0.32, y: 0.61, scale: 1.00, flip: false
    - id: desk_far,  pose_class: seated,   x: 0.68, y: 0.58, scale: 0.94, flip: true
    - id: doorway,   pose_class: standing, x: 0.12, y: 0.55, scale: 1.05, flip: false
```

Library stays N poses, not N poses × L locations.

Two things make it work: **a fixed camera contract** (same framing, eye
level, and distance in every sprite prompt — locked, never varied) and a
**contact shadow ellipse** under each sprite. The shadow is trivial CSS
and is the single biggest contributor to "placed" vs "floating".

Anchors are derived at location-creation time by grounding a VLM
(Florence-2 or Qwen2-VL) on the *generated image* — not the prompt, since
the diffusion model doesn't put furniture where the description said.
Anchor point = bottom-center of the detected bbox. Pose class from the
object label. Scale from perspective:

```
scale(y) = (y - horizon_y) / (y_ref - horizon_y)
```

`flip = anchor.x > 0.5` by default. Followed by a 15-second draggable
human correction pass. Keep an `anchor.source` column (`detected` /
`manual`) so re-running detection doesn't stomp human fixes.

**Findings from the compositing harness** (Stage Plot — a browser page that
runs the anchor maths with placeholder art, no models and no GPU):

- **The example anchors above do not satisfy the scale formula.** With
  `horizon_y 0.38` and `y_ref 0.61` the denominator is 0.230, so `desk_far`
  computes 0.870 against a stored 0.94, and `doorway` computes 0.739 against
  a stored 1.05 — a figure that clips out of frame. The doorway is *further
  from camera* than the near desk and stored *larger*. Validate detected
  scales against the ground plane before writing them.
- **Bottom-centre of the bbox is not the ground contact for a seated pose.**
  It is fine standing. Seated, the legs project forward and the feet land
  around 0.75 of the sprite's width, so centring the bbox on the anchor puts
  the body a quarter of a sprite-width off its own shadow. Store a
  `contact_x` per pose class alongside the anchor.
- **Depth ordering is a third thing.** A desk the character sits behind has
  to draw *over* the sprite, and a sprite cannot express "the desk is in
  front of me". Each location needs a foreground plate and a rule for which
  anchors sit behind it.
- `camera_contract` is now a stored column on `location`, not a convention to
  remember. Vary it and every anchor for that location becomes invalid.

Resist anchor creep — 2–3 staging positions per location, like a stage
play.

### Stage Manager: constrained decode over sprite inventory

Do NOT ask the model to describe the scene and then map its description
onto sprites. Build a GBNF grammar **from the sprite table** at turn time
and constrain decoding to it. Invalid states become structurally
impossible. A 3–4B model is plenty — it's classification.

First job is classifying the action, because **most actions don't change
visual state**:

- **transient** — gestures, glances, fidgets. No change. The majority.
- **postural** — sits/stands/leans/turns. Sprite swap.
- **wardrobe** — removes, adjusts, puts on. Outfit swap.
- **positional** — moves elsewhere. Background swap.
- **affective** — expression only. Cheap, safe to be liberal with.

`<taps one manicured nail against the desk>` is transient.
`<uncrosses her legs, the movement deliberate>` is postural.

**Corrections from the first run — read before building this:**

- **"Skip on no-signal" does not fire.** It assumed most turns are pure
  dialogue. Zero of twenty were. Recost the classifier as running on every
  turn. The gate is still worth keeping for the case where the new format
  rules actually reduce tagging, but do not budget for it.
- **Compound actions are the norm, not the exception** — 19 of 20 carried two
  or three clauses, and they cross categories:
  `<stands, shrugging out of her coat>` is postural *and* wardrobe.
  A grammar that forces one label per turn discards half the state change.
  Emit a set, and define which axis wins when two fire at once.
- **`positional` conflates two different renders.** `<drifts closer to Echo>`
  is an *anchor change* within the location; `<walks out>` is a *location
  change*. The first is a sprite reposition, the second a background swap.
  Four anchor-changes and zero location-changes in twenty turns — so the
  common case is the one the current definition would get wrong.
- **The Narrator channel is unhandled.** All five categories describe a
  character's body. `Narrator: the platform lights cut out` is a scene-state
  change with no speaker and no anchor, and it is the most likely trigger for
  a background or lighting change. It currently falls straight through.

Two rules that matter more than they look:

- **Skip on no-signal.** Gate the classifier on the presence of action
  markup at all.
- **Sticky state with different volatilities.** Pose and outfit persist
  until explicitly changed; expression moves freely; location changes
  only on explicit transition. Without this you get sprite flicker —
  standing, sitting, standing across three lines — which is worse than no
  animation.

Enforce outside the grammar: a postural change must land on an anchor
whose `pose_class` matches the new pose.

### Sprite library: seeded, then grows

Outfits are bounded (enumerated at character creation). Poses and
expressions are not.

1. At character creation, batch-generate 4–6 poses × 4 expressions per
   outfit.
2. At play time, look up the state tuple.
3. **Hit** → swap instantly.
4. **Miss** → show nearest neighbor immediately, queue the real render,
   hot-swap when it lands. New sprite joins the library permanently.

Nearest-neighbor distance: expression substitution is cheap, pose
substitution is expensive. Prefer swapping expression.

**Generate expression variants by inpainting the face region** of an
existing pose sprite — same seed, masked to the head, low denoise. ~3–4×
faster, and guarantees the body is pixel-identical across expressions. A
body that subtly shifts when only the face should have changed is very
visible.

### Animation

Not video generation. Ken Burns drift, parallax layers, particle
overlays, crossfades — client-side, free. Keep full-scene T2I as a
deliberate 🎬 action: user asks for key art of the current moment, one
good image, ~15s, cached in the session.

### Later

- Per-character TTS via the existing Chatterbox pipeline (uses the GB10)
- Memory consolidation between sessions ("Let them sleep on it" in the
  original) — rolling summarization into a per-character fact store

---

## Immediate next steps, in order

1. **Rerun the twenty with two humans who want incompatible things.** The
   creation layer now demands a collision and writes the relationship, so
   this is the first honest run. Judge it on: do they disagree, does anyone
   change position, and is the action rate below 100%.
2. **Then run twenty where you write one side.** That is the actual UX — the
   turn-control model has the user occupying a slot. `/auto` runs both sides
   autonomously, which is a mode the design never targeted and a harsher test
   than the product will face. Do not conclude the product is broken from an
   autoplay result.
3. **`/export` the good transcripts.** They are the Stage Manager's fixtures.
   Real action tags from real play beat invented cases — and the first run
   already proves invented cases would have been wrong about the skip rate.
4. **Build the Stage Manager against those fixtures**, standalone, with the
   four corrections above applied.
5. **Then** the ComfyUI sprite pipeline and anchor derivation.

## Open questions

- **Does the model hold voice once the personas actually differ?** The first
  run could not answer this — the characters were specified nearly
  identically and both locked onto templates. Unresolved until step 1 above.
- Does a 27B MoE (~3B active) hold character voice well enough, or does this
  need a dense model? Only worth testing after a clean run. If personas
  collide properly and voices still blur, try `qwen2.5:14b-instruct` dense as
  a control, or a dense 27–32B.
- Temperature sensitivity: `/temp 1.0` vs `/temp 0.6` on the same scene. Note
  that template lock survived 0.85, so temperature alone is unlikely to be
  the lever.
- **Does the rolling summary cost more than it saves?** It caps context and
  weakens the transcript's grip on voice, but a summary is lossy and the
  characters lose access to their own exact words. Watch for callbacks that
  stop landing. `IMAGINARIUM_WINDOW=999` disables it in practice.
- **Does the anti-mirror resample fire often enough to matter, or too often?**
  It triggers on a third identical opening. Instrumentation would tell us
  whether the new format rules already prevent the lock without it.
- Cutout sprites vs scene-integrated staging for seated poses — the harness
  says cutouts work provided `contact_x` is per-pose and there is a
  foreground plate. Worth revisiting against real diffusion output, which
  will not place feet as obligingly as placeholder art.
