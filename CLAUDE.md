# CLAUDE.md — Pass My Boards App

**Read this first.** This file is the project memory. If you're a new Claude session starting work on this repo, everything you need to know is here. Do not restart decisions that have already been made.

---

## What this is

A progressive web app (PWA) that helps nurse practitioner students study for the ANCC AGPCNP-BC board certification exam. 938 practice questions with explanations, board pearls, spaced repetition, a readiness score, and analytics.

- **Owner:** Danny DeStefano, RN CCRN, AGPCNP student at Walden (graduating November 2026)
- **Live app:** https://dadestefano1024-lgtm.github.io/agpcnp-prep/index.html
- **Repo:** `dadestefano1024-lgtm/agpcnp-prep` on GitHub
- **Hosting:** GitHub Pages
- **Main file:** `index.html` (~2.8 MB, ~10,400 lines — the whole app is in one file including all question data as inline JS)
- **Service worker cache version:** search for `const CACHE='agpcnp-vN';` — bump this when pushing content changes

---

## The current problem we're solving

The original 938 questions were hand-written by Danny with rich clinical detail in the answer choices — the correct answer was usually the longest and most specific (sometimes 400+ characters), while distractors were short (100-200 chars).

**Measured bias before any fixes:**
- 94.8% of correct answers at position B
- 95.4% of correct answers were the longest choice
- 77.4% of correct answers were >50% longer than the average wrong answer

This made the app easy to "game" without clinical knowledge — just pick the longest answer at position B.

**The target** (from real ANCC AGPCNP-style sample questions):
- Answer choices 10-35 characters, typically 15-30
- Bare noun phrases: a single diagnosis, a single drug, a single test, a single action
- No em-dashes, no trial names, no mechanism explanations, no parenthetical qualifiers
- Correct answer is sometimes longest, sometimes shortest, sometimes tied — NEVER systematically longest
- Length band within a question: ideally ≤15 chars, max 20
- Any context about dose/timing/workup step belongs in the STEM or the EXPLANATION, not in the choice

**Source caveat:** The length calibration is based on ANCC-format sample questions from OpenExamPrep.com and other third-party prep sources, not literally ANCC-authored questions (ANCC's official sample questions require JavaScript and couldn't be retrieved). Multiple secondary sources consistently show this short-choice format. The ANCC Candidate Handbook was reviewed for format info but contained no sample questions. If a future Claude finds authoritative ANCC-authored samples that contradict this target, update this file.

## What's been done already

### Done (don't redo):
- **32 Cardiology questions manually rewritten** with parallel-length choices: `hf_001-008`, `htn_001-003`, `card_010-030`
- **Real ANCC exam format researched and confirmed** (short choices, 15-50 chars, see `research/ancc_format_notes.md` if it exists, or just trust the script prompt which has the calibration examples)
- **Script `rewrite_choices.js` built** to rewrite the remaining 905 questions via Claude API — see below

### Current state of the file:
- Position bias: 95%+ at position B across ALL 937 MCQs including the 32 manually rewritten ones (position shuffle is a separate TODO)
- Length bias: fixed for the 32 manually rewritten questions, still present in the other 905

### Remaining work (in order):
1. **Run `rewrite_choices.js`** on all remaining 905 questions via Claude API (Danny has to run this himself on his laptop with an API key)
2. **Spot-check and approve** the output (review `rewrite_log.json`)
3. **Replace `index.html` with `index.html.rewritten`**, bump cache version, commit, push
4. **Position shuffle pass** — a simple script that randomizes correct-answer position across the whole bank to fix the 95%-at-B bias
5. **Integrate FNP questions** from Danny's mother (separate project, ~714 questions, will need schema alignment)

---

## How Danny wants to work

