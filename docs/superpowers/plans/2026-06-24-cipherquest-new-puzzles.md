# CipherQuest v5 — New Puzzle Types & Scoring Overhaul Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 7 historical cipher puzzle types and a redesigned encoder/decoder scoring system to CipherQuest, producing `cipherquest_v5.html`.

**Architecture:** All changes occur in a single HTML file (`cipherquest_v4.html` → `cipherquest_v5.html`). New cipher engine functions follow the same pattern as existing ones (`toMorse`, `caesarShift`). Scoring reads a difficulty rating computed per-puzzle and updated by community fail-rate stats stored in localStorage after each attempt. Pigpen symbols are rendered as inline SVG generated entirely in JavaScript.

**Tech Stack:** Vanilla HTML/CSS/JavaScript, localStorage, inline SVG (Pigpen), Web Audio API (unchanged)

## Global Constraints

- Source: `C:\Users\alexf\Downloads\cipherquest_v4.html`
- Output: `C:\Users\alexf\Downloads\cipherquest_v5.html`
- No external assets, libraries, or build steps
- All localStorage keys prefixed with `cq_`
- Letters A–Z and spaces only in all cipher inputs/outputs
- J shares cell 24 with I in Polybius and Playfair grids
- Pigpen symbols are 24×24 inline SVG
- Difficulty tiers: `'easy'` | `'medium'` | `'hard'` (lowercase strings throughout)
- Community refinement triggers after 10+ decode attempts on a single puzzle
- Encoder points: timeout only (+20/35/50 by difficulty); slow correct solve >70% time (+10 flat); quit = 0
- Decoder score: `(timeLeft × 3) + diffBonus[diff] − (hintUsed ? 10 : 0)`; diffBonus = {easy:0, medium:25, hard:50}

---

## File Map

Only one file is modified end-to-end. Sections referenced below by their role, not line numbers (line numbers shift as content is inserted):

| Section | Role |
|---|---|
| `<style>` block | All CSS — add new UI styles here |
| `<!-- GAME -->` div | Add 7 new `*-ui` divs alongside existing ones |
| `<!-- PUZZLE SELECT -->` div | Update puzzle grid to 11 cards |
| `<!-- ENCODER PLATFORM -->` div | Category layout, extra inputs, difficulty badge |
| `// ── DATA ──` JS block | Add 14 new word pool arrays |
| `// ── GAME CORE ──` JS block | Add cipher engine functions, setup/check functions, routing |
| `// ── ENCODER PLATFORM ──` JS block | Update encoder functions |

---

## Task 1: Copy File and Add Cipher Engine Functions

**Files:**
- Create: `C:\Users\alexf\Downloads\cipherquest_v5.html` (copy of v4)
- Modify: JS section immediately after the existing `caesarShift` function

**Interfaces:**
- Produces:
  - `toAtbash(word: string): string`
  - `toVigenere(word: string, keyword: string): string`
  - `toRailFence(word: string, rails: number): string`
  - `toPolybius(word: string): string`
  - `toColumnar(word: string, keyword: string): string`
  - `toPigpenSVG(letter: string): string` (returns SVG element string)
  - `toPigpen(word: string): string` (returns concatenated SVG string)
  - `buildPlayfairGrid(keyword: string): string[]` (25-char array)
  - `toPlayfair(word: string, keyword: string): string`

- [ ] **Step 1: Copy v4 to v5**

```powershell
Copy-Item "C:\Users\alexf\Downloads\cipherquest_v4.html" "C:\Users\alexf\Downloads\cipherquest_v5.html"
```

- [ ] **Step 2: Add all cipher engine functions**

Open `cipherquest_v5.html`. Find the line `function caesarShift(text, shift) {` and insert the following block **immediately after the closing `}` of `caesarShift`**:

```javascript
// ── NEW CIPHER ENGINES ──

function toAtbash(word) {
  return word.toUpperCase().split('').map(c => {
    if (!/[A-Z]/.test(c)) return c;
    return String.fromCharCode(90 - (c.charCodeAt(0) - 65));
  }).join('');
}

function toVigenere(word, keyword) {
  const key = keyword.toUpperCase().replace(/[^A-Z]/g, '');
  let ki = 0;
  return word.toUpperCase().split('').map(c => {
    if (c === ' ') return ' ';
    if (!/[A-Z]/.test(c)) return c;
    const shift = key.charCodeAt(ki % key.length) - 65;
    ki++;
    return String.fromCharCode(((c.charCodeAt(0) - 65 + shift) % 26) + 65);
  }).join('');
}

function toRailFence(word, rails) {
  const fence = Array.from({length: rails}, () => []);
  let rail = 0, dir = 1;
  for (const c of word.toUpperCase()) {
    fence[rail].push(c);
    if (rail === 0) dir = 1;
    if (rail === rails - 1) dir = -1;
    rail += dir;
  }
  return fence.map(r => r.join('')).join(' ');
}

function toPolybius(word) {
  // 5×5 grid A-Z with I/J sharing cell (no J in grid)
  const grid = 'ABCDEFGHIKLMNOPQRSTUVWXYZ';
  return word.toUpperCase().split('').map(c => {
    if (c === ' ') return ' ';
    const ch = c === 'J' ? 'I' : c;
    const idx = grid.indexOf(ch);
    if (idx === -1) return c;
    return `${Math.floor(idx / 5) + 1}${(idx % 5) + 1}`;
  }).join(' ');
}

function toColumnar(word, keyword) {
  const key = keyword.toUpperCase().replace(/[^A-Z]/g, '');
  const numCols = key.length;
  const text = word.toUpperCase().replace(/ /g, 'X');
  const padded = text.padEnd(Math.ceil(text.length / numCols) * numCols, 'X');
  const numRows = padded.length / numCols;
  const grid = [];
  for (let r = 0; r < numRows; r++) {
    grid.push(padded.slice(r * numCols, (r + 1) * numCols).split(''));
  }
  const order = [...key].map((c, i) => ({c, i}))
    .sort((a, b) => a.c.localeCompare(b.c)).map(x => x.i);
  return order.map(col => grid.map(row => row[col]).join('')).join(' ');
}

function toPigpenSVG(letter) {
  const sz = 24, h = 12, pad = 4, r = sz - pad;
  const st = `stroke="#eee" stroke-width="2.5" stroke-linecap="round" fill="none"`;
  // Grid letters A–R: [top, right, bottom, left, hasDot]
  const G = {
    A:[0,1,1,0,0], B:[0,1,1,1,0], C:[0,0,1,1,0],
    D:[1,1,1,0,0], E:[1,1,1,1,0], F:[1,0,1,1,0],
    G:[1,1,0,0,0], H:[1,1,0,1,0], I:[1,0,0,1,0],
    J:[0,1,1,0,1], K:[0,1,1,1,1], L:[0,0,1,1,1],
    M:[1,1,1,0,1], N:[1,1,1,1,1], O:[1,0,1,1,1],
    P:[1,1,0,0,1], Q:[1,1,0,1,1], R:[1,0,0,1,1],
  };
  // X-pattern letters S–Z: [topLeft, topRight, bottomRight, bottomLeft, hasDot]
  const X = {
    S:[1,0,0,0,0], T:[0,1,0,0,0], U:[0,0,1,0,0], V:[0,0,0,1,0],
    W:[1,0,0,0,1], X:[0,1,0,0,1], Y:[0,0,1,0,1], Z:[0,0,0,1,1],
  };
  let inner = '';
  const vb = `0 0 ${sz} ${sz}`;
  const wrap = content =>
    `<svg viewBox="${vb}" width="${sz}" height="${sz}" style="display:inline-block;vertical-align:middle;margin:3px">${content}</svg>`;

  if (letter === ' ') return wrap('');
  if (G[letter]) {
    const [top, right, bottom, left, dot] = G[letter];
    if (top)    inner += `<line x1="${pad}" y1="${pad}" x2="${r}" y2="${pad}" ${st}/>`;
    if (right)  inner += `<line x1="${r}" y1="${pad}" x2="${r}" y2="${r}" ${st}/>`;
    if (bottom) inner += `<line x1="${pad}" y1="${r}" x2="${r}" y2="${r}" ${st}/>`;
    if (left)   inner += `<line x1="${pad}" y1="${pad}" x2="${pad}" y2="${r}" ${st}/>`;
    if (dot)    inner += `<circle cx="${h}" cy="${h}" r="2.5" fill="#eee"/>`;
  } else if (X[letter]) {
    const [tl, tr, br, bl, dot] = X[letter];
    if (tl) inner += `<line x1="${pad}" y1="${pad}" x2="${h}" y2="${h}" ${st}/>`;
    if (tr) inner += `<line x1="${r}" y1="${pad}" x2="${h}" y2="${h}" ${st}/>`;
    if (br) inner += `<line x1="${h}" y1="${h}" x2="${r}" y2="${r}" ${st}/>`;
    if (bl) inner += `<line x1="${h}" y1="${h}" x2="${pad}" y2="${r}" ${st}/>`;
    if (dot) inner += `<circle cx="${h}" cy="${h}" r="2.5" fill="#eee"/>`;
  }
  return wrap(inner);
}

function toPigpen(word) {
  return word.toUpperCase().split('').map(toPigpenSVG).join('');
}

function buildPlayfairGrid(keyword) {
  const key = keyword.toUpperCase().replace(/J/g, 'I').replace(/[^A-Z]/g, '');
  const seen = new Set();
  const grid = [];
  for (const c of key + 'ABCDEFGHIKLMNOPQRSTUVWXYZ') {
    if (!seen.has(c)) { seen.add(c); grid.push(c); }
  }
  return grid; // 25-element array
}

function playfairPos(grid, c) {
  const idx = grid.indexOf(c === 'J' ? 'I' : c);
  return {row: Math.floor(idx / 5), col: idx % 5};
}

function toPlayfair(word, keyword) {
  const grid = buildPlayfairGrid(keyword);
  const text = word.toUpperCase().replace(/J/g, 'I').replace(/[^A-Z]/g, '');
  const digraphs = [];
  let i = 0;
  while (i < text.length) {
    const a = text[i];
    const b = (i + 1 < text.length && text[i + 1] !== a) ? text[i + 1] : 'X';
    digraphs.push([a, b]);
    i += (b === 'X' && (i + 1 >= text.length || text[i + 1] === a)) ? 1 : 2;
  }
  return digraphs.map(([a, b]) => {
    const pa = playfairPos(grid, a), pb = playfairPos(grid, b);
    let ea, eb;
    if (pa.row === pb.row) {
      ea = grid[pa.row * 5 + (pa.col + 1) % 5];
      eb = grid[pb.row * 5 + (pb.col + 1) % 5];
    } else if (pa.col === pb.col) {
      ea = grid[((pa.row + 1) % 5) * 5 + pa.col];
      eb = grid[((pb.row + 1) % 5) * 5 + pb.col];
    } else {
      ea = grid[pa.row * 5 + pb.col];
      eb = grid[pb.row * 5 + pa.col];
    }
    return ea + eb;
  }).join(' ');
}
```

