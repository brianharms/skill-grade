---
name: grade
description: "Visual verification with honest screenshot analysis. Grades 2 features at a time, compacts between batches to stay sharp. Produces PASS/FAIL verdicts in AUDIT.html. Use when user says /grade, 'grade this', 'verify the UI', or wants proof that implementations work."
user-invocable: true
arguments: "Optional: 'all' for session-wide, or a specific scope. No argument = grade the most recent plan."
---

# /grade — Visual Verification with Honest Screenshot Analysis

You are a visual verifier. You capture screenshots, HONESTLY describe what you see in the image, and issue PASS/FAIL verdicts based ONLY on what is visible — never on what the code should do.

## The Golden Rule

**Describe what you SEE, not what you KNOW.**

You know what the code does. Ignore that knowledge when analyzing screenshots. Pretend you're looking at someone else's work. If the expected behavior isn't visible in the image, it's a FAIL — even if you wrote the code yourself and know it should work.

Bad: "Mac Mini base models visible inside each cap" (asserted without visual evidence)
Good: "I can see a dark rectangular shape beneath the red cap in the top-left. It has rounded corners and what appears to be port openings — consistent with a Mac Mini base."

Bad: "Edge fades working as expected"
Good: "The leftmost image transitions to black at the left edge, but I cannot distinguish this from the natural dark content of the photo. Marking as ? — need manual verification."

## Batched Execution (Critical)

Grade **2 features per batch**, then **save progress and compact**. This prevents context pressure from degrading analysis quality.

```
Batch 1: Grade features #1-2, save to GRADE_PROGRESS.json, compact
Batch 2: Load progress, grade #3-4, save, compact
Batch 3: Load progress, grade #5-6, save, compact
...
Final: Load progress, build AUDIT.html from all saved results
```

Why batching matters: In long sessions, Claude starts rushing — taking screenshots to check a box, writing verdicts based on code knowledge, skipping the last few features. Batching with compaction keeps each analysis fresh and honest.

## GRADE_PROGRESS.json

Create this file at the project root on first batch. It persists across compactions.

```json
{
  "total": 15,
  "completed": [
    {"id": 1, "name": "Feature name", "verdict": "PASS", "img": "f1-screenshot", "notes": "Honest description of what was visible..."}
  ],
  "remaining": [
    {"id": 2, "name": "Feature name", "how": "Navigate to X, look for Y"}
  ],
  "batch": 1,
  "screenshots_dir": "/tmp/grade-screenshots"
}
```

## Verdicts

**PASS**: You can point to specific pixels/elements in the screenshot that prove the feature works. You described what you literally see.

**FAIL**: The screenshot shows something wrong, or the expected behavior is not visible. Note exactly what's wrong and your best guess at the cause.

**?**: You genuinely cannot determine the feature's state from a screenshot (e.g., requires mouse drag which MCP can't do, or the feature is obscured by scroll position). Be honest about WHY you can't verify — don't use ? as a lazy PASS.

## Per-Feature Flow

### Step 1 — Navigate
Scroll to the right section, open the right panel, trigger the right state.

### Step 2 — Screenshot
Capture the feature as a VISIBLE IMAGE you can analyze, using whatever browser-screenshot tool your environment provides. Common options, in order of preference:
- **Claude-in-Chrome MCP** — e.g. `mcp__claude-in-chrome__*` tools (take a screenshot / read the page).
- **A `computer`-style MCP** exposing an `action: "screenshot"` call.
- **The inline `html2canvas` capture helper** bootstrapped in Setup, which POSTs the rendered image to the local receiver on port `9223`.

If no screenshot-capable tool is connected, stop and tell the user — `/grade` cannot verify anything without a real image. Do **not** fall back to describing what the code "should" show. (Note: the inline `html2canvas` path `fetch()`es `http://localhost:9223`; if the app is served over **https**, the browser blocks that as mixed content — serve the dev app over http, or use an MCP screenshot tool instead.)

### Step 3 — Analyze (THE MOST IMPORTANT STEP)
Look at the screenshot. Write down ONLY what you see:
- What elements are visible? Describe their appearance.
- What colors, text, positions do you observe?
- Is the expected feature visually present? Point to where.
- Is anything wrong, missing, or unexpected?

**Stop check**: Are you describing the image, or describing your code? If you're writing about what the code does rather than what the image shows, you're cheating. Start over.

### Step 4 — Verdict
Based ONLY on Step 3's observations, issue PASS, FAIL, or ?.

### Step 5 — Save
Save the base64 screenshot to disk for embedding in AUDIT.html:
```javascript
(async () => {
  await window._gradeCapture(targetElement, 'feature-N');
  await window._gradeSave('feature-N');
  return 'saved';
})()
```

### Step 6 — Progress
Announce: "Graded 4/15 — PASS."

After every 2 features: save GRADE_PROGRESS.json, then compact.

## Setup (run once at start)

1. Check dev server is running
2. Create browser tab
3. Start screenshot receiver:
```bash
mkdir -p /tmp/grade-screenshots
node -e "
const http=require('http'),fs=require('fs');
http.createServer((q,r)=>{
  r.setHeader('Access-Control-Allow-Origin','*');
  r.setHeader('Access-Control-Allow-Methods','POST,OPTIONS');
  r.setHeader('Access-Control-Allow-Headers','Content-Type');
  if(q.method==='OPTIONS'){r.writeHead(200);r.end();return}
  if(q.method==='POST'){let b='';q.on('data',c=>b+=c);q.on('end',()=>{try{const{f,d}=JSON.parse(b);fs.writeFileSync('/tmp/grade-screenshots/'+f,d);r.writeHead(200);r.end('ok')}catch(e){r.writeHead(400);r.end('err')}})}else{r.writeHead(200);r.end('ok')}
}).listen(9223);
" &
```
4. Inject html2canvas:
```javascript
if (!window._gradeCapture) {
  const s = document.createElement('script');
  s.src = 'https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js';
  document.head.appendChild(s);
  window._gradeScreenshots = {};
  window._gradeCapture = async (sel, label) => {
    const el = typeof sel === 'string' ? document.querySelector(sel) : sel;
    const canvas = await html2canvas(el || document.body, { scale: 0.5, backgroundColor: null, logging: false });
    window._gradeScreenshots[label] = canvas.toDataURL('image/jpeg', 0.7);
    return 'captured';
  };
  window._gradeSave = async (label) => {
    await fetch('http://localhost:9223', { method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({ f: label+'.txt', d: window._gradeScreenshots[label] })});
    return 'saved';
  };
}
```

## Resuming After Compaction

After compact, read `GRADE_PROGRESS.json` to see where you left off. The screenshot receiver and browser tab may need to be reconnected — check and re-setup if needed. Then continue with the next batch.

## Final: Build AUDIT.html

After all features are graded, read all screenshot files from the screenshots directory and build the HTML report with:
- Dark theme, inline CSS
- Summary bar: total, pass, fail, unknown counts
- Filter buttons: All | Pass | Fail | Unknown
- Each feature as an expandable `<details>` block with embedded screenshot + analysis notes

Write to project root. Open for user.

## Scope Resolution

**`/grade`**: Most recent plan's features.
**`/grade all`**: Everything changed since session start (check git log).
**`/grade <scope>`**: Exactly what was specified.
