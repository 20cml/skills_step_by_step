---
name: htmltranscript
description: Given a YouTube video link, extract its full transcript, save it as a raw .txt, then build a simplified, flowchart-illustrated HTML explainer styled in the "Editorial Moderno Claro" design system and save it too. Triggers on "/htmltranscript [video link]" or requests to turn a YouTube video's transcript into an HTML/PDF-style explainer document.
---

# HTML Transcript Pipeline

End-to-end, no confirmation pauses, from a YouTube link to two files in the user's
Downloads folder: the raw transcript (`.txt`) and an illustrated HTML explainer
(`.html`). Only stop for the cases in "Failure handling" below.

## Input

Extract the URL from the arguments/message. If no valid `youtube.com`/`youtu.be` URL is
present, ask for one before doing anything else.

## Step 0 — Load tools

Single `ToolSearch` call, not one at a time:

```
select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__find,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__tabs_close_mcp
```

## Step 1 — Open the video and reveal the transcript panel

Automated clicks on "Show transcript" are unreliable — YouTube's `get_transcript`
endpoint intermittently 400s for automation-driven clicks specifically (suspected
bot-detection), even though the identical click works for a real human. So the user
does all the clicking; you only open the tab and take over after.

1. `tabs_context_mcp` with `createIfEmpty: true`, then `navigate` to the video URL.
2. `wait` ~2s, then ask the user (one short message) to click through it themselves:
   expand the description ("...more") and click the blue "Show transcript" button.
   Do not click either of these yourself.
3. Once they confirm, verify:
   ```js
   document.querySelectorAll("ytd-transcript-segment-renderer").length + "|" +
   document.querySelectorAll("transcript-segment-view-model").length
   ```
   If both `0`, ask them to wait a moment or re-click, then check again.

No "Show transcript" button at all → stop, tell the user this video has no
transcript available. Don't fabricate content.

If it still spins after the user clicked, check network requests for `get_transcript`
— a non-200 (seen: 400) confirms the fetch failed; a fresh tab + the user clicking
again is what has worked. A "couldn't determine which page this action targets" error
means the tab group was lost (e.g. user touched the browser) — just re-run
`tabs_context_mcp`, it's not a page-load failure. A freshly opened tab may resume
playback from watch history instead of 0:00 — reset with
`document.querySelector('video').currentTime = 0` (and `.pause()`).

## Step 2 — Extract the transcript

YouTube has shipped two DOM structures for the transcript panel; segments can also be
duplicated in the DOM (real panel + hidden mirror), so dedupe by `timestamp + text`:

```js
let segs = document.querySelectorAll("ytd-transcript-segment-renderer");
let getTime = s => s.querySelector(".segment-timestamp")?.textContent.trim() || "";
let getText = s => s.querySelector(".segment-text")?.textContent.trim() || "";

if (segs.length === 0) {
  // Newer YouTube frontend (seen 2026-09)
  segs = document.querySelectorAll("transcript-segment-view-model");
  getTime = s => s.querySelector(".ytwTranscriptSegmentViewModelTimestamp")?.textContent.trim() || "";
  getText = s => s.querySelector(".ytAttributedStringHost")?.textContent.trim() || "";
}

const seen = new Set();
const lines = [];
for (const s of segs) {
  const time = getTime(s);
  const text = getText(s);
  const key = time + "|" + text;
  if (seen.has(key)) continue;
  seen.add(key);
  lines.push(`[${time}] ${text}`);
}
window.__fullTranscript = lines.join("\n");
window.__fullTranscript.length;
```

If `segs.length` is still `0` (a third DOM structure), probe live tag names instead of
guessing:

```js
[...new Set([...document.querySelectorAll('*')]
  .filter(e => e.tagName.toLowerCase().includes('transcript'))
  .map(e => e.tagName))];
```

Inspect a match's children's `.className` (not raw `outerHTML`, which can be blocked)
to find the new timestamp/text classes, then adapt the extraction.

Sanity check: if `lines.length` is roughly double the expected count, or early lines
repeat later, dedupe is still broken — fix before saving.

Note the transcript's language label (shown under the segment list) — it sets the
language for every downstream artifact.