- [ ] **Step 3: Verify cipher functions in browser console**

Open `cipherquest_v5.html` in a browser. Open DevTools console and run:

```javascript
console.assert(toAtbash('CAT') === 'XZG', 'Atbash failed');
console.assert(toVigenere('HELLO', 'KEY') === 'RIJVS', 'Vigenere failed');
console.assert(toRailFence('HELLO', 2) === 'HLO EL', 'RailFence 2-rail failed');
console.assert(toPolybius('ACE') === '11 13 15', 'Polybius failed');
console.assert(toPlayfair('HIDE', 'CIPHER').includes('EP'), 'Playfair failed');
console.assert(toPigpen('ACE').includes('<svg'), 'Pigpen SVG failed');
console.log('All cipher engine checks passed');
```

Expected: "All cipher engine checks passed" with no assertion errors.

- [ ] **Step 4: Commit**

```powershell
git -C "C:\Users\alexf\Downloads" init; git -C "C:\Users\alexf\Downloads" add cipherquest_v5.html; git -C "C:\Users\alexf\Downloads" commit -m "feat: add 7 cipher engine functions to CipherQuest v5"
```

---

## Task 2: Add Word Data Pools

**Files:**
- Modify: `cipherquest_v5.html` — JS section immediately after `BINARY_HARD` array

**Interfaces:**
- Consumes: nothing (pure data)
- Produces: 14 new arrays — `ATBASH_EASY`, `ATBASH_HARD`, `VIGENERE_EASY`, `VIGENERE_HARD`, `RAILFENCE_EASY`, `RAILFENCE_HARD`, `POLYBIUS_EASY`, `POLYBIUS_HARD`, `COLUMNAR_EASY`, `COLUMNAR_HARD`, `PIGPEN_EASY`, `PIGPEN_HARD`, `PLAYFAIR_EASY`, `PLAYFAIR_HARD`
- Each entry shape: `{word: string, hint: string, keyword?: string, rails?: number}`

- [ ] **Step 1: Insert word pools after `BINARY_HARD` array**

Find the line `const BINARY_HARD = [` and locate its closing `];`. Insert the following block immediately after:

```javascript
// ── ATBASH POOLS ──
const ATBASH_EASY = [
  {word:'CAT',   hint:'Common household pet'},
  {word:'HERO',  hint:'A brave person'},
  {word:'MIND',  hint:'Where thoughts live'},
  {word:'PLANT', hint:'A living organism that grows'},
  {word:'CROWN', hint:'Worn by royalty'},
  {word:'FLAME', hint:'Fire in action'},
  {word:'BRAVE', hint:'Showing courage'},
  {word:'SLEEP', hint:'State of rest'},
];
const ATBASH_HARD = [
  {word:'KINGDOM',   hint:'Land ruled by a king'},
  {word:'MYSTERY',   hint:'Something unknown or secret'},
  {word:'PHANTOM',   hint:'A ghost or apparition'},
  {word:'LABYRINTH', hint:'A complex maze'},
  {word:'CLOCKWORK', hint:'Mechanical gears in motion'},
  {word:'UNIVERSE',  hint:'All of space and time'},
];

// ── VIGENERE POOLS ──
const VIGENERE_EASY = [
  {word:'OCEAN',  keyword:'KEY', hint:'Large body of salt water'},
  {word:'TIGER',  keyword:'CAT', hint:'Striped big cat'},
  {word:'FOREST', keyword:'OAK', hint:'Dense area of trees'},
  {word:'PLANET', keyword:'SUN', hint:'Orbits a star'},
  {word:'CASTLE', keyword:'KEY', hint:'Where royalty lived'},
  {word:'SILVER', keyword:'ORE', hint:'Shiny precious metal'},
  {word:'WINTER', keyword:'ICE', hint:'Coldest season'},
  {word:'BRIDGE', keyword:'KEY', hint:'Spans a gap'},
];
const VIGENERE_HARD = [
  {word:'SIGNAL',     keyword:'MORSE', hint:'A transmitted message'},
  {word:'PHANTOM',    keyword:'GHOST', hint:'A supernatural apparition'},
  {word:'LABYRINTH',  keyword:'MAZE',  hint:'A complex maze'},
  {word:'CLOCKWORK',  keyword:'TIME',  hint:'Mechanical gears in motion'},
  {word:'ASTRONOMY',  keyword:'STARS', hint:'Study of celestial objects'},
  {word:'REVOLUTION', keyword:'POWER', hint:'A complete turn or uprising'},
];

// ── RAIL FENCE POOLS ──
const RAILFENCE_EASY = [
  {word:'HELLO',  rails:2, hint:'Common greeting'},
  {word:'TIGER',  rails:2, hint:'Striped predator'},
  {word:'CASTLE', rails:2, hint:'Medieval fortress'},
  {word:'DRAGON', rails:2, hint:'Mythical fire-breather'},
  {word:'PLANET', rails:2, hint:'Orbits a star'},
  {word:'WINTER', rails:2, hint:'Coldest season'},
  {word:'BRIDGE', rails:2, hint:'Spans a gap'},
  {word:'SILVER', rails:2, hint:'Shiny precious metal'},
];
const RAILFENCE_HARD = [
  {word:'PHANTOM',   rails:3, hint:'A ghost or apparition'},
  {word:'LABYRINTH', rails:3, hint:'A complex maze'},
  {word:'CLOCKWORK', rails:3, hint:'Mechanical gears in motion'},
  {word:'DISCOVERY', rails:3, hint:'Finding something new'},
  {word:'FIREWORKS', rails:3, hint:'Colorful explosion display'},
  {word:'MESSENGER', rails:3, hint:'One who carries information'},
];

// ── POLYBIUS POOLS ──
const POLYBIUS_EASY = [
  {word:'ACE', hint:'Number one rank'},
  {word:'KEY', hint:'Opens a lock'},
  {word:'FOX', hint:'Clever orange animal'},
  {word:'MAP', hint:'Shows you where to go'},
  {word:'NET', hint:'Catches fish or connects computers'},
  {word:'OWL', hint:'Nocturnal bird of wisdom'},
  {word:'SUN', hint:'Center of our solar system'},
  {word:'WAR', hint:'Armed conflict between groups'},
];
const POLYBIUS_HARD = [
  {word:'SIGNAL',  hint:'A transmitted message'},
  {word:'PYTHON',  hint:'A large constricting snake'},
  {word:'KNIGHT',  hint:'Armored warrior on horseback'},
  {word:'GALAXY',  hint:'A system of billions of stars'},
  {word:'SPHINX',  hint:'Ancient Egyptian monument'},
  {word:'TYPHOON', hint:'A tropical cyclone'},
];

// ── COLUMNAR POOLS ──
const COLUMNAR_EASY = [
  {word:'HELLO',  keyword:'KEY',  hint:'Common greeting'},
  {word:'OCEAN',  keyword:'SEA',  hint:'Large body of salt water'},
  {word:'TIGER',  keyword:'CAT',  hint:'Striped big cat'},
  {word:'CASTLE', keyword:'FORT', hint:'Medieval fortress'},
  {word:'DRAGON', keyword:'FIRE', hint:'Mythical fire-breather'},
  {word:'PLANET', keyword:'STAR', hint:'Orbits a star'},
  {word:'WINTER', keyword:'COLD', hint:'Coldest season'},
  {word:'SILVER', keyword:'ORE',  hint:'Shiny precious metal'},
];
const COLUMNAR_HARD = [
  {word:'PHANTOM',   keyword:'GHOST',   hint:'A supernatural apparition'},
  {word:'LABYRINTH', keyword:'MAZE',    hint:'A complex maze'},
  {word:'CLOCKWORK', keyword:'GEARS',   hint:'Mechanical gears in motion'},
  {word:'DISCOVERY', keyword:'EXPLORE', hint:'Finding something new'},
  {word:'FIREWORKS', keyword:'SPARK',   hint:'Colorful explosion display'},
  {word:'MESSENGER', keyword:'CARRY',   hint:'One who carries information'},
];

// ── PIGPEN POOLS ──
const PIGPEN_EASY = [
  {word:'ACE', hint:'Number one rank'},
  {word:'BEE', hint:'Buzzing insect that makes honey'},
  {word:'DOG', hint:'Man\'s best friend'},
  {word:'EGG', hint:'Laid by a bird'},
  {word:'FIG', hint:'Sweet purple fruit'},
  {word:'HOP', hint:'A small jump'},
  {word:'ICE', hint:'Frozen water'},
  {word:'OAK', hint:'A large sturdy tree'},
];
const PIGPEN_HARD = [
  {word:'BEACON',  hint:'A guiding light or signal'},
  {word:'CHAPEL',  hint:'A small place of worship'},
  {word:'DRAGON',  hint:'Mythical fire-breather'},
  {word:'FALCON',  hint:'A fast bird of prey'},
  {word:'GOBLIN',  hint:'A mischievous mythical creature'},
  {word:'HERALD',  hint:'One who announces important news'},
];

// ── PLAYFAIR POOLS ──
const PLAYFAIR_EASY = [
  {word:'HIDE', keyword:'CIPHER', hint:'To conceal something'},
  {word:'ROCK', keyword:'STONE',  hint:'Solid mineral formation'},
  {word:'FIRE', keyword:'FLAME',  hint:'Hot and glowing'},
  {word:'GOLD', keyword:'MINE',   hint:'Precious yellow metal'},
  {word:'WIND', keyword:'STORM',  hint:'Moving air'},
  {word:'MOON', keyword:'NIGHT',  hint:'Orbits the Earth'},
  {word:'DARK', keyword:'SHADE',  hint:'Absence of light'},
  {word:'WOLF', keyword:'PACK',   hint:'Wild canine predator'},
];
const PLAYFAIR_HARD = [
  {word:'SIGNAL',  keyword:'MORSE',  hint:'A transmitted message'},
  {word:'PHANTOM', keyword:'GHOST',  hint:'A supernatural apparition'},
  {word:'KINGDOM', keyword:'CROWN',  hint:'Land ruled by a king'},
  {word:'ANCIENT', keyword:'RELIC',  hint:'From a very old time'},
  {word:'ECLIPSE', keyword:'SHADOW', hint:'When one body blocks another'},
  {word:'PYRAMID', keyword:'EGYPT',  hint:'Ancient triangular monument'},
];
```