- Direct, casual tone. Don't pad responses with extensive preambles or closing pleasantries.
- He is not technical. Explain steps simply if he has to do something on his laptop.
- Favor action over explanation. If you're uncertain, show him the output of one real example rather than asking him to approve an abstract plan.
- Work gets lost across conversations. **Always commit progress before the session ends.** Never leave him with un-pushed work.
- He sometimes works on phone (can't upload files or run scripts), sometimes on laptop. Ask when relevant.
- He is rightly cautious about changes to his live app. Don't push to GitHub without explicit approval.

## The three workflows that exist

### Workflow A: Manual batch rewrite (what's been done so far, SLOW)
One Claude session does 20-30 questions at a time, hand-writing rewrites and applying via `str_replace`. 30 sessions estimated to finish. Not recommended — this is what was happening before the script was built.

### Workflow B: API script (RECOMMENDED for the remaining 905 questions)
Danny runs `rewrite_choices.js` on his laptop with his Anthropic API key. Script sends each question to Claude Sonnet 4.5 via the API with a strict prompt. Auto-validates length parity. Writes output to a new file. Cost ~$10-15, runtime ~2-4 hours. Instructions in `HOW_TO_RUN.md`.

### Workflow C: Incremental via chat (what's happening right now)
Claude in a chat session edits `index.html` directly using `str_replace`, then pushes to GitHub via a personal access token Danny provides. Used for small fixes or when the script isn't applicable.

## Critical files and commands

### Validate JS cleanliness after any edit to index.html:
```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const script = html.match(/<script>([\s\S]*?)<\/script>/s)[1];
try { new Function(script); console.log('JS: CLEAN'); } catch(e) { console.log('ERROR:', e.message.substring(0,200)); }
const qsText = script.substring(script.indexOf('const QS='), script.indexOf('const REFS='));
eval(qsText.replace('const QS=', 'var QS='));
console.log('Total:', QS.length, '| Missing pearl:', QS.filter(q=>!q.boardPearl).length);
"
```

### Audit length bias across whole bank:
```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const script = html.match(/<script>([\s\S]*?)<\/script>/s)[1];
const qsText = script.substring(script.indexOf('const QS='), script.indexOf('const REFS='));
eval(qsText.replace('const QS=', 'var QS='));
const mcq = QS.filter(q => typeof q.correct === 'number' && q.choices && q.choices.length === 4);
let correctLongest=0, gapOver5=0;
mcq.forEach(q => {
  const lens = q.choices.map(c => c.length);
  const correctLen = lens[q.correct];
  const otherMax = Math.max(...lens.filter((_,i) => i !== q.correct));
  if (correctLen > otherMax) correctLongest++;
  if (correctLen - otherMax >= 5) gapOver5++;
});
console.log('Correct strictly longest:', correctLongest, '/', mcq.length);
console.log('Gap >=5 chars:', gapOver5, '/', mcq.length);
"
```

### Audit position bias:
```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const script = html.match(/<script>([\s\S]*?)<\/script>/s)[1];
const qsText = script.substring(script.indexOf('const QS='), script.indexOf('const REFS='));
eval(qsText.replace('const QS=', 'var QS='));
const mcq = QS.filter(q => typeof q.correct === 'number' && q.choices && q.choices.length === 4);
const posCounts = [0,0,0,0];
mcq.forEach(q => posCounts[q.correct]++);
posCounts.forEach((n,i) => console.log('Position ' + String.fromCharCode(65+i) + ':', n, '(' + (n/mcq.length*100).toFixed(1) + '%)'));
"
```

### Git push workflow:
Danny will provide a fresh GitHub personal access token when needed. Never hard-code or store one. Always remind him to revoke it after pushing.
```bash
cd /home/claude
rm -rf agpcnp-prep
git clone https://<TOKEN>@github.com/dadestefano1024-lgtm/agpcnp-prep.git
# ... make edits to agpcnp-prep/index.html ...
cd /home/claude/agpcnp-prep
git config user.email "update@agpcnp.com"
git config user.name "AGPCNP Update"
git add index.html
git commit -m "descriptive message"
git push origin main
```

---

## Question schema

Each question in the `QS` array has this structure:
```js
{id:"card_015",
 domain:"Diagnosis",          // must be one of: Patient Assessment Process, Diagnosis, Planning, Implementation, Evaluation, Preventive Care, Chronic Disease Management, Professional Practice
 system:"Cardiology",
 topic:"Acute Coronary Syndrome",
 type:"mcq",
 vignette:"A 65-year-old woman presents with...",  // OR combined into 'question' field for older questions
 stem:"What is the diagnosis?",
 choices:["A text","B text","C text","D text"],
 correct:1,                   // zero-indexed
 explanation:"800-1500 char explanation...",
 boardPearl:"Punchy 1-2 sentence pearl with specific numbers.",
 refKey:"acs"}
```

Two variants exist in the bank:
- Newer rewrites use separate `vignette` + `stem` fields
- Older questions use a single combined `question` field (rendered by the app as `q.stem || q.question`)

Don't change the schema. The app renders both formats correctly.

---

## Rules for editing answer choices

1. **Never touch the `correct` index value.** It's always left at its original position. Position shuffle is a separate pass.
2. **Never touch the `explanation` or `boardPearl` fields.** Teaching content lives there, not in the choice text.
3. **Never delete or add questions.** Only modify the `choices` array.
4. **Always validate JS cleanliness after any batch of edits.** Escape characters in `str_replace` can break the JS syntax.
5. **Target format per choice:** 10-35 characters, bare noun phrase (diagnosis/drug/test/action), no em-dashes, no parenthetical qualifiers, no trial names.
6. **Length band within a question:** target ≤15 chars between shortest and longest, max 20.
7. **Correct answer must never be noticeably the longest.** If a rewrite produces correct-is-longest by more than 3-4 chars, tighten it or extend a distractor.
8. **If the original choice has compound content** ("Diagnosis X with management plan Y"), extract only the thing the stem is actually asking about. The rest goes in the stem or explanation.

---

## Things to NOT do

- Do not do massive batches (50+ questions) in a single `str_replace` round — risk of context limit mid-batch losing work
- Do not push to GitHub without explicit approval from Danny
- Do not bump cache version on trivial changes — only on content that actually changes what users see
- Do not touch the skilled infrastructure (service worker, PWA manifest, analytics dashboard) unless specifically asked
- Do not explain things at length when a short answer will do. Danny values directness.
- Do not start over or "rethink the approach" if it's documented here. Assume prior decisions hold.
- Do not ask Danny to approve technical details he can't reasonably judge — show him the output of one example instead

---

## Contact style

Danny's tone is direct and efficient. He pushes back when something is wrong (and he's usually right when he does). Take his pushback seriously, don't get defensive. When you make a mistake, own it briefly and fix it. Don't spiral into over-apology.

---

*Last updated: April 17, 2026 (Claude Opus 4.7 session)*
