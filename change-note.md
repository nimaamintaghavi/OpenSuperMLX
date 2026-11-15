## Preserve Word's next paragraph and bullet when applying a preset

Word presets and voice commands now replace a saved native Word range while leaving its
final paragraph mark in place. This adapts the working replacement approach in VoiceMark's
`raycast/voicemark-dictate-hud.swift` and `voicemark-common.sh`.

### Problem

Applying a preset could pull the following bullet up, change list formatting, or insert a
rewrite beside the original text. The earlier attempts preserved boundary whitespace and
changed the paste fallback, but still used Word's live selection and replaced paragraph
marks along with the text.

Word's paragraph mark carries paragraph and list formatting. Deleting that mark and
inserting another carriage return can rebuild the paragraph or restyle the list. Keeping
the newline characters in a plain-text replacement therefore did not preserve the native
paragraph boundary. Word can also lose its visible highlight when focus changes, making a
generic paste an unreliable way to replace the original selection.

VoiceMark avoids both problems: save Word's own range, exclude its final paragraph mark,
and assign text to that range's content.

### Fix

- Capture the selected text, document identity, native start/end offsets, and list state
  through Word's scripting dictionary before requesting the rewrite.
- Replace that saved range through Word Automation. Verify the active document and the
  exact original range text before writing; do not derive Word offsets from Swift string
  lengths.
- When the selection ends in a paragraph mark, subtract one from the replacement range's
  end and remove the corresponding trailing CR from the replacement. The original Word
  paragraph mark remains untouched, preserving the boundary and its formatting.
- For an existing native list, remove an extra leading literal bullet from the model's
  response so Word's own bullet is not duplicated. Plain paragraphs retain literal bullets.
- If the saved range is unavailable, changed, or cannot be written, copy the result without
  falling back to paste. Word's focused state is not written back through Accessibility.
- Preserve boundary whitespace for ordinary selection replacements. Remove the earlier
  Word-specific generic-paste workaround; Word now has a dedicated native replacement path.
- Describe Word Automation accurately in the macOS permission prompt.

### Tests and verification

- 55 tests pass in a standalone XCTest bundle built from the real service/helper sources
  with injected infrastructure. They cover paragraph and bullet boundaries, partial and
  multi-paragraph selections, Unicode offsets, invalid ranges, native capture parsing,
  literal bullets, failed native writes, and both preset and voice-command routing.
- The paragraph regression fails when the range-end protection is removed. The native
  routing tests also failed before the service used the saved Word range.
- Both AppleScript templates compile against the installed Word scripting dictionary.
  The compiler check uses Word's application path because this sandbox cannot resolve its
  application ID through Launch Services; the app uses Word's bundle ID at runtime.
- All 92 app Swift sources compile and link using the target's Xcode linker command with
  existing native dependencies and Metal resources. The normal Xcode/hosted-test route is
  blocked by this session's nested sandbox restrictions.
- Rebuilt and locally signed `build/WordParagraphFix/OpenSuperMLX.app` with the Word
  Automation permission description. `codesign --verify --deep --strict` and the
  `OpenSuperMLX help correct` startup check both pass.

Contributor: [@nimaamintaghavi](https://github.com/nimaamintaghavi).
