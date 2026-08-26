# Imagineverse — Project Handoff

Local AI roleplay / visual-novel engine. Modeled on imaginarium.rocks
after reverse-engineering it, but with a deliberately different rendering
architecture. Runs entirely local on an M3 MacBook Pro, 64GB, via Ollama.

**Status: v0 works.** Text-only. Characters talk, you can write their
lines or generate them. No images yet, by design.

---

## Environment

- **Machine:** M3 MacBook Pro, 64GB unified memory
- **Model:** `qwen3.8:27b-mlx` via Ollama (18GB). Also installed:
  `qwen2.5:14b-instruct`, `qwen2.5:7b-instruct`, `mistral-nemo:12b`,
  `qwen3:8b`, `qwen3-coder:latest`
- **Code lives at:** `~/Documents/Imagineverse`
- **Python 3.14**, stdlib only — no pip dependencies
- **Also available:** a loaned NVIDIA GB10 Grace-Blackwell workstation,
  currently used for a separate Chatterbox TTS voice-cloning pipeline.
  Relevant later — sprite batch generation and per-character TTS.

## Running it

```bash
cd ~/Documents/Imagineverse
export IMAGINARIUM_CTX=16384

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
silently truncates — a long transcript loses its head with no error.

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

---

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

Two rules that matter more than they look:

- **Skip on no-signal.** Gate the classifier on the presence of action
  markup at all. Most turns are pure dialogue.
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

1. **Play sessions and judge voice quality.** This is what v0 is for.
   Run `/auto 20`, cover the names, ask whether you can tell who's
   speaking. If not, fix `CHARACTER_SYSTEM` in `creation.py` — push
   harder on incompatible wants and specific avoidance behaviors. No
   downstream architecture repairs flat personas.
2. **Watch action inflation.** If every line carries a gesture tag,
   that's model padding, and it makes the transient/postural distinction
   load-bearing sooner.
3. **`/export` good transcripts.** They become the test fixtures for the
   Stage Manager — real action tags from real play beats invented cases.
4. **Build the Stage Manager against those fixtures**, standalone. Feed
   it transcript lines, check the state tuples. Get it solid before any
   image code exists.
5. **Then** the ComfyUI sprite pipeline and anchor derivation.

## Open questions

- Does a 27B MoE (~3B active) hold character voice well enough, or does
  this need a dense model? If personas are good and voices still blur,
  that's the next variable — try `qwen2.5:14b-instruct` dense as a
  control, or a dense 27–32B.
- Temperature sensitivity: `/temp 1.0` vs `/temp 0.6` on the same scene.
  Divergence between characters usually varies more than expected.
- Cutout sprites vs scene-integrated staging for seated poses —
  currently resolved as "cutouts + anchors + contact shadow," but worth
  revisiting if seated poses look wrong in practice.