- [ ] **Step 2: Verify pools in browser console**

```javascript
console.assert(ATBASH_EASY.length === 8, 'ATBASH_EASY count wrong');
console.assert(VIGENERE_EASY[0].keyword === 'KEY', 'VIGENERE keyword missing');
console.assert(RAILFENCE_HARD[0].rails === 3, 'RAILFENCE rails missing');
console.assert(POLYBIUS_EASY.length === 8, 'POLYBIUS_EASY count wrong');
console.assert(COLUMNAR_EASY[0].keyword === 'KEY', 'COLUMNAR keyword missing');
console.assert(PIGPEN_EASY.length === 8, 'PIGPEN_EASY count wrong');
console.assert(PLAYFAIR_EASY[0].keyword === 'CIPHER', 'PLAYFAIR keyword missing');
console.log('All pool checks passed');
```

Expected: "All pool checks passed" with no assertion errors.

- [ ] **Step 3: Commit**

```powershell
git -C "C:\Users\alexf\Downloads" add cipherquest_v5.html; git -C "C:\Users\alexf\Downloads" commit -m "feat: add word pools for 7 new cipher types"
```

---

## Task 3: Update getWordPool() and startGame() Routing

**Files:**
- Modify: `cipherquest_v5.html` — `getWordPool()` function and `startGame()` function

**Interfaces:**
- Consumes: 14 new pool arrays from Task 2
- Produces:
  - `getWordPool(type: string): Array` — extended to handle all 11 types
  - `startGame(type: string, daily?: boolean)` — extended UI visibility for 11 types

- [ ] **Step 1: Extend getWordPool()**

Find and replace the existing `getWordPool` function:

```javascript
// REPLACE THIS:
function getWordPool(type) {
  if (type==='morse')   return difficulty==='hard' ? MORSE_HARD   : MORSE_EASY;
  if (type==='anagram') return difficulty==='hard' ? ANAGRAM_HARD : ANAGRAM_EASY;
  if (type==='caesar')  return difficulty==='hard' ? CAESAR_HARD  : CAESAR_EASY;
  if (type==='binary')  return difficulty==='hard' ? BINARY_HARD  : BINARY_EASY;
  return [];
}

// WITH THIS:
function getWordPool(type) {
  const pools = {
    morse:    [MORSE_EASY,    MORSE_HARD],
    anagram:  [ANAGRAM_EASY,  ANAGRAM_HARD],
    caesar:   [CAESAR_EASY,   CAESAR_HARD],
    binary:   [BINARY_EASY,   BINARY_HARD],
    atbash:   [ATBASH_EASY,   ATBASH_HARD],
    vigenere: [VIGENERE_EASY, VIGENERE_HARD],
    railfence:[RAILFENCE_EASY,RAILFENCE_HARD],
    polybius: [POLYBIUS_EASY, POLYBIUS_HARD],
    columnar: [COLUMNAR_EASY, COLUMNAR_HARD],
    pigpen:   [PIGPEN_EASY,   PIGPEN_HARD],
    playfair: [PLAYFAIR_EASY, PLAYFAIR_HARD],
  };
  const pair = pools[type];
  if (!pair) return [];
  return difficulty === 'hard' ? pair[1] : pair[0];
}
```

- [ ] **Step 2: Extend startGame() UI visibility toggle**

Find this line inside `startGame()`:

```javascript
['morse','anagram','caesar','binary'].forEach(t =>
  document.getElementById(t+'-ui').style.display = t===type ? 'block' : 'none'
);
```

Replace it with:

```javascript
['morse','anagram','caesar','binary','atbash','vigenere','railfence','polybius','columnar','pigpen','playfair'].forEach(t =>
  document.getElementById(t+'-ui').style.display = t===type ? 'block' : 'none'
);
```

- [ ] **Step 3: Extend nextRound() routing**

Find `nextRound()` and extend its routing block:

```javascript
// REPLACE THIS block inside nextRound():
  if (currentGameType==='morse')   setupMorseRound();
  else if (currentGameType==='anagram') setupAnagramRound();
  else if (currentGameType==='caesar')  setupCaesarRound();
  else setupBinaryRound();

// WITH THIS:
  const setupMap = {
    morse:     setupMorseRound,
    anagram:   setupAnagramRound,
    caesar:    setupCaesarRound,
    binary:    setupBinaryRound,
    atbash:    setupAtbashRound,
    vigenere:  setupVigenereRound,
    railfence: setupRailFenceRound,
    polybius:  setupPolybiusRound,
    columnar:  setupColumnarRound,
    pigpen:    setupPigpenRound,
    playfair:  setupPlayfairRound,
  };
  (setupMap[currentGameType] || setupMorseRound)();
```

- [ ] **Step 4: Extend startGame() labels**

Find the `labels` object inside `startGame()`:

```javascript
// REPLACE:
const labels = {morse:'MORSE CODE',anagram:'ANAGRAM',caesar:'CAESAR CIPHER',binary:'BINARY'};

// WITH:
const labels = {
  morse:'MORSE CODE', anagram:'ANAGRAM', caesar:'CAESAR CIPHER', binary:'BINARY',
  atbash:'ATBASH CIPHER', vigenere:'VIGENÈRE CIPHER', railfence:'RAIL FENCE',
  polybius:'POLYBIUS SQUARE', columnar:'COLUMNAR', pigpen:'PIGPEN CIPHER',
  playfair:'PLAYFAIR CIPHER',
};
```

- [ ] **Step 5: Verify in browser console**

```javascript
difficulty = 'easy';
console.assert(getWordPool('atbash').length === 8, 'atbash easy pool wrong');
difficulty = 'hard';
console.assert(getWordPool('playfair').length === 6, 'playfair hard pool wrong');
difficulty = 'easy';
console.log('getWordPool checks passed');
```

Expected: "getWordPool checks passed"

- [ ] **Step 6: Commit**

```powershell
git -C "C:\Users\alexf\Downloads" add cipherquest_v5.html; git -C "C:\Users\alexf\Downloads" commit -m "feat: wire 11 cipher types into game routing"
```

---

## Task 4: Add CSS and HTML for 7 New Game UI Sections

**Files:**
- Modify: `cipherquest_v5.html` — `<style>` block and `<!-- GAME -->` HTML section

**Interfaces:**
- Consumes: nothing
- Produces: 7 new `<div id="*-ui">` blocks; new CSS classes `.keyword-display`, `.pigpen-display`, `.rf-rail`, `.polybius-ref`

- [ ] **Step 1: Add CSS for new UI elements**

Find the closing `</style>` tag and insert the following **before** it:

```css
/* ── NEW CIPHER UI ── */
.keyword-display { font-size: 11px; color: #c9a84c; letter-spacing: 3px; text-align: center; margin-bottom: 6px; }
.keyword-val { font-size: 15px; font-weight: bold; color: #c9a84c; letter-spacing: 4px; }
.pigpen-display { display: flex; flex-wrap: wrap; justify-content: center; align-items: center; padding: 0.8rem; min-height: 60px; gap: 2px; }
.rf-visual { font-family: monospace; font-size: 11px; color: #666; letter-spacing: 2px; line-height: 1.8; text-align: center; margin-bottom: 6px; }
.polybius-ref { display: grid; grid-template-columns: repeat(5, 1fr); gap: 3px; font-size: 9px; }
.polybius-cell { text-align: center; padding: 2px; }
.polybius-cell span { display: block; color: #888; font-size: 10px; font-weight: bold; }
.playfair-grid { display: grid; grid-template-columns: repeat(5, 1fr); gap: 2px; font-size: 10px; }
.playfair-cell { text-align: center; padding: 3px 0; color: #888; border: 0.5px solid #222; border-radius: 2px; }
```

- [ ] **Step 2: Add 7 new game UI divs inside the GAME screen**

Find `<div class="feedback" id="feedback"></div>` inside `<!-- GAME -->` and insert the following **immediately before** it:

