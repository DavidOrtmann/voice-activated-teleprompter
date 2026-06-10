# Spec: Voice-Activated Souffleur

## Overview

Extend the existing voice teleprompter into a theatre souffleur. The core speech matching engine stays intact; what's new is **play structure awareness** (characters, stage directions, multi-character filtering) and a **prompter box view** optimised for prompting from the wings.

---

## 1. Play Parsing (`src/lib/play-parser.ts`)

**Input:** Raw text pasted from the web.

**Rules:**
- A line matching `/^([A-Z][A-Z\s]+):\s*(.*)/` is a character line. Group 1 = character name, group 2 = start of dialogue (may be empty if dialogue is on the next line).
- Text inside `(...)` is a stage direction — **stripped completely** from dialogue before matching and display.
- All other lines (ACT, SCENE headers, blank lines, continuation dialogue) are treated as continuation of the last active character.
- Lines that are purely whitespace or pure stage direction are discarded.

**Output data model:**
```ts
type PlayLine = {
  character: string      // e.g. "HAMLET"
  dialogue: string       // clean text, stage directions stripped
  lineIndex: number      // sequential position in the play
}

type ParsedPlay = {
  lines: PlayLine[]
  characters: string[]   // unique, in order of first appearance
}
```

---

## 2. Speech Matching Adaptation

The existing Levenshtein matcher operates on a flat `TextElement[]`. For the souffleur it needs to work on a **filtered token list**:

- Build a flat token sequence from dialogue only (stage directions already stripped by the parser).
- When a character filter is active, build the token sequence from **only that character's lines**.
- Each token carries a `lineIndex` so we can map position back to `PlayLine` for display.

The matcher itself (`computeSpeechRecognitionTokenIndex`) is unchanged; only its input changes.

---

## 3. Character Filter

- A dropdown in the navbar listing all characters extracted by the parser, plus an "All characters" option.
- Default: "All characters" (souffleur mode).
- When a character is selected, only their dialogue participates in speech matching, but **all lines remain visible** in scroll view — other characters' lines are visible but dimmed.

---

## 4. Display Modes (toggled by a button in the navbar)

### Scroll view (existing behaviour, adapted)
- Renders each `PlayLine` as a block: character name in a distinct colour/weight, dialogue text tokenised for highlighting.
- Stage directions: not rendered at all.
- Current position highlighted as before; auto-scrolls.

### Prompter box view (new)
- Shows only a **window of ~5 lines** centred on the current position.
- Layout per line: `[CHARACTER NAME]` on a small label above the dialogue.
- The current line is large and bright; upcoming lines shrink and fade.
- Past lines not shown.
- Dark background, maximised for readability.

---

## 5. State changes (Redux)

Extend `contentSlice` (or a new `playSlice`) with:
```ts
parsedPlay: ParsedPlay | null
selectedCharacter: string | "ALL"
displayMode: "scroll" | "prompter"
```

`setContent` triggers re-parse. `selectedCharacter` and `displayMode` persist across start/stop cycles.

---

## 6. What stays the same

- Speech recogniser (`speech-recognizer.ts`) — untouched.
- Levenshtein core (`levenshtein.ts`) — untouched.
- Edit mode (textarea paste) — untouched.
- Navbar controls: font size, language, scroll offset, opacity — all still relevant.
- Keyboard navigation (arrow keys to nudge position).

---

## 7. Out of scope (this iteration)

- Mobile / touch optimisation.
- Multiple simultaneous character assignments (one device per actor).
- Any non-`ALL CAPS:` script formats.
- Exporting or saving parsed plays.
