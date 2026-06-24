# CipherQuest v5 — New Puzzle Types & Scoring Overhaul

**Date:** 2026-06-24  
**Status:** Approved  
**Scope:** Add 7 historical cipher types (11 total), overhaul encoder/decoder scoring, update Encoder Platform

---

## Overview

CipherQuest v5 adds 7 new historical cipher puzzle types to the existing 4, bringing the total to 11. All 11 types are available in both the Decoder game and the Encoder Platform. The scoring system is redesigned to create a more competitive encoder vs. decoder rivalry, with puzzle difficulty auto-calculated and refined by community performance.

---

## 1. New Puzzle Types

### Existing (unchanged)
1. Morse Code
2. Anagram
3. Caesar Cipher
4. Binary

### New — Substitution Ciphers
5. **Atbash** — Reverse alphabet (A↔Z, B↔Y, C↔X). Ancient Hebrew cipher.
6. **Vigenère** — Keyword-based multi-shift Caesar. Used in the US Civil War.
7. **Pigpen** — Letters replaced by geometric shapes from a grid. Used by Freemasons.

### New — Transposition Ciphers
8. **Rail Fence** — Text written in zigzag across N rails, read row by row.
9. **Columnar** — Text written in columns under a keyword, columns reordered alphabetically.

### New — Grid/Coordinate Ciphers
10. **Polybius Square** — 5×5 grid; each letter encoded as row+col number (A=11, B=12, I/J share cell).
11. **Playfair** — Pairs of letters encoded via a 5×5 keyword grid. Used in WWI.

---

## 2. Word Pools

Each new cipher type gets:
- **8 easy entries** — 3–6 letter words
- **6 hard entries** — longer words or multi-word phrases

Follows the same structure as existing `MORSE_EASY`, `MORSE_HARD` arrays.

---

## 3. Cipher Engine Functions

New encode functions follow the same pattern as existing `toMorse()` and `caesarShift()`:

| Function | Signature |
|---|---|
| `toAtbash(word)` | Returns reversed-alphabet string |
| `toVigenere(word, keyword)` | Returns Vigenère-encoded string |
| `toRailFence(word, rails)` | Returns rail fence encoded string |
| `toPolybius(word)` | Returns space-separated number pairs |
| `toColumnar(word, keyword)` | Returns columnar transposition string |
| `toPigpen(word)` | Returns inline SVG string of symbols |
| `toPlayfair(word, keyword)` | Returns Playfair-encoded digraph string |

**Pigpen SVG:** Symbols generated entirely in JavaScript as inline SVG. No external assets. The Pigpen reference grid is always visible during a round.

---

## 4. Scoring Overhaul

### 4a. Puzzle Difficulty Rating

Each puzzle receives an Easy / Medium / Hard rating. Two-stage calculation:

**Stage 1 — Base rating at submission (cipher type + word length):**

| Cipher Type | Base Complexity |
|---|---|
| Atbash, Morse | Low → Easy |
| Caesar, Binary, Vigenère, Rail Fence, Polybius, Anagram | Medium |
| Pigpen, Columnar, Playfair | High → Hard |

Word length modifier:
- ≤4 characters → drop one tier
- 5–7 characters → no change
- ≥8 characters → raise one tier

**Stage 2 — Community refinement (after 10+ decode attempts):**

| Community Fail Rate | Adjustment |
|---|---|
| < 30% | Drop one tier (floor: Easy) |
| 30–60% | No change |
| > 60% | Raise one tier (ceiling: Hard) |

Stored per puzzle in localStorage: `{ solves, fails, totalTime }`. Recalculated on each attempt.

---

### 4b. Decoder Scoring

`score = (timeLeft × 3) + difficulty_bonus − hint_penalty`

| Difficulty | Bonus | Hint Penalty |
|---|---|---|
| Easy | +0 | −10 pts |
| Medium | +25 | −10 pts |
| Hard | +50 | −10 pts |

---

### 4c. Encoder Scoring

Points awarded only on timeout (decoder runs out of time). No points for quit/dropout.

| Event | Easy | Medium | Hard |
|---|---|---|---|
| Decoder times out | +20 pts | +35 pts | +50 pts |
| Decoder solves but used >70% of time | +10 pts | +10 pts | +10 pts |
| Decoder quits / drops out | 0 | 0 | 0 |

---

## 5. Encoder Platform Updates

### 5a. Cipher Type Selector

Replace the 2×2 grid with a categorized scrollable layout:

**Substitution:** Morse, Atbash, Caesar, Vigenère, Pigpen  
**Transposition:** Anagram, Rail Fence, Columnar  
**Grid/Coordinate:** Binary, Polybius, Playfair

### 5b. Extra Inputs by Cipher Type

Some ciphers require additional encoder input before generating the encoded preview:

| Cipher | Extra Input |
|---|---|
| Vigenère | Keyword (shown to decoder during round) |
| Playfair | Keyword (used to build 5×5 grid) |
| Columnar | Keyword (determines column order) |
| Rail Fence | Rail count: 2 or 3 (encoder picks) |

All other ciphers require only the word/phrase + optional hint.

### 5c. Difficulty Badge on Submission

After submitting, encoder sees the auto-calculated difficulty rating (Easy / Medium / Hard) so they understand what tier their puzzle enters.

### 5d. Encoder Leaderboard

Unchanged in structure. Point totals updated to reflect new scoring model.

---

## 6. Submission Validation Rules

### Character Rules
- Letters A–Z and spaces only (no numbers, punctuation, or special characters)
- Minimum: 3 letters excluding spaces
- Maximum: 20 characters (unchanged)

### Heuristic Validation
- Must contain at least one vowel (A, E, I, O, U)
- No more than 3 consecutive identical characters (prevents "AAAA", "ZZZZZZ")

### Community Moderation
- Flag/report button on each community puzzle during a round
- Puzzle hidden from the pool after 5 reports

---

## 7. Technical Notes

### Output File
`cipherquest_v4.html` → `cipherquest_v5.html` (single file, no build step)

### Affected Code Areas
- `startGame()` — add 7 new type branches
- `getWordPool()` — add 7 new pool lookups
- `nextRound()` — route to new setup functions
- `resolveRound()` / `timeUp()` — new encoder scoring logic
- `submitPuzzle()` — validation rules + difficulty calculation + extra inputs
- Encoder Platform HTML — category-based cipher selector
- Leaderboard — no structural changes, scoring values update automatically

### Data Migration
No migration needed. New scoring applies to new entries only. Existing localStorage scores unaffected.