```html
  <!-- ATBASH UI -->
  <div id="atbash-ui" style="display:none">
    <div class="cipher-box">
      <div class="cipher-label">REVERSE THE ALPHABET — A↔Z, B↔Y, C↔X</div>
      <div class="cipher-text" id="atbash-display"></div>
    </div>
    <div class="ref-box">
      <div class="ref-title">ATBASH REFERENCE</div>
      <div id="atbash-ref-grid"></div>
    </div>
    <input class="answer-input" id="atbash-input" type="text" placeholder="TYPE DECODED WORD..." autocomplete="off" />
    <div class="hint-row">
      <div class="hint-text" id="atbash-hint-text"></div>
      <button class="hint-btn" onclick="showHint('atbash')">HINT −10pts</button>
    </div>
    <button class="submit-btn" onclick="checkAtbash()">SUBMIT ↵</button>
  </div>

  <!-- VIGENERE UI -->
  <div id="vigenere-ui" style="display:none">
    <div class="cipher-box">
      <div class="cipher-label">VIGENÈRE — KEYWORD SHIFTS EACH LETTER</div>
      <div class="keyword-display">KEYWORD: <span class="keyword-val" id="vigenere-keyword-display"></span></div>
      <div class="cipher-text" id="vigenere-display"></div>
    </div>
    <div class="ref-box">
      <div class="ref-title">HOW TO DECODE: SUBTRACT KEYWORD LETTER POSITION FROM EACH LETTER</div>
      <div id="vigenere-ref-grid" style="font-size:10px; color:#444; text-align:center; padding:4px;"></div>
    </div>
    <input class="answer-input" id="vigenere-input" type="text" placeholder="TYPE DECODED MESSAGE..." autocomplete="off" />
    <div class="hint-row">
      <div class="hint-text" id="vigenere-hint-text"></div>
      <button class="hint-btn" onclick="showHint('vigenere')">HINT −10pts</button>
    </div>
    <button class="submit-btn" onclick="checkVigenere()">SUBMIT ↵</button>
  </div>

  <!-- RAIL FENCE UI -->
  <div id="railfence-ui" style="display:none">
    <div class="cipher-box">
      <div class="cipher-label" id="railfence-label">RAIL FENCE — READ THE ZIGZAG</div>
      <div class="rf-visual" id="railfence-visual"></div>
      <div class="cipher-text" id="railfence-display"></div>
    </div>
    <input class="answer-input" id="railfence-input" type="text" placeholder="TYPE DECODED WORD..." autocomplete="off" />
    <div class="hint-row">
      <div class="hint-text" id="railfence-hint-text"></div>
      <button class="hint-btn" onclick="showHint('railfence')">HINT −10pts</button>
    </div>
    <button class="submit-btn" onclick="checkRailFence()">SUBMIT ↵</button>
  </div>

  <!-- POLYBIUS UI -->
  <div id="polybius-ui" style="display:none">
    <div class="cipher-box">
      <div class="cipher-label">POLYBIUS SQUARE — NUMBER PAIRS = ROW, COLUMN</div>
      <div class="cipher-text" id="polybius-display" style="font-size:14px; letter-spacing:2px;"></div>
    </div>
    <div class="ref-box">
      <div class="ref-title">POLYBIUS GRID (A–Z, I=J)</div>
      <div id="polybius-ref-grid" class="polybius-ref"></div>
    </div>
    <input class="answer-input" id="polybius-input" type="text" placeholder="TYPE DECODED WORD..." autocomplete="off" />
    <div class="hint-row">
      <div class="hint-text" id="polybius-hint-text"></div>
      <button class="hint-btn" onclick="showHint('polybius')">HINT −10pts</button>
    </div>
    <button class="submit-btn" onclick="checkPolybius()">SUBMIT ↵</button>
  </div>

  <!-- COLUMNAR UI -->
  <div id="columnar-ui" style="display:none">
    <div class="cipher-box">
      <div class="cipher-label">COLUMNAR — REORDER COLUMNS BY KEYWORD</div>
      <div class="keyword-display">KEYWORD: <span class="keyword-val" id="columnar-keyword-display"></span></div>
      <div class="cipher-text" id="columnar-display" style="font-size:13px; letter-spacing:2px;"></div>
    </div>
    <div class="ref-box">
      <div class="ref-title">COLUMN ORDER</div>
      <div id="columnar-ref-grid" style="font-size:10px; color:#555; text-align:center;"></div>
    </div>
    <input class="answer-input" id="columnar-input" type="text" placeholder="TYPE DECODED WORD..." autocomplete="off" />
    <div class="hint-row">
      <div class="hint-text" id="columnar-hint-text"></div>
      <button class="hint-btn" onclick="showHint('columnar')">HINT −10pts</button>
    </div>
    <button class="submit-btn" onclick="checkColumnar()">SUBMIT ↵</button>
  </div>

  <!-- PIGPEN UI -->
  <div id="pigpen-ui" style="display:none">
    <div class="cipher-box">
      <div class="cipher-label">PIGPEN CIPHER — MATCH SYMBOLS TO LETTERS</div>
      <div class="pigpen-display" id="pigpen-display"></div>
    </div>
    <div class="ref-box">
      <div class="ref-title">PIGPEN REFERENCE</div>
      <div id="pigpen-ref-grid" style="text-align:center; padding:4px;"></div>
    </div>
    <input class="answer-input" id="pigpen-input" type="text" placeholder="TYPE DECODED WORD..." autocomplete="off" />
    <div class="hint-row">
      <div class="hint-text" id="pigpen-hint-text"></div>
      <button class="hint-btn" onclick="showHint('pigpen')">HINT −10pts</button>
    </div>
    <button class="submit-btn" onclick="checkPigpen()">SUBMIT ↵</button>
  </div>

  <!-- PLAYFAIR UI -->
  <div id="playfair-ui" style="display:none">
    <div class="cipher-box">
      <div class="cipher-label">PLAYFAIR CIPHER — DECODE LETTER PAIRS</div>
      <div class="keyword-display">KEYWORD: <span class="keyword-val" id="playfair-keyword-display"></span></div>
      <div class="cipher-text" id="playfair-display" style="font-size:14px; letter-spacing:3px;"></div>
    </div>
    <div class="ref-box">
      <div class="ref-title">PLAYFAIR GRID</div>
      <div id="playfair-ref-grid" class="playfair-grid"></div>
    </div>
    <input class="answer-input" id="playfair-input" type="text" placeholder="TYPE DECODED WORD..." autocomplete="off" />
    <div class="hint-row">
      <div class="hint-text" id="playfair-hint-text"></div>
      <button class="hint-btn" onclick="showHint('playfair')">HINT −10pts</button>
    </div>
    <button class="submit-btn" onclick="checkPlayfair()">SUBMIT ↵</button>
  </div>
```

- [ ] **Step 3: Verify HTML in browser**

Open `cipherquest_v5.html`. Open DevTools Elements panel and confirm:
- `document.getElementById('atbash-ui')` returns a div
- `document.getElementById('pigpen-display')` returns a div
- `document.getElementById('playfair-ref-grid')` returns a div

- [ ] **Step 4: Commit**

```powershell
git -C "C:\Users\alexf\Downloads" add cipherquest_v5.html; git -C "C:\Users\alexf\Downloads" commit -m "feat: add CSS and HTML for 7 new cipher game UI sections"
```

---

## Task 5: Add Setup, Check, and Reference Builder Functions

**Files:**
- Modify: `cipherquest_v5.html` — JS section after `setupBinaryRound()` function

**Interfaces:**
- Consumes: cipher engine functions from Task 1; word pools from Task 2; `state` object; `pickUnused()`; `startTimer()`
- Produces:
  - `setupAtbashRound()`, `checkAtbash()`
  - `setupVigenereRound()`, `checkVigenere()`
  - `setupRailFenceRound()`, `checkRailFence()`
  - `setupPolybiusRound()`, `checkPolybius()`, `buildPolybiusRef()`
  - `setupColumnarRound()`, `checkColumnar()`
  - `setupPigpenRound()`, `checkPigpen()`, `buildPigpenRef()`
  - `setupPlayfairRound()`, `checkPlayfair()`, `buildPlayfairRef(keyword)`
- State additions used: `state.currentKeyword`, `state.currentRails`

- [ ] **Step 1: Add state fields to startGame()**

Find this line inside `startGame()`:

```javascript
state = { score:0, round:0, totalRounds:5, timer:null, timeLeft:timeLimit, timeLimit, correct:0, totalTime:0, hintUsed:false, currentWord:'', caesarShift:0 };
```

Replace with:

```javascript
state = { score:0, round:0, totalRounds:5, timer:null, timeLeft:timeLimit, timeLimit, correct:0, totalTime:0, hintUsed:false, currentWord:'', caesarShift:0, currentKeyword:'', currentRails:2, currentDifficulty:'easy', _puzzleId:null };
```

- [ ] **Step 2: Add setup and check functions for all 7 types**

Find the closing `}` of `setupBinaryRound()` and insert the following block immediately after:

