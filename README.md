# LOOB — Local Entries Prototype

A single-file, local-first browser prototype for LOOB. Runs Whisper small
entirely client-side via [transformers.js](https://huggingface.co/docs/transformers.js) —
no server, no upload, no network involvement beyond the one-time model
download. Everything below (transcripts, edits, notes, tags) is stored in
this browser's own IndexedDB, on this device, and nowhere else.

## Running it

No build step. Serve the folder over HTTP (opening `index.html` directly as
a `file://` URL breaks ES module imports and microphone access):

```bash
python3 -m http.server 8080
```

Then open `http://localhost:8080` in Chrome or Edge (WebGPU support gives a
real speed boost; it falls back to WASM otherwise). If this is already
hosted via GitHub Pages, just open that URL.

## What's in it

**Capture**
- Record from the mic (unbounded length — chunked into 15s windows under
  the hood) or upload an audio file
- Or write an entry directly as text, no transcription step
- Primary + optional secondary language, for code-switching recordings
  (each ~15s window is tried against both explicitly and the better match
  is kept — see the in-app footnote for the full reasoning and known
  limits of this approach)
- Per-entry Where / When / Life Era / Private-vs-shareable fields, set at
  save time
- A "Download recording (.wav)" button appears right after a mic
  recording — that's the only window to keep the audio, since **saved
  entries never store audio**

**Library**
- Every saved entry: title, transcript, metadata, tags, life era, notes
- Search across titles, transcripts, notes, tags, life era, and location
- Open any entry to edit the title/transcript (the original
  as-transcribed text is always kept alongside your edits, with a
  one-click "Restore original transcript")
- Add/remove notes on an entry, timestamped
- Rule-based tag suggestions (keyword matching across categories like
  work, family, health, relationships, travel, gratitude, loss & grief,
  achievement, conflict, milestone) — click a suggested tag to apply it,
  or add your own custom tags
- Two-press delete (no undo, matching the no-server storage model)
- **AI reflection**, generated on demand per entry by a model running
  locally via [Ollama](https://ollama.com) (e.g. Llama 3.3 or Qwen3) — see
  below. This is the only feature in the app that sends entry data
  anywhere, and it only ever goes to `localhost`, never a third-party API.
- **Myth vs Truth**, a rule-based (non-AI, on-device) scan of the
  transcript for limiting-belief language — see below.

## Myth vs Truth (rule-based, on-device — with an optional Ollama fallback)

Every entry is scanned, sentence by sentence, for four patterns of
limiting-belief language:

- **Self-blame** — "my fault," "I ruined everything," "I'm the problem"
- **Catastrophizing** — "disaster," "the worst," "never going to get better"
- **Should-statements** — "I should have," "I need to," "they should"
- **Absolutist phrasing** — "always," "never," "everyone," "nothing" (checked
  last, since these words often co-occur with the more specific categories
  above — a sentence matching more than one now shows **all** matching
  category badges, not just the first)

Each matching sentence is quoted **verbatim** as the "Myth." Underneath, a
"Truth" is shown — by default a gentle, templated reframe, slot-filled with
the entry's own date, location, and first tag (no model call, just string
substitution). The user can instead click **"Answer this myself instead"**
for a category-specific follow-up question and write their own Truth, which
takes over from the template. A **"Revert to suggested truth"** link
switches back at any time.

**Reducing false positives:** a small negation guard checks the few words
immediately before a matched phrase — "not everyone," "wasn't my fault,"
"isn't always" read as realistic/self-exonerating statements, not the
limiting belief itself, and are no longer flagged.

**False positives that still get through:** click **"Not a myth —
dismiss"** on any flagged sentence. Dismissed items disappear from the
active list and collect under a "N dismissed — show" toggle at the bottom
of the panel, with a **Restore** button per item. Dismissals are keyed to
the sentence's own hash, so they persist across reopening the entry as
long as the sentence itself doesn't change.

**Other languages:** the rule list above is English-only. The **"Also scan
with Ollama (any language)"** button sends the transcript to the same
local Ollama connection used for AI reflection and asks it to find
limiting-belief sentences in whatever language the entry is written in.
Results are merged into the same Myth/Truth list, badged "AI-detected ·
Ollama" to distinguish them from the deterministic rule-based ones. Every
"quote" the model returns is checked against the transcript character-for-
character before being shown — anything that isn't an exact match is
silently dropped, so this can't display a fabricated or paraphrased quote
as if it were the user's own words. This scan is a one-time, opt-in call
(same privacy model as AI reflection: local Ollama only, never a
third-party API), not something re-run automatically on every open.

This whole feature is deterministic pattern-matching plus one optional,
clearly-labeled model call — distinct from the always-network AI
reflection feature above.

A saved Truth or dismissal is keyed to a hash of the exact sentence text,
so it survives reopening the entry as long as the flagged sentence hasn't
changed. If the transcript is edited such that a previously-flagged
sentence changes or is removed, that myth (and any saved Truth or
dismissal for it) simply stops appearing — nothing is deleted from
storage, it just no longer matches on re-scan.

## AI reflection (local Ollama)

Opening a saved entry now shows an "AI reflection" panel with a **Generate
reflection** button. This sends the entry's transcript to a model server
running on your own machine and streams back a short, warm reflective
response — never advice or a diagnosis, and never a point-by-point summary.
Reflections are saved with the entry (timestamped, removable) so you can
build up a small history per entry over time.

**Setup:**
1. Install [Ollama](https://ollama.com).
2. Pull a model once: `ollama pull llama3.3` or `ollama pull qwen3`.
3. Start the server with this page's origin allowed (Ollama blocks
   cross-origin browser requests by default):
   ```bash
   OLLAMA_ORIGINS=http://localhost:8080 ollama serve
   ```
   (Swap the origin if you're serving from a different port or a GitHub
   Pages URL.)
4. In the entry detail panel, confirm the endpoint (defaults to
   `http://localhost:11434`) and pick a model, then hit **Check
   connection** to confirm it's reachable before generating.

The endpoint and model choice are remembered in this browser's
`localStorage` (a device setting, not entry data) so this is a one-time
setup per browser. If the status badge reads "not reachable," it's almost
always the `OLLAMA_ORIGINS` step above — check the browser console for the
exact error.

## Known limitations

- **Not exercised in a live browser session by me** — verified by static
  analysis (HTML structure, JS syntax, unit tests on the tag-rule engine
  and date formatting), but please test a full record → save → edit →
  tag → delete pass yourself before wider sharing.
- Transcription auto-translation avoidance, chunk-boundary dedup, and
  silence-padding fixes are all as before — see in-app footnote for full
  detail on Whisper's known failure modes and how this pipeline works
  around them.
- The rule-based tag engine is intentionally simple keyword matching, not
  inference — it will under-tag and occasionally mis-tag. That's expected
  at this stage, not a bug to chase; richer auto-tagging is scoped as a
  later, MCP-connected upgrade in the Feature Set 3 brief.
- The Private/Shareable toggle is a schema field and a UI signal right
  now, not an enforced permission — nothing in this prototype sends data
  anywhere regardless of its value, **except AI reflection**, which sends
  the transcript to your local Ollama server regardless of the
  private/shareable flag. The toggle doesn't gate this yet — a natural
  next step is to disable the reflect button (or warn) on entries marked
  private, if that distinction should matter even for a local-only call.
- AI reflection has no timeout or retry: a slow or wedged local model will
  leave the button on "Generating…" until the request resolves or the
  browser gives up. There's also no token/length cap on the model's
  response beyond whatever the model itself defaults to.
- Myth vs Truth's rule-based pass is regex/keyword matching and English-only
  by design — that's what the Ollama fallback scan is for. It still won't
  catch every limiting belief even in English (anything outside the
  keyword list), and the negation guard is a simple 3-word lookback, not
  real grammar parsing, so unusual phrasing can still slip past it in
  either direction (missed or over-flagged). Expected of a first-pass
  keyword system, not a bug to chase.
- The Ollama myth-scan fallback depends on the model actually returning
  valid JSON with verbatim quotes; weaker or more heavily quantized models
  may paraphrase instead of quoting exactly, in which case the verbatim
  check silently drops that item rather than showing an inaccurate quote.
  Larger/more capable models will follow the instruction more reliably.
- Entries live in this browser's IndexedDB for this origin: they don't
  sync across devices or browsers, and clearing this site's data (or
  using a private/incognito window) deletes them.