## Step 3 — Save the raw transcript

1. Get the video title (page `<title>`/H1), sanitize it for filesystem/`file://` safety:
   strip `# / : | ? * < > "`, collapse whitespace, keep spaces/dashes.
2. Download it to the user's Downloads folder as `<Sanitized Title> - transcript.txt`:

```js
const blob = new Blob([window.__fullTranscript], { type: "text/plain" });
const url = URL.createObjectURL(blob);
const a = document.createElement("a");
a.href = url;
a.download = "<Sanitized Video Title> - transcript.txt";
document.body.appendChild(a);
a.click();
a.remove();
```

3. Verify with `Bash`: `ls -la "$HOME/Downloads/<filename>"`. Never `Read` under
   `$HOME/Downloads` (sandbox blocks it) — only touch it via `Bash`.

## Step 4 — Bring the transcript into context

```bash
cp "$HOME/Downloads/<filename>.txt" "<this-session-scratchpad>/transcript.txt"
```

(Scratchpad path comes from the current environment info — never reuse one from a
past session.) Then `Read` it from there.

## Step 5 — Author the HTML explainer

Content rules (settled defaults, don't re-ask):

- **Language**: entire document in the transcript's language. Never translate to
  English by default.
- **Scope**: skip opening small talk/greetings and closing goodbyes; cover only the
  substantive middle content.
- **Reading level**: simple enough for a 12-year-old, no slang, no dumbing down of
  concepts — explain plainly instead of cutting them.
- **Diagrams mandatory**: one inline SVG flowchart per major concept/relationship/
  comparison (no charting library).
- Break up prose with callouts, a small fact-grid, and diagrams — not a wall of text.
- **Close with a ~5-item takeaway summary**, styled as the index-list below.

Design system — **"Editorial Moderno Claro"**, fixed default:

- **Palette**: `--bg:#ffffff`, `--bg-alt:#eef0f1`, `--ink:#121212`,
  `--ink-soft:#57564f`, `--accent:#d7263d` (sparingly), `--line:#d9d8d3`.
- **Type**: Headings in Archivo 800–900, large/tight (`h1 ~ clamp(2.6rem, 8vw, 4.4rem)`,
  negative letter-spacing, line-height ~1.0–1.2). Body Archivo 400, 17–18px, 1.65
  line-height. Labels/captions/numbers in IBM Plex Mono, uppercase, +1px
  letter-spacing, 11–13px. Google Fonts, with system-sans/monospace fallbacks.
- **Layout**: single column, max-width ~740px. Masthead: 3px black top rule + "N° 01"
  edition number. A "Sumário" (TOC) near the top with red mono numbers as anchor links,
  2 columns on wide screens. Each section: its own 3px black top rule + kicker line
  (red mono `N.0X` · gray mono category label).
- **Components**: callouts as pull-quotes (no box, thin red left border, giant
  low-opacity quote mark); fact grids as bordered "ficha técnica" tables (no shadow, no
  rounded corners); closing summary as an index list (red mono numbers, `--line`
  dividers, no boxes); SVG diagrams white bg, ~1.3px black stroke, accent red reserved
  only for the highlighted element.
- **Hard rules**: zero shadow, zero rounded corners, zero gradient, one accent color,
  hierarchy from weight/rules only.

Write with `Write` into the scratchpad first (`<scratchpad>/<Sanitized Title>.html`),
not directly to Downloads.

## Step 6 — Save the HTML to Downloads

```bash
cp "<scratchpad>/<Sanitized Video Title>.html" "$HOME/Downloads/<Sanitized Video Title>.html"
ls -la "$HOME/Downloads/<Sanitized Video Title>.html"
```

Re-running for the same video later: overwrite the same two filenames, don't create
"(1)"-suffixed duplicates.

## Step 7 — Report back

One short message: confirm both files are in Downloads (name each), one-line
description of what the HTML covers. Don't paste transcript or HTML into chat.

## Failure handling

Stop and ask only for:

- No YouTube URL found.
- Video has no transcript/captions available.
- Browser tools unavailable, or a page action keeps failing after 2–3 retries.

Everything else (language, reading level, design, file locations, sanitization,
dedup) is a settled default — apply without asking.