```javascript
// ── ATBASH ──
function buildAtbashRef() {
  document.getElementById('atbash-ref-grid').innerHTML =
    `<div style="display:grid;grid-template-columns:repeat(6,1fr);gap:3px;font-size:9px;text-align:center">` +
    'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.split('').map(c => {
      const enc = toAtbash(c);
      return `<div><span style="color:#888;font-weight:bold">${enc}</span><span style="color:#444">→${c}</span></div>`;
    }).join('') + `</div>`;
}
function setupAtbashRound() {
  const {item} = pickUnused();
  state.currentWord = item.word || item.text;
  state.currentHint = item.hint;
  state._encoder = item._encoder;
  state._puzzleId = item._id || null;
  state.currentDifficulty = calcDifficulty('atbash', state.currentWord.replace(/ /g,'').length);
  document.getElementById('atbash-display').textContent = toAtbash(state.currentWord);
  document.getElementById('atbash-input').value = '';
  document.getElementById('atbash-input').className = 'answer-input';
  document.getElementById('atbash-hint-text').textContent = '';
  buildAtbashRef();
}
function checkAtbash() {
  const val = document.getElementById('atbash-input').value.trim().toUpperCase();
  if (!val) return;
  clearInterval(state.timer);
  const correct = val === state.currentWord;
  document.getElementById('atbash-input').className = 'answer-input ' + (correct ? 'correct' : 'wrong');
  resolveRound(correct);
}

// ── VIGENERE ──
function buildVigenereRef(keyword) {
  document.getElementById('vigenere-ref-grid').innerHTML =
    keyword.split('').map((k, i) =>
      `<span style="color:#555">${String.fromCharCode(65+i%26)}+${k}(${k.charCodeAt(0)-64}) </span>`
    ).join('') + '<br><span style="color:#444;font-size:9px;">To decode: subtract each keyword letter value from encoded letter</span>';
}
function setupVigenereRound() {
  const {item} = pickUnused();
  state.currentWord = item.word || item.text;
  state.currentHint = item.hint;
  state.currentKeyword = item.keyword || 'KEY';
  state._encoder = item._encoder;
  state._puzzleId = item._id || null;
  state.currentDifficulty = calcDifficulty('vigenere', state.currentWord.replace(/ /g,'').length);
  document.getElementById('vigenere-keyword-display').textContent = state.currentKeyword;
  document.getElementById('vigenere-display').textContent = toVigenere(state.currentWord, state.currentKeyword);
  document.getElementById('vigenere-input').value = '';
  document.getElementById('vigenere-input').className = 'answer-input';
  document.getElementById('vigenere-hint-text').textContent = '';
  buildVigenereRef(state.currentKeyword);
}
function checkVigenere() {
  const val = document.getElementById('vigenere-input').value.trim().toUpperCase();
  if (!val) return;
  clearInterval(state.timer);
  const correct = val === state.currentWord;
  document.getElementById('vigenere-input').className = 'answer-input ' + (correct ? 'correct' : 'wrong');
  resolveRound(correct);
}

// ── RAIL FENCE ──
function buildRailFenceVisual(word, rails) {
  const fence = Array.from({length: rails}, () => Array(word.length).fill(' '));
  let rail = 0, dir = 1;
  for (let i = 0; i < word.length; i++) {
    fence[rail][i] = word[i];
    if (rail === 0) dir = 1;
    if (rail === rails - 1) dir = -1;
    rail += dir;
  }
  return fence.map(r => r.join('')).join('\n');
}
function setupRailFenceRound() {
  const {item} = pickUnused();
  state.currentWord = item.word || item.text;
  state.currentHint = item.hint;
  state.currentRails = item.rails || 2;
  state._encoder = item._encoder;
  state._puzzleId = item._id || null;
  state.currentDifficulty = calcDifficulty('railfence', state.currentWord.replace(/ /g,'').length);
  document.getElementById('railfence-label').textContent = `RAIL FENCE — ${state.currentRails} RAILS`;
  document.getElementById('railfence-visual').textContent = buildRailFenceVisual(state.currentWord, state.currentRails);
  document.getElementById('railfence-display').textContent = toRailFence(state.currentWord, state.currentRails);
  document.getElementById('railfence-input').value = '';
  document.getElementById('railfence-input').className = 'answer-input';
  document.getElementById('railfence-hint-text').textContent = '';
}
function checkRailFence() {
  const val = document.getElementById('railfence-input').value.trim().toUpperCase();
  if (!val) return;
  clearInterval(state.timer);
  const correct = val === state.currentWord;
  document.getElementById('railfence-input').className = 'answer-input ' + (correct ? 'correct' : 'wrong');
  resolveRound(correct);
}

// ── POLYBIUS ──
function buildPolybiusRef() {
  const grid = 'ABCDEFGHIKLMNOPQRSTUVWXYZ';
  document.getElementById('polybius-ref-grid').innerHTML =
    grid.split('').map((c, i) => {
      const row = Math.floor(i / 5) + 1, col = (i % 5) + 1;
      return `<div class="polybius-cell"><span>${c}</span>${row}${col}</div>`;
    }).join('');
}
function setupPolybiusRound() {
  const {item} = pickUnused();
  state.currentWord = item.word || item.text;
  state.currentHint = item.hint;
  state._encoder = item._encoder;
  state._puzzleId = item._id || null;
  state.currentDifficulty = calcDifficulty('polybius', state.currentWord.replace(/ /g,'').length);
  document.getElementById('polybius-display').textContent = toPolybius(state.currentWord);
  document.getElementById('polybius-input').value = '';
  document.getElementById('polybius-input').className = 'answer-input';
  document.getElementById('polybius-hint-text').textContent = '';
  buildPolybiusRef();
}
function checkPolybius() {
  const val = document.getElementById('polybius-input').value.trim().toUpperCase().replace(/J/g,'I');
  if (!val) return;
  clearInterval(state.timer);
  const correct = val === state.currentWord.replace(/J/g,'I');
  document.getElementById('polybius-input').className = 'answer-input ' + (correct ? 'correct' : 'wrong');
  resolveRound(correct);
}

// ── COLUMNAR ──
function buildColumnarRef(keyword) {
  const key = keyword.toUpperCase();
  const order = [...key].map((c, i) => ({c, i}))
    .sort((a, b) => a.c.localeCompare(b.c));
  document.getElementById('columnar-ref-grid').innerHTML =
    `<div style="display:flex;gap:8px;justify-content:center;flex-wrap:wrap">` +
    order.map((o, rank) =>
      `<div style="text-align:center"><div style="color:#888;font-weight:bold">${o.c}</div><div style="color:#444">#${rank+1}</div></div>`
    ).join('') + `</div>`;
}
function setupColumnarRound() {
  const {item} = pickUnused();
  state.currentWord = item.word || item.text;
  state.currentHint = item.hint;
  state.currentKeyword = item.keyword || 'KEY';
  state._encoder = item._encoder;
  state._puzzleId = item._id || null;
  state.currentDifficulty = calcDifficulty('columnar', state.currentWord.replace(/ /g,'').length);
  document.getElementById('columnar-keyword-display').textContent = state.currentKeyword;
  document.getElementById('columnar-display').textContent = toColumnar(state.currentWord, state.currentKeyword);
  document.getElementById('columnar-input').value = '';
  document.getElementById('columnar-input').className = 'answer-input';
  document.getElementById('columnar-hint-text').textContent = '';
  buildColumnarRef(state.currentKeyword);
}
function checkColumnar() {
  const val = document.getElementById('columnar-input').value.trim().toUpperCase().replace(/X/g,'');
  if (!val) return;
  clearInterval(state.timer);
  const correct = val === state.currentWord.replace(/ /g,'');
  document.getElementById('columnar-input').className = 'answer-input ' + (correct ? 'correct' : 'wrong');
  resolveRound(correct);
}

// ── PIGPEN ──
function buildPigpenRef() {
  const letters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
  document.getElementById('pigpen-ref-grid').innerHTML =
    `<div style="display:flex;flex-wrap:wrap;justify-content:center;gap:4px">` +
    letters.split('').map(c =>
      `<div style="text-align:center">${toPigpenSVG(c)}<div style="font-size:8px;color:#555;margin-top:2px">${c}</div></div>`
    ).join('') + `</div>`;
}
function setupPigpenRound() {
  const {item} = pickUnused();
  state.currentWord = item.word || item.text;
  state.currentHint = item.hint;
  state._encoder = item._encoder;
  state._puzzleId = item._id || null;
  state.currentDifficulty = calcDifficulty('pigpen', state.currentWord.replace(/ /g,'').length);
  document.getElementById('pigpen-display').innerHTML = toPigpen(state.currentWord);
  document.getElementById('pigpen-input').value = '';
  document.getElementById('pigpen-input').className = 'answer-input';
  document.getElementById('pigpen-hint-text').textContent = '';
  buildPigpenRef();
}
function checkPigpen() {
  const val = document.getElementById('pigpen-input').value.trim().toUpperCase();
  if (!val) return;
  clearInterval(state.timer);
  const correct = val === state.currentWord;
  document.getElementById('pigpen-input').className = 'answer-input ' + (correct ? 'correct' : 'wrong');
  resolveRound(correct);
}

// ── PLAYFAIR ──
function buildPlayfairRef(keyword) {
  const grid = buildPlayfairGrid(keyword);
  document.getElementById('playfair-ref-grid').innerHTML =
    grid.map(c => `<div class="playfair-cell">${c}</div>`).join('');
}
function setupPlayfairRound() {
  const {item} = pickUnused();
  state.currentWord = item.word || item.text;
  state.currentHint = item.hint;
  state.currentKeyword = item.keyword || 'CIPHER';
  state._encoder = item._encoder;
  state._puzzleId = item._id || null;
  state.currentDifficulty = calcDifficulty('playfair', state.currentWord.replace(/ /g,'').length);
  document.getElementById('playfair-keyword-display').textContent = state.currentKeyword;
  document.getElementById('playfair-display').textContent = toPlayfair(state.currentWord, state.currentKeyword);
  document.getElementById('playfair-input').value = '';
  document.getElementById('playfair-input').className = 'answer-input';
  document.getElementById('playfair-hint-text').textContent = '';
  buildPlayfairRef(state.currentKeyword);
}
function checkPlayfair() {
  const val = document.getElementById('playfair-input').value.trim().toUpperCase().replace(/J/g,'I');
  if (!val) return;
  clearInterval(state.timer);
  const correct = val === state.currentWord.replace(/J/g,'I');
  document.getElementById('playfair-input').className = 'answer-input ' + (correct ? 'correct' : 'wrong');
  resolveRound(correct);
}
```

- [ ] **Step 3: Add keyboard listeners for 7 new inputs**

Find the `// ── KEYBOARD ──` section:

```javascript
// FIND:
document.getElementById('answer-input').addEventListener('keydown',e=>{ if(e.key==='Enter') checkMorse(); });
document.getElementById('caesar-input').addEventListener('keydown',e=>{ if(e.key==='Enter') checkCaesar(); });
document.getElementById('binary-input').addEventListener('keydown',e=>{ if(e.key==='Enter') checkBinary(); });
document.getElementById('name-input').addEventListener('keydown',e=>{ if(e.key==='Enter') saveName(); });

// ADD IMMEDIATELY AFTER (before // ── INIT ──):
document.getElementById('atbash-input').addEventListener('keydown',e=>{ if(e.key==='Enter') checkAtbash(); });
document.getElementById('vigenere-input').addEventListener('keydown',e=>{ if(e.key==='Enter') checkVigenere(); });
document.getElementById('railfence-input').addEventListener('keydown',e=>{ if(e.key==='Enter') checkRailFence(); });
document.getElementById('polybius-input').addEventListener('keydown',e=>{ if(e.key==='Enter') checkPolybius(); });
document.getElementById('columnar-input').addEventListener('keydown',e=>{ if(e.key==='Enter') checkColumnar(); });
document.getElementById('pigpen-input').addEventListener('keydown',e=>{ if(e.key==='Enter') checkPigpen(); });
document.getElementById('playfair-input').addEventListener('keydown',e=>{ if(e.key==='Enter') checkPlayfair(); });
```