- `crypto.randomUUID` (used for entry/tag/note IDs) needs HTTPS or
  localhost — both GitHub Pages and `python3 -m http.server` qualify.
  There's a fallback ID generator regardless.

## Recent changes (this update)

- Myth vs Truth: added multi-category badges per sentence (a sentence can
  now show more than one matched category instead of only the first), a
  negation guard to cut false positives like "not everyone" / "wasn't my
  fault," a "Not a myth — dismiss" action with a collapsible dismissed
  list and per-item Restore, and an opt-in "Also scan with Ollama (any
  language)" fallback that reuses the AI reflection connection to catch
  limiting-belief language in languages the English rule list can't,
  with every returned quote checked verbatim against the transcript
  before being shown.
- Added Myth vs Truth: rule-based (non-AI) detection of absolutist
  phrasing, self-blame, catastrophizing, and should-statements, quoting
  each match verbatim as a "Myth." Truth is templated by default
  (slot-filled with the entry's date/where/tags) or user-elicited via a
  category-specific follow-up question, with the user's own answer saved
  and take precedence over the template until reverted.
- Added on-demand AI reflection per entry, calling a locally-run Ollama
  model (Llama 3.3 / Qwen3 / custom) over a streaming chat API. Reflections
  are saved with the entry, timestamped and removable. Endpoint/model
  settings persist in `localStorage`.
- Added saved entries backed by IndexedDB (fully on-device), a Library
  view with search, per-entry transcript editing with restore-to-original,
  and timestamped notes.
- Added a one-time raw-audio WAV download offered immediately after each
  recording — audio is never stored with entries.
- Added text entry as a third input path alongside record/upload.
- Added Where / When / Life Era fields per entry, editable after saving.
- Added a per-entry Private/Shareable flag, private by default.
- Added rule-based tag suggestions plus custom tags, with search now
  covering tags, life era, and location too.

## Files

```
index.html   Everything — UI, styling, and the full app logic in one file
```
