# Korean IME Fixes on macOS (winit)

This document records the Korean IME issues discovered and fixed in winit's
macOS `NSTextInputClient` implementation for the Alacritty terminal emulator.

All changes are in `src/platform_impl/macos/view.rs`.

---

## Issue 1: First Korean character not composing after input source switch

**Problem**: After switching from English to Korean (e.g., Cmd+Space), the
first Korean character is committed directly instead of entering Hangul
composition as preedit.

**Root Cause**: Apple Korean IME bug (rdar://FB17460926). After an input
source switch, the IME calls `insertText()` instead of `setMarkedText()` for
the first character, bypassing the normal composition flow.

**Fix**: Re-dispatch pattern in `keyDown`. When `insertText` is called in
`Disabled` state with non-ASCII text and IME is allowed, the IME is enabled
(transition to `Ground` state) without committing the text. Then in `keyDown`,
this condition is detected (state went from `Disabled` to `Ground` with no
marked text and no forward) and `interpretKeyEvents` is called a second time.
On the second pass, the IME properly calls `setMarkedText`, enabling correct
Hangul composition.

**Key code locations**: `insertText` (Disabled branch), `keyDown` (re-dispatch
section).

---

## Issue 2: Black preedit background on Korean + space

**Problem**: After typing Korean text and pressing space, the composed
character and space appear with preedit-style rendering (black background)
before being committed.

**Root Cause**: Two compounding issues:

1. Korean IME sends intermediate preedit states during `interpretKeyEvents`
   (e.g., `setMarkedText("한 ")` with the space included) before the final
   commit. These intermediate states reach the application and cause visual
   artifacts.
2. Korean IME bundles the triggering character (space) with the commit string
   in `insertText("한 ")` instead of sending `insertText("한")` + space separately.

**Fix**: Two-part solution:

1. **Deferred preedit sync** (inspired by Ghostty): Added `in_key_event` flag
   and `deferred_preedit` field. During `keyDown`, `setMarkedText` stores
   preedit events instead of emitting them immediately. Only the final preedit
   state is emitted after `interpretKeyEvents` completes. This prevents
   intermediate states from reaching the application.
2. **Commit string splitting**: Added `preedit_before_key_event` field that
   saves the preedit content at the start of `keyDown` (before
   `interpretKeyEvents`). In `insertText`, the commit string is compared with
   this saved value. If the commit string starts with the saved preedit and is
   longer, only the preedit portion is committed, and the `forward_key_to_app`
   flag is set so the original key event (e.g., Space) is forwarded to the
   application as a regular `KeyboardInput` event.

**Key code locations**: `setMarkedText` (deferred preedit), `keyDown`
(`in_key_event` flag, `preedit_before_key_event` save, deferred emit),
`insertText` (commit string splitting).

---

## Issue 3: Backspace on single jamo not working

**Problem**: When a single Korean jamo (e.g., "U+3147") is being composed and
the user presses backspace, the jamo stays on screen and the cursor moves
right instead of deleting the character.

**Root Cause**: The `doCommandBySelector` method had a blanket guard for
`ImeState::Committed` that blocked ALL commands, including
`deleteBackward:`. When Korean IME commits a single jamo on backspace, the
subsequent `doCommandBySelector` with `deleteBackward:` gets blocked, so the
delete never reaches the application.

**Fix**: Modified the `Committed` state guard in `doCommandBySelector` to
allow delete commands (`deleteBackward:`, `deleteForward:`,
`deleteWordBackward:`, `deleteWordForward:`) through by setting
`forward_key_to_app` to true. Other commands (like `insertNewline:` for
Enter) remain blocked to prevent double input.

**Key code location**: `doCommandBySelector` (Committed state handler).

---

## Additional Changes

- **selectedRange / markedRange**: Changed to return `NSRange::new(0, 0)`
  instead of `NSRange::new(NSNotFound, 0)` for the empty case. This matches
  Ghostty's behavior and signals a valid (empty) text state to CJK IMEs.

- **Input source tracking**: The `input_source` field update in `keyDown` now
  runs regardless of whether IME is currently enabled. This ensures the input
  source change is always detected, even when transitioning from a non-IME
  state. Additionally, `setMarkedText` updates `input_source` when called in
  `Disabled` state.

- **Event identity preservation**: `interpretKeyEvents` now receives the
  original `NSEvent` object rather than a modified copy. This preserves
  NSEvent object identity, which is important for Korean IME's internal state
  tracking. The option-as-alt replacement (`replaced_event`) is only used for
  `update_modifiers` and `create_key_event`.

- **Committed state forward (winit PR #4478 pattern)**: After committing composed
  text, subsequent `insertText` calls for triggering characters (e.g., space
  after Korean commit) are forwarded as regular key events instead of being
  committed through IME.

---

## Architecture Notes

The fix adds three new fields to `ViewState`:

| Field | Type | Purpose |
|---|---|---|
| `in_key_event` | `Cell<bool>` | Guards deferred preedit during `interpretKeyEvents` |
| `deferred_preedit` | `RefCell<Option<(String, Option<(usize, usize)>)>>` | Holds the last preedit state |
| `preedit_before_key_event` | `RefCell<String>` | Snapshot of preedit content before key processing |

All changes are backward-compatible and only affect Korean (and potentially
other CJK) IME code paths on macOS.

---

## References

- Alacritty issue: [#8079](https://github.com/alacritty/alacritty/issues/8079)
- Apple Radar: rdar://FB17460926 (Korean IME `insertText` instead of
  `setMarkedText` after input source switch)
- Ghostty's `NSTextInputClient` implementation (deferred preedit sync pattern)
- Godot PR [#85458](https://github.com/godotengine/godot/pull/85458) (Korean
  IME calling both `insertText` and `setMarkedText`)