- [ ] **Step 4: Also extend showHint() to handle new types**

Find `showHint(type)` function:

```javascript
// REPLACE:
function showHint(type) {
  if (state.hintUsed) return;
  state.hintUsed = true;
  const ids = {morse:'morse-hint-text',anagram:'anagram-hint-text',caesar:'caesar-hint-text',binary:'binary-hint-text'};
  document.getElementById(ids[type]).textContent = '💡 ' + state.currentHint;
}

// WITH:
function showHint(type) {
  if (state.hintUsed) return;
  state.hintUsed = true;
  const ids = {
    morse:'morse-hint-text', anagram:'anagram-hint-text',
    caesar:'caesar-hint-text', binary:'binary-hint-text',
    atbash:'atbash-hint-text', vigenere:'vigenere-hint-text',
    railfence:'railfence-hint-text', polybius:'polybius-hint-text',
    columnar:'columnar-hint-text', pigpen:'pigpen-hint-text',
    playfair:'playfair-hint-text',
  };
  const el = document.getElementById(ids[type]);
  if (el) el.textContent = '💡 ' + state.currentHint;
}
```

- [ ] **Step 5: Manual gameplay test**

Open `cipherquest_v5.html` → Play → select each new cipher type and complete one round. Verify:
- Cipher text displays correctly
- Reference box appears
- Correct answer is accepted
- Wrong answer shows red border
- Hint works and deducts 10 pts

- [ ] **Step 6: Commit**

```powershell
git -C "C:\Users\alexf\Downloads" add cipherquest_v5.html; git -C "C:\Users\alexf\Downloads" commit -m "feat: add setup/check functions for all 7 new cipher types"
```

---

## Task 6: Scoring Overhaul

**Files:**
- Modify: `cipherquest_v5.html` — JS section containing `resolveRound()`, `timeUp()`, and the `// ── DATA ──` block

**Interfaces:**
- Consumes: `state.currentDifficulty`, `state._puzzleId`, `state._encoder`, `state.timeLimit`, `state.timeLeft`
- Produces:
  - `calcDifficulty(type: string, wordLength: number): 'easy'|'medium'|'hard'`
  - `updatePuzzleStats(puzzleId: string|null, solved: boolean, timeUsed: number): void`
  - Modified `resolveRound(correct: boolean)` — new decoder bonus + encoder partial points
  - Modified `timeUp()` — new encoder timeout points by difficulty

- [ ] **Step 1: Add calcDifficulty() and updatePuzzleStats()**

Find `// ── STORAGE ──` section and insert the following block **immediately after** the `incrementEncoderSubmitted` function:

```javascript
// ── DIFFICULTY & STATS ──
function calcDifficulty(type, wordLength) {
  const complexity = {
    morse:0, atbash:0,
    caesar:1, binary:1, vigenere:1, railfence:1, polybius:1, anagram:1,
    pigpen:2, columnar:2, playfair:2,
  };
  let tier = complexity[type] ?? 1;
  if (wordLength <= 4) tier = Math.max(0, tier - 1);
  if (wordLength >= 8) tier = Math.min(2, tier + 1);
  return ['easy','medium','hard'][tier];
}

function refineDifficulty(baseDiff, puzzle) {
  const attempts = (puzzle.solves || 0) + (puzzle.fails || 0);
  if (attempts < 10) return baseDiff;
  const failRate = (puzzle.fails || 0) / attempts;
  const tiers = ['easy','medium','hard'];
  let idx = tiers.indexOf(baseDiff);
  if (failRate < 0.3) idx = Math.max(0, idx - 1);
  else if (failRate > 0.6) idx = Math.min(2, idx + 1);
  return tiers[idx];
}

function updatePuzzleStats(puzzleId, solved, timeUsed) {
  if (!puzzleId) return;
  const puzzles = getEncoderPuzzles();
  const idx = puzzles.findIndex(p => p.id === puzzleId);
  if (idx === -1) return;
  if (solved) puzzles[idx].solves = (puzzles[idx].solves || 0) + 1;
  else        puzzles[idx].fails  = (puzzles[idx].fails  || 0) + 1;
  puzzles[idx].totalTime = (puzzles[idx].totalTime || 0) + timeUsed;
  save('cq_encoder_puzzles', puzzles.slice(0, 50));
}
```

- [ ] **Step 2: Replace resolveRound() with new scoring logic**

```javascript
// REPLACE the entire resolveRound function:
function resolveRound(correct) {
  const diff = state.currentDifficulty || 'easy';
  const diffBonus = {easy:0, medium:25, hard:50};
  const timeUsed = state.timeLimit - state.timeLeft;

  if (correct) {
    const timePts = Math.max(10, state.timeLeft * 3);
    const pts = timePts + diffBonus[diff] - (state.hintUsed ? 10 : 0);
    state.score += pts;
    state.correct++;
    showFeedback(true, '+' + pts + ' pts — CORRECT!');
    playCorrect();
    if (state._encoder && timeUsed > state.timeLimit * 0.7) {
      addEncoderPoints(state._encoder, 10);
    }
    updatePuzzleStats(state._puzzleId, true, timeUsed);
  } else {
    showFeedback(false, 'WRONG — Answer: ' + state.currentWord);
    playWrong();
    updatePuzzleStats(state._puzzleId, false, timeUsed);
  }
  state.totalTime += timeUsed;
  setTimeout(() => nextRound(), 1600);
}
```

- [ ] **Step 3: Replace timeUp() with new encoder timeout scoring**

```javascript
// REPLACE the entire timeUp function:
function timeUp() {
  const diff = state.currentDifficulty || 'easy';
  const encPts = {easy:20, medium:35, hard:50};
  showFeedback(false, "TIME'S UP — Answer: " + state.currentWord);
  playWrong();
  if (state._encoder) addEncoderPoints(state._encoder, encPts[diff]);
  updatePuzzleStats(state._puzzleId, false, state.timeLimit);
  state.totalTime += state.timeLimit;
  setTimeout(() => nextRound(), 1800);
}
```

- [ ] **Step 4: Set currentDifficulty in all existing setup functions**

For each of the 4 original setup functions, add a `state.currentDifficulty` line. Find `setupMorseRound`, `setupAnagramRound`, `setupCaesarRound`, `setupBinaryRound` and add the calc call after `state.currentWord` is set:

In `setupMorseRound()`, after `state.currentWord = item.word || item.text;` add:
```javascript
  state.currentDifficulty = calcDifficulty('morse', state.currentWord.replace(/ /g,'').length);
  state._puzzleId = item._id || null;
```

In `setupAnagramRound()`, after `state.currentWord = item.word || item.text;` add:
```javascript
  state.currentDifficulty = calcDifficulty('anagram', state.currentWord.replace(/ /g,'').length);
  state._puzzleId = item._id || null;
```

In `setupCaesarRound()`, after `state.currentWord = item.text || item.word;` add:
```javascript
  state.currentDifficulty = calcDifficulty('caesar', state.currentWord.replace(/ /g,'').length);
  state._puzzleId = item._id || null;
```

In `setupBinaryRound()`, after `state.currentWord = item.word || item.text;` add:
```javascript
  state.currentDifficulty = calcDifficulty('binary', state.currentWord.replace(/ /g,'').length);
  state._puzzleId = item._id || null;
```

- [ ] **Step 5: Verify scoring in browser console**

```javascript
console.assert(calcDifficulty('morse', 3) === 'easy', 'morse 3-char should be easy');
console.assert(calcDifficulty('playfair', 6) === 'hard', 'playfair 6-char should be hard');
console.assert(calcDifficulty('caesar', 3) === 'easy', 'caesar 3-char drops to easy');
console.assert(calcDifficulty('pigpen', 9) === 'hard', 'pigpen 9-char ceiling hard');
console.log('calcDifficulty checks passed');
```

Expected: "calcDifficulty checks passed"

Play one full game. On the result screen, verify scores reflect difficulty bonuses (Medium puzzle correct = timePts + 25).

- [ ] **Step 6: Commit**

```powershell
git -C "C:\Users\alexf\Downloads" add cipherquest_v5.html; git -C "C:\Users\alexf\Downloads" commit -m "feat: scoring overhaul with difficulty bonuses and encoder timeout points"
```

---

## Task 7: Update Puzzle Select Screen (11 Cards)

**Files:**
- Modify: `cipherquest_v5.html` — `<!-- PUZZLE SELECT -->` HTML section

**Interfaces:**
- Consumes: `startGame(type)` — all 11 type strings
- Produces: 11 puzzle cards in a 2-column scrollable grid grouped under a single section title

- [ ] **Step 1: Replace puzzle grid HTML**

Find the entire `<div class="puzzle-grid">` block inside `<!-- PUZZLE SELECT -->` and replace it with:

