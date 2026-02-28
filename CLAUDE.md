# Winit - macOS IME Fix Project

## Current Work

Branch: `fix/korean-ime-composition-macos`
Base: `master`

### What was done

macOS에서 한국어/일본어/중국어(CJK) IME 입력 소스 전환 후 발생하는 composition 문제 수정.

#### Commit 1: `78fd6cd2` - Korean IME composition issues
- Apple Korean IME가 입력소스 전환 후 `setMarkedText` 대신 `insertText`를 호출하는 버그 대응 (re-dispatch)
- Space 입력 시 "한 " 처럼 트리거 문자가 번들링되는 문제 (preedit snapshot + commit string splitting)
- 중간 preedit 상태가 앱에 전달되는 문제 (deferred preedit, Ghostty 패턴 참고)
- 단일 자모 백스페이스 시 삭제 안 되는 문제 (doCommandBySelector delete 예외)
- `markedRange`/`selectedRange`가 `{0,0}` 반환하도록 변경 (CJK IME composition 초기화 지원)

#### Commit 2: `5812f7f5` - Harden IME key event state management
- `insertText`의 preedit stripping에 `in_key_event` 가드 추가 (keyDown 외부에서 잘못된 splitting 방지)
- `preedit_before_key_event`를 keyDown 종료 시 클리어 (stale state 방지)
- keyDown 내 redundant `marked_text` 클리어 제거 (`insertText`가 commit 시 이미 클리어)

### Test Results
- Korean IME: 입력소스 전환 후 첫 글자 composition, space 확정, 백스페이스 모두 정상
- Japanese IME: `あい` 조합 중 영어 전환 시 `い` 유지 확인 (이전에는 소실)
- Chinese Pinyin: 기본 입력/확정, 입력소스 전환 모두 정상

### Key Files
- `src/platform_impl/macos/view.rs` — 모든 변경이 이 파일에 집중
- `docs/korean-ime-macos-fix.md` — 기술 문서

### Next Steps
- PR 생성 (upstream winit으로 또는 alacritty 연동용)
- alacritty에서 `Cargo.toml`의 winit 의존성이 `path = "../winit"`으로 로컬 참조 중 — PR 머지 후 버전 참조로 변경 필요

## Architecture Notes
- macOS IME는 `NSTextInputClient` 프로토콜을 통해 동작
- `interpretKeyEvents` → `setMarkedText`/`insertText`/`doCommandBySelector` 콜백 순서로 처리
- `ImeState`: `Disabled` → `Ground` → `Preedit` → `Committed` → `Ground` 순환
- Deferred preedit: `in_key_event` 플래그로 `keyDown` 내 중간 상태를 억제하고 최종 상태만 전달
