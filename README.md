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
- Entries live in this browser's IndexedDB for this origin: they don't
  sync across devices or browsers, and clearing this site's data (or
  using a private/incognito window) deletes them.
- `crypto.randomUUID` (used for entry/tag/note IDs) needs HTTPS or
  localhost — both GitHub Pages and `python3 -m http.server` qualify.
  There's a fallback ID generator regardless.

## Recent changes (this update)

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