```html
  <div class="puzzle-grid">
    <div class="puzzle-card" onclick="startGame('morse')">
      <div class="puzzle-icon">•−</div>
      <div class="puzzle-name">MORSE CODE</div>
      <div class="puzzle-desc">Decode dot-dash signals</div>
      <span class="badge available">AVAILABLE</span>
    </div>
    <div class="puzzle-card" onclick="startGame('anagram')">
      <div class="puzzle-icon">ABC</div>
      <div class="puzzle-name">ANAGRAM</div>
      <div class="puzzle-desc">Unscramble letter tiles</div>
      <span class="badge available">AVAILABLE</span>
    </div>
    <div class="puzzle-card" onclick="startGame('caesar')">
      <div class="puzzle-icon">+3</div>
      <div class="puzzle-name">CAESAR CIPHER</div>
      <div class="puzzle-desc">Reverse the letter shift</div>
      <span class="badge available">AVAILABLE</span>
    </div>
    <div class="puzzle-card" onclick="startGame('binary')">
      <div class="puzzle-icon">01</div>
      <div class="puzzle-name">BINARY</div>
      <div class="puzzle-desc">Convert 8-bit groups</div>
      <span class="badge available">AVAILABLE</span>
    </div>
    <div class="puzzle-card" onclick="startGame('atbash')">
      <div class="puzzle-icon">A↔Z</div>
      <div class="puzzle-name">ATBASH</div>
      <div class="puzzle-desc">Ancient reverse alphabet</div>
      <span class="badge available">AVAILABLE</span>
    </div>
    <div class="puzzle-card" onclick="startGame('vigenere')">
      <div class="puzzle-icon">KEY</div>
      <div class="puzzle-name">VIGENÈRE</div>
      <div class="puzzle-desc">Keyword multi-shift cipher</div>
      <span class="badge available">AVAILABLE</span>
    </div>
    <div class="puzzle-card" onclick="startGame('railfence')">
      <div class="puzzle-icon">≋</div>
      <div class="puzzle-name">RAIL FENCE</div>
      <div class="puzzle-desc">Zigzag transposition</div>
      <span class="badge available">AVAILABLE</span>
    </div>
    <div class="puzzle-card" onclick="startGame('polybius')">
      <div class="puzzle-icon">##</div>
      <div class="puzzle-name">POLYBIUS</div>
      <div class="puzzle-desc">Number grid coordinates</div>
      <span class="badge available">AVAILABLE</span>
    </div>
    <div class="puzzle-card" onclick="startGame('columnar')">
      <div class="puzzle-icon">|||</div>
      <div class="puzzle-name">COLUMNAR</div>
      <div class="puzzle-desc">Column reorder by keyword</div>
      <span class="badge available">AVAILABLE</span>
    </div>
    <div class="puzzle-card" onclick="startGame('pigpen')">
      <div class="puzzle-icon">⊞</div>
      <div class="puzzle-name">PIGPEN</div>
      <div class="puzzle-desc">Masonic symbol cipher</div>
      <span class="badge available">AVAILABLE</span>
    </div>
    <div class="puzzle-card" onclick="startGame('playfair')">
      <div class="puzzle-icon">5×5</div>
      <div class="puzzle-name">PLAYFAIR</div>
      <div class="puzzle-desc">WWI digraph grid cipher</div>
      <span class="badge available">AVAILABLE</span>
    </div>
  </div>
```

- [ ] **Step 2: Manual test**

Open the game → PLAY → verify all 11 cards appear in a 2-column scrollable grid. Tap each card and confirm the correct game type launches.

- [ ] **Step 3: Commit**

```powershell
git -C "C:\Users\alexf\Downloads" add cipherquest_v5.html; git -C "C:\Users\alexf\Downloads" commit -m "feat: update puzzle select to show all 11 cipher types"
```

---

## Task 8: Update Encoder Platform

**Files:**
- Modify: `cipherquest_v5.html` — Encoder Platform HTML and JS functions

**Interfaces:**
- Consumes: cipher engine functions from Task 1; `calcDifficulty()` from Task 6
- Produces:
  - Categorized 11-type cipher selector
  - Extra input fields: `#enc-keyword-input`, `#enc-rails-select`
  - Difficulty badge shown on submission: `#enc-stat-difficulty`
  - Updated `selectEncType()`, `updateEncPreview()`, `submitPuzzle()`, `resetEncoder()`

- [ ] **Step 1: Replace Encoder Platform HTML**

Find the entire `<!-- ENCODER PLATFORM -->` div and replace its `enc-step-1` content (the part containing `.enc-type-grid`) with:

```html
  <div class="enc-step active" id="enc-step-1">
    <div style="font-size:10px; color:#555; letter-spacing:1px; margin-bottom:1rem;">Pick a cipher type and submit a word or phrase. You earn points every time a decoder's time runs out on your puzzle. Harder puzzles earn more.</div>

    <div class="enc-label">SUBSTITUTION CIPHERS</div>
    <div class="enc-type-grid">
      <div class="enc-type-card" onclick="selectEncType('morse')" id="enc-morse">
        <div class="enc-type-icon">•−</div><div class="enc-type-name">MORSE</div>
      </div>
      <div class="enc-type-card" onclick="selectEncType('atbash')" id="enc-atbash">
        <div class="enc-type-icon">A↔Z</div><div class="enc-type-name">ATBASH</div>
      </div>
      <div class="enc-type-card" onclick="selectEncType('caesar')" id="enc-caesar">
        <div class="enc-type-icon">+3</div><div class="enc-type-name">CAESAR</div>
      </div>
      <div class="enc-type-card" onclick="selectEncType('vigenere')" id="enc-vigenere">
        <div class="enc-type-icon">KEY</div><div class="enc-type-name">VIGENÈRE</div>
      </div>
      <div class="enc-type-card" onclick="selectEncType('pigpen')" id="enc-pigpen">
        <div class="enc-type-icon">⊞</div><div class="enc-type-name">PIGPEN</div>
      </div>
    </div>

    <div class="enc-label" style="margin-top:1rem">TRANSPOSITION CIPHERS</div>
    <div class="enc-type-grid">
      <div class="enc-type-card" onclick="selectEncType('anagram')" id="enc-anagram">
        <div class="enc-type-icon">ABC</div><div class="enc-type-name">ANAGRAM</div>
      </div>
      <div class="enc-type-card" onclick="selectEncType('railfence')" id="enc-railfence">
        <div class="enc-type-icon">≋</div><div class="enc-type-name">RAIL FENCE</div>
      </div>
      <div class="enc-type-card" onclick="selectEncType('columnar')" id="enc-columnar">
        <div class="enc-type-icon">|||</div><div class="enc-type-name">COLUMNAR</div>
      </div>
    </div>

    <div class="enc-label" style="margin-top:1rem">GRID / COORDINATE CIPHERS</div>
    <div class="enc-type-grid">
      <div class="enc-type-card" onclick="selectEncType('binary')" id="enc-binary">
        <div class="enc-type-icon">01</div><div class="enc-type-name">BINARY</div>
      </div>
      <div class="enc-type-card" onclick="selectEncType('polybius')" id="enc-polybius">
        <div class="enc-type-icon">##</div><div class="enc-type-name">POLYBIUS</div>
      </div>
      <div class="enc-type-card" onclick="selectEncType('playfair')" id="enc-playfair">
        <div class="enc-type-icon">5×5</div><div class="enc-type-name">PLAYFAIR</div>
      </div>
    </div>

    <div class="enc-label">YOUR WORD OR PHRASE</div>
    <input class="enc-input" id="enc-word-input" placeholder="TYPE HERE..." maxlength="20" autocomplete="off" oninput="updateEncPreview()" />

    <div id="enc-keyword-row" style="display:none">
      <div class="enc-label">KEYWORD (REQUIRED)</div>
      <input class="enc-input" id="enc-keyword-input" placeholder="ENTER KEYWORD..." maxlength="12" autocomplete="off" oninput="updateEncPreview()" style="font-size:13px;margin-top:4px;" />
    </div>

    <div id="enc-rails-row" style="display:none">
      <div class="enc-label">NUMBER OF RAILS</div>
      <div style="display:flex;gap:8px;margin-top:4px">
        <button class="diff-btn active-easy" id="enc-rail-2" onclick="setEncRails(2)">2 RAILS</button>
        <button class="diff-btn" id="enc-rail-3" onclick="setEncRails(3)">3 RAILS</button>
      </div>
    </div>

    <div class="enc-preview-box" id="enc-preview-box" style="display:none">
      <div class="enc-preview-label">ENCODED PREVIEW</div>
      <div class="enc-preview-text" id="enc-preview-text"></div>
    </div>

    <div class="enc-label">HINT FOR DECODERS (OPTIONAL)</div>
    <input class="enc-hint-input" id="enc-hint-input" placeholder="Give a clue..." maxlength="40" autocomplete="off" />
    <button class="enc-btn gold" onclick="submitPuzzle()">SUBMIT PUZZLE ↵</button>
  </div>

  <div class="enc-step" id="enc-step-2">
    <div class="enc-success">
      <div class="enc-success-icon">✓</div>
      <div class="enc-success-title">PUZZLE SUBMITTED!</div>
      <div class="enc-success-sub">Decoders will now attempt your puzzle.<br>You earn points when their time runs out.</div>
      <div class="enc-stats-row">
        <div class="stat-item"><div class="stat-num" id="enc-stat-submitted" style="color:#c9a84c">0</div><div class="stat-lbl">SUBMITTED</div></div>
        <div class="stat-item"><div class="stat-num" id="enc-stat-points" style="color:#c9a84c">0</div><div class="stat-lbl">ENC POINTS</div></div>
        <div class="stat-item"><div class="stat-num" id="enc-stat-difficulty" style="color:#c9a84c">—</div><div class="stat-lbl">DIFFICULTY</div></div>
      </div>
      <button class="enc-btn gold" onclick="resetEncoder()">+ SUBMIT ANOTHER</button>
      <button class="enc-btn green" style="margin-top:8px" onclick="showScreen('menu')">⌂ MAIN MENU</button>
    </div>
  </div>
```

- [ ] **Step 2: Replace Encoder Platform JS functions**

Find the `// ── ENCODER PLATFORM ──` JS section and replace `selectEncType`, `updateEncPreview`, `submitPuzzle`, and `resetEncoder` with:

```javascript
// ── ENCODER PLATFORM ──
const KEYWORD_TYPES = ['vigenere','columnar','playfair'];
const RAILS_TYPES   = ['railfence'];
let encRails = 2;

function setEncRails(n) {
  encRails = n;
  document.getElementById('enc-rail-2').className = 'diff-btn' + (n===2?' active-easy':'');
  document.getElementById('enc-rail-3').className = 'diff-btn' + (n===3?' active-easy':'');
  updateEncPreview();
}

function selectEncType(type) {
  encType = type;
  document.querySelectorAll('.enc-type-card').forEach(c => c.classList.remove('selected'));
  document.getElementById('enc-' + type).classList.add('selected');
  document.getElementById('enc-keyword-row').style.display = KEYWORD_TYPES.includes(type) ? 'block' : 'none';
  document.getElementById('enc-rails-row').style.display   = RAILS_TYPES.includes(type)   ? 'block' : 'none';
  updateEncPreview();
}

function updateEncPreview() {
  const word = document.getElementById('enc-word-input').value.trim().toUpperCase();
  const keyword = document.getElementById('enc-keyword-input').value.trim().toUpperCase() || 'KEY';
  const box = document.getElementById('enc-preview-box');
  const txt = document.getElementById('enc-preview-text');
  if (!word || !encType) { box.style.display = 'none'; return; }
  box.style.display = 'block';
  if (encType === 'morse')     txt.textContent = toMorse(word);
  else if (encType === 'atbash')    txt.textContent = toAtbash(word);
  else if (encType === 'anagram')   txt.textContent = word.split('').sort(() => Math.random() - 0.5).join(' ');
  else if (encType === 'caesar')    { const sh = Math.floor(Math.random()*8)+2; txt.textContent = `SHIFT +${sh}: ${caesarShift(word,sh)}`; }
  else if (encType === 'binary')    txt.textContent = toBinary(word);
  else if (encType === 'vigenere')  txt.textContent = `KEY: ${keyword} → ${toVigenere(word, keyword)}`;
  else if (encType === 'railfence') txt.textContent = `${encRails} RAILS → ${toRailFence(word, encRails)}`;
  else if (encType === 'polybius')  txt.textContent = toPolybius(word);
  else if (encType === 'columnar')  txt.textContent = `KEY: ${keyword} → ${toColumnar(word, keyword)}`;
  else if (encType === 'pigpen')    txt.innerHTML   = toPigpen(word);
  else if (encType === 'playfair')  txt.textContent = `KEY: ${keyword} → ${toPlayfair(word, keyword)}`;
}

function validateSubmission(word) {
  if (word.length < 3) return 'Word must be at least 3 letters.';
  if (!/[AEIOU]/.test(word)) return 'Word must contain at least one vowel.';
  if (/(.)\1{3,}/.test(word)) return 'Word cannot repeat the same letter more than 3 times in a row.';
  if (/[^A-Z ]/.test(word)) return 'Only letters A–Z and spaces are allowed.';
  return null;
}

function submitPuzzle() {
  const word    = document.getElementById('enc-word-input').value.trim().toUpperCase();
  const hint    = document.getElementById('enc-hint-input').value.trim();
  const keyword = document.getElementById('enc-keyword-input').value.trim().toUpperCase() || 'KEY';

  const err = validateSubmission(word);
  if (err) { alert(err); return; }
  if (!encType) { alert('Please select a cipher type.'); return; }
  if (KEYWORD_TYPES.includes(encType) && keyword.length < 2) {
    alert('Please enter a keyword of at least 2 letters.'); return;
  }

  const name = getPlayerName();
  const wordLetters = word.replace(/ /g,'');
  const diff = calcDifficulty(encType, wordLetters.length);
  const puzzle = {
    id: Date.now(), type: encType, word, hint: hint || 'No hint provided',
    encoder: name, solves: 0, fails: 0, totalTime: 0, date: Date.now(),
    difficulty: diff,
    ...(KEYWORD_TYPES.includes(encType) && {keyword}),
    ...(RAILS_TYPES.includes(encType)   && {rails: encRails}),
  };
  saveEncoderPuzzle(puzzle);
  incrementEncoderSubmitted(name);

  const board = getEncoderBoard();
  const enc = board.find(e => e.name === name);
  document.getElementById('enc-stat-submitted').textContent  = enc ? enc.submitted : 1;
  document.getElementById('enc-stat-points').textContent     = enc ? enc.points : 0;
  document.getElementById('enc-stat-difficulty').textContent = diff.toUpperCase();

  document.getElementById('enc-step-1').classList.remove('active');
  document.getElementById('enc-step-2').classList.add('active');
}

function resetEncoder() {
  encType = null;
  encRails = 2;
  document.getElementById('enc-word-input').value    = '';
  document.getElementById('enc-keyword-input').value = '';
  document.getElementById('enc-hint-input').value    = '';
  document.getElementById('enc-preview-box').style.display    = 'none';
  document.getElementById('enc-keyword-row').style.display    = 'none';
  document.getElementById('enc-rails-row').style.display      = 'none';
  document.getElementById('enc-rail-2').className = 'diff-btn active-easy';
  document.getElementById('enc-rail-3').className = 'diff-btn';
  document.querySelectorAll('.enc-type-card').forEach(c => c.classList.remove('selected'));
  document.getElementById('enc-step-2').classList.remove('active');
  document.getElementById('enc-step-1').classList.add('active');
}
```

- [ ] **Step 3: Manual test of Encoder Platform**

Open the game → ENCODER PLATFORM. Verify:
- 3 category sections with correct ciphers listed
- Selecting Vigenère shows keyword input field
- Selecting Rail Fence shows 2/3 rails toggle; selecting Morse hides both
- Encoded preview updates as you type
- Submitting without a vowel shows validation error
- Submitting shows difficulty badge on success screen

- [ ] **Step 4: Commit**

```powershell
git -C "C:\Users\alexf\Downloads" add cipherquest_v5.html; git -C "C:\Users\alexf\Downloads" commit -m "feat: update Encoder Platform with category layout, extra inputs, and difficulty badge"
```

---

## Task 9: Community Flagging

**Files:**
- Modify: `cipherquest_v5.html` — game UI HTML and JS

**Interfaces:**
- Consumes: `state._puzzleId`, `getEncoderPuzzles()`, `save()`
- Produces:
  - `flagPuzzle()` — increments report count; hides puzzle from pool after 5 reports
  - `#flag-btn` — appears during game only when a community puzzle is active

- [ ] **Step 1: Add flag button CSS**

In the `<style>` block before `</style>`, add:

```css
.flag-btn { background: none; border: 0.5px solid #2a2a2a; border-radius: 6px; font-size: 9px; color: #333; cursor: pointer; padding: 3px 8px; font-family: monospace; margin-top: 6px; }
.flag-btn:hover { color: #D85A30; border-color: #D85A30; }
```

- [ ] **Step 2: Add flag button HTML**

Find `<div class="feedback" id="feedback"></div>` and insert immediately before it:

```html
  <div style="text-align:right; margin-top:4px;">
    <button class="flag-btn" id="flag-btn" style="display:none" onclick="flagPuzzle()">⚑ REPORT PUZZLE</button>
  </div>
```

- [ ] **Step 3: Add flagPuzzle() function**

Find the `// ── KEYBOARD ──` section and insert immediately before it:

```javascript
// ── COMMUNITY FLAGGING ──
function flagPuzzle() {
  if (!state._puzzleId) return;
  const puzzles = getEncoderPuzzles();
  const idx = puzzles.findIndex(p => p.id === state._puzzleId);
  if (idx === -1) return;
  puzzles[idx].reports = (puzzles[idx].reports || 0) + 1;
  if (puzzles[idx].reports >= 5) puzzles[idx].hidden = true;
  save('cq_encoder_puzzles', puzzles.slice(0, 50));
  document.getElementById('flag-btn').textContent = '⚑ REPORTED';
  document.getElementById('flag-btn').disabled = true;
}
```

- [ ] **Step 4: Show/hide flag button based on whether current puzzle is community-submitted**

In each setup function (all 11), after `state._puzzleId = item._id || null;` add:

```javascript
  document.getElementById('flag-btn').style.display = state._puzzleId ? 'inline-block' : 'none';
  document.getElementById('flag-btn').textContent = '⚑ REPORT PUZZLE';
  document.getElementById('flag-btn').disabled = false;
```

- [ ] **Step 5: Filter hidden puzzles from pickUnused()**

Find `pickUnused()` and update the `encPuzzles` filter line:

```javascript
// REPLACE:
const encPuzzles = getEncoderPuzzles().filter(p => p.type === currentGameType);

// WITH:
const encPuzzles = getEncoderPuzzles().filter(p => p.type === currentGameType && !p.hidden);
```

- [ ] **Step 6: Manual test**

Submit a community puzzle from the Encoder Platform. Play the same cipher type. When the community puzzle appears, verify the flag button is visible. Tap it — button should change to "REPORTED" and become disabled.

- [ ] **Step 7: Commit**

```powershell
git -C "C:\Users\alexf\Downloads" add cipherquest_v5.html; git -C "C:\Users\alexf\Downloads" commit -m "feat: add community flagging for submitted puzzles"
```

---

## Self-Review Checklist

- [x] All 7 new cipher types in Decoder game (Tasks 1–5)
- [x] All 7 new cipher types in Encoder Platform (Task 8)
- [x] New scoring: decoder difficulty bonus (Task 6)
- [x] New scoring: encoder timeout points by difficulty (Task 6)
- [x] New scoring: encoder partial points on slow correct solve (Task 6)
- [x] Encoder quit = 0 points (no code path adds points on quit)
- [x] Auto-calculated difficulty via `calcDifficulty()` (Task 6)
- [x] Community difficulty refinement via `refineDifficulty()` + `updatePuzzleStats()` (Task 6)
- [x] Encoder Platform: category layout (Task 8)
- [x] Encoder Platform: keyword input for Vigenère/Playfair/Columnar (Task 8)
- [x] Encoder Platform: rail count for Rail Fence (Task 8)
- [x] Encoder Platform: difficulty badge on submission (Task 8)
- [x] Submission validation: vowel check, repeat check, char whitelist (Task 8)
- [x] Community flagging: report button, hide after 5 reports (Task 9)
- [x] Puzzle select: all 11 cards (Task 7)
- [x] Keyboard Enter listeners for all new inputs (Task 5)
- [x] `showHint()` extended for all 11 types (Task 5)
