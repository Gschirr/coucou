# Linux-Sprachsteuerung – Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Push-to-talk-Sprachsteuerung im Linux-Build von Coucou: Diktat im Chat, Freigaben per Sprache, Anweisungen an kitty-Sessions, gesprochene Antworten auf Fragen zu laufenden Claude-Code-Sessions.

**Architecture:** Rust liefert dünne Werkzeug-Befehle (Aufnahme mit `pw-record`, Transkription über einen lokalen `whisper-server`, Sprachausgabe mit Piper, `claude -p` als Brain, `kitten @ send-text`). Die Logik sitzt im Frontend: ein reines, testbares Modul `src/core/voice.ts` (Session-Register + Router) und die Verdrahtung `src/island/voice.ts`. Freigaben laufen über die vorhandenen Befehle `approval_decision` / `approval_answer`.

**Tech Stack:** Tauri 2, Rust (tokio, reqwest), TypeScript ohne Framework, `node --test`, whisper.cpp, Piper, PipeWire, kitty Remote Control, Claude Code CLI 2.1.x.

**Spec:** `docs/superpowers/specs/2026-10-12-linux-voice-design.md` (bei Widerspruch gilt die Spec; Entscheidung: `docs/decisions/2026-10-12-linux-voice.md`)

Alle Pfade ohne Präfix sind relativ zu `windows/`. Befehle laufen in `windows/`, sofern nicht anders angegeben. Rust-Tests: `cargo test -p coucou --lib <filter>` (aus `windows/`). Frontend-Tests: `npm test` oder einzeln `node --import ./tests/setup.mjs --test tests/<name>.test.mjs`.

## Global Constraints

- Eigener Fork `Gschirr/coucou`, Branch `linux-voice`; Commits auf Englisch, Doku auf Deutsch.
- Keine neuen Crates, keine neuen npm-Pakete. Erlaubt: neue Features bestehender Crates (`reqwest` → `multipart`, `tokio` → `process`).
- Audio verlässt den Rechner nie; WAV-Dateien in `$XDG_RUNTIME_DIR`, nach Gebrauch löschen.
- `claude -p` immer mit `--safe-mode --tools ""`, Prompt über stdin, Arbeitsordner `~/.local/share/coucou/brain`, Timeout 60 s.
- Freigabe per Sprache nur durch festen Wortabgleich in `src/core/voice.ts`, nur auf `State.pendingApproval`, nie über `claude -p`, nie „Immer erlauben".
- Bestehende Ansichten nicht umstylen; Anzeige nur über `State.noteMessage` + View `"note"`.
- Hook darf Claude Code nie blockieren (Upstream-Regel, bleibt unberührt).
- Default-Shortcut `pushToTalk` = `Ctrl+Alt+V`; Action-IDs nie umbenennen.
- Sprache Deutsch (`language=de` für whisper).
- Neue UI-Texte nur in `src/i18n/extra.json` mit allen Sprachen außer Englisch (zh-Hans, hi, es, ar, fr, bn, pt, ru, id); `src/i18n/strings.json` wird generiert und nie von Hand geändert. Wo der Plan „strings.json" und deutsche Übersetzungen nennt, gilt das hier (Ruling 12.10.).
- Ohne `whisper-server`/`piper`/`pw-record` startet Coucou normal; nur die Sprachfunktion meldet, was fehlt.

## Review Focus

1. **Sprachaufnahme ohne Loslassen-Signal** (Portal liefert kein `Deactivated`): zweiter Tastendruck muss die Aufnahme beenden, nicht eine zweite starten. → Test in Task 9.
2. **Transkript mit Satzzeichen/Großschreibung** („Erlauben.", „Ja!") muss als Freigabe zählen, „Ja, aber nur lesen." nicht. → Tests in Task 7.
3. **Session endet während der 2-s-Vorschau**: Senden muss sauber scheitern („Session nicht erreichbar"), nicht in ein fremdes Fenster tippen. → Test in Task 9 (Register entfernt Session bei `SessionEnd`, Send prüft vor dem Senden erneut).
4. **Text mit Shell-/Steuerzeichen** (Anführungszeichen, `$`, Zeilenumbrüche) an `kitten send-text`: wird als einzelnes Argument ohne Shell übergeben, eingebettete Zeilenumbrüche werden zu Leerzeichen, genau ein `\r` am Ende. → Test in Task 8.
5. **whisper-server noch beim Laden des Modells** beim ersten Druck: Transkription wartet bis zu 30 s auf den Port statt sofort zu scheitern. → Test in Task 4.

---

### Task 0: Systemvorbereitung auf wintermute (Hauptsession, mit Stefan)

Kein Subagent: braucht `sudo` und Stefans Bestätigung (CLAUDE.md: erst erklären, dann handeln).

**Files:** keine im Repo. Extern: `~/.config/kitty/kitty.conf` (yadm), `~/Dokumente/system/CHANGELOG.md`.

- [ ] **Step 1: Snapshot**

```bash
sudo snapper -c root create -d "vor coucou voice"
```

- [ ] **Step 2: Pakete installieren (volles Upgrade, kein Teil-Upgrade)**

```bash
paru -Syu whisper-cpp piper-tts-bin
paru -S ggml-vulkan   # GPU backend; without it whisper runs on the CPU (10 s instead of 0.2 s)
```

✅ Erledigt 12.10.: Binary `piper-tts`, alle whisper-server-Flags vorhanden, Vulkan auf RX 7900 XT (encode 160 ms).

- [ ] **Step 3: Binärnamen und Flags festhalten**

```bash
pacman -Ql whisper-cpp | grep /bin/
pacman -Ql piper-tts-bin | grep /bin/
whisper-server --help 2>&1 | grep -E -- '--(host|port|model|language|no-timestamps)'
```

Erwartet: `whisper-server` mit `--host`, `--port`, `--model`, `--language`, `--no-timestamps`; Piper-Binary heißt `piper` oder `piper-tts`. **Wenn ein Name oder Flag abweicht:** Konstanten `WHISPER_SERVER`, `PIPER_NAMES` und `server_args()` in Task 4 vor dem Dispatch anpassen.

- [ ] **Step 4: Modell und Stimme laden**

```bash
mkdir -p ~/.local/share/coucou/models
cd ~/.local/share/coucou/models
curl -L -o ggml-large-v3-turbo-q5_0.bin https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v3-turbo-q5_0.bin
curl -L -o de_DE-thorsten-medium.onnx https://huggingface.co/rhasspy/piper-voices/resolve/main/de/de_DE/thorsten/medium/de_DE-thorsten-medium.onnx
curl -L -o de_DE-thorsten-medium.onnx.json https://huggingface.co/rhasspy/piper-voices/resolve/main/de/de_DE/thorsten/medium/de_DE-thorsten-medium.onnx.json
```

- [ ] **Step 5: Kette von Hand prüfen**

```bash
D=$XDG_RUNTIME_DIR
timeout -s INT 4 pw-record --rate 16000 --channels 1 --format s16 $D/t.wav   # 4 s sprechen
whisper-cli -m ~/.local/share/coucou/models/ggml-large-v3-turbo-q5_0.bin -l de -nt -f $D/t.wav
echo "Hallo Stefan, die Sprachausgabe funktioniert." | piper --model ~/.local/share/coucou/models/de_DE-thorsten-medium.onnx --output_file $D/s.wav && pw-play $D/s.wav
```

Erwartet: korrektes deutsches Transkript; hörbare Ansage. **Wenn nichts erkannt wird:** Default-Quelle in KDE auf das echte Mikrofon stellen (Default ist derzeit ein USB-„Line"-Eingang). **Wenn `piper` anders heißt oder `--output_file` nicht kennt:** in Task 4 `speak_args()` anpassen.

- [ ] **Step 6: kitty Remote Control**

In `~/.config/kitty/kitty.conf` ergänzen:

```
allow_remote_control socket-only
listen_on unix:@kitty-{kitty_pid}
```

```bash
yadm add ~/.config/kitty/kitty.conf && yadm commit -m "kitty: enable socket-only remote control for coucou" && yadm push
```

kitty neu starten. In einem neuen Fenster prüfen: `echo $KITTY_LISTEN_ON` → `unix:@kitty-<pid>`.

- [ ] **Step 7: CHANGELOG**

Eintrag in `~/Dokumente/system/CHANGELOG.md`: Datum 12.10.2026 (bzw. Ausführungstag), Was (whisper-cpp, piper-tts-bin, Modelle unter `~/.local/share/coucou/models`, kitty remote control), Warum (Coucou-Sprachsteuerung), Befehle (Steps 2, 4, 6), Rollback (`paru -Rns whisper-cpp piper-tts-bin`, Ordner löschen, zwei kitty-Zeilen entfernen, Snapshot „vor coucou voice").

---

### Task 1: Baseline und Fork-PKGBUILD

**Files:**
- Modify: `../linux/arch/PKGBUILD`

**Interfaces:**
- Consumes: nichts.
- Produces: grüne Baseline; PKGBUILD baut aus `Gschirr/coucou` (`main`).

- [ ] **Step 1: Baseline-Tests**

```bash
npm ci && npm test
cargo test -p coucou --lib
```

Erwartet: alles grün. **Wenn nicht grün:** Fehler notieren und an Stefan melden, bevor irgendetwas geändert wird.

- [ ] **Step 2: PKGBUILD auf den Fork umstellen**

In `../linux/arch/PKGBUILD`:

```bash
# Maintainer: FabioLukas123 (upstream), Stefan Gschirr (fork with Linux voice control)
url="https://github.com/Gschirr/coucou"
```

`optdepends` ersetzen durch:

```bash
optdepends=(
  'gnome-keyring: store API keys (or any other Secret Service provider)'
  'kwallet: store API keys (or any other Secret Service provider)'
  'whisper-cpp: push-to-talk speech recognition (whisper-server)'
  'ggml-vulkan: run whisper on the GPU'
  'piper-tts-bin: spoken answers'
  'pipewire: microphone capture and playback (pw-record, pw-play)'
  'kitty: send dictated instructions to Claude Code sessions'
)
```

`source` ersetzen durch:

```bash
source=("$_pkgname::git+https://github.com/Gschirr/coucou.git#branch=main")
```

- [ ] **Step 3: Dev-Build starten und Hooks prüfen**

```bash
npm run tauri dev
```

In Coucou: Settings → Claude Code → Install hooks (Stefan klickt, Diff prüfen). Dann in einem zweiten Terminal:

```bash
echo '{"hook_event_name":"UserPromptSubmit","session_id":"t1","cwd":"/home/stefan/Code/finanzen","prompt":"hallo"}' | ~/.local/share/coucou/bin/coucou-hook
```

Erwartet: Island zeigt die Pill mit Namen „finanzen". Danach die geänderte `~/.claude/settings.json` im claude-config-Repo committen und pushen (`git -C ~/.claude commit -am "coucou: install hooks" && git -C ~/.claude push`).

- [ ] **Step 4: Commit**

```bash
git add ../linux/arch/PKGBUILD
git commit -m "build(arch): build the fork, add voice optdepends"
```

---

### Task 2: `claude -p` als Brain und Chat-Anbieter

**Files:**
- Create: `src-tauri/src/claude_cli.rs`
- Modify: `src-tauri/Cargo.toml:23` (tokio-Feature `process`)
- Modify: `src-tauri/src/lib.rs` (`mod claude_cli;`, Befehl `brain_ask`, in `generate_handler!`)
- Modify: `src-tauri/src/chat.rs:200-220` (`send`: neuer Zweig)
- Modify: `src/core/providers.ts:24` (`PROVIDERS`)
- Modify: `src/core/bridge.ts` (`brainAsk`)

**Interfaces:**
- Produces (Rust): `claude_cli::PROVIDER: &str = "claude-cli"`; `claude_cli::args(system: &str, model: &str, resume: Option<&str>, persist: bool) -> Vec<String>`; `claude_cli::parse(stdout: &[u8]) -> Result<(String, Option<String>), String>`; `claude_cli::run(args: Vec<String>, prompt: &str) -> Result<(String, Option<String>), String>` (async); `claude_cli::send(chat, model, query, context) -> Result<ChatReply, String>` (async).
- Produces (Tauri): `brain_ask(system: String, query: String, model: String) -> Result<String, String>`.
- Produces (TS): `Bridge.brainAsk(system: string, query: string, model: string): Promise<string>`.

- [ ] **Step 1: Failing Tests schreiben** (Modul `tests` am Ende der neuen Datei `src-tauri/src/claude_cli.rs`)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn args_keep_the_users_setup_out_and_take_no_tools() {
        let a = args("Be brief.", "haiku", None, false);
        assert_eq!(&a[..6], ["-p", "--safe-mode", "--tools", "", "--output-format", "json"]);
        assert!(a.windows(2).any(|w| w == ["--model", "haiku"]));
        assert!(a.windows(2).any(|w| w == ["--system-prompt", "Be brief."]));
        assert!(a.contains(&"--no-session-persistence".to_string()));
        assert!(!a.contains(&"--resume".to_string()));
    }

    #[test]
    fn a_chat_turn_resumes_its_session_and_keeps_it() {
        let a = args("", "", Some("abc-123"), true);
        assert!(a.windows(2).any(|w| w == ["--resume", "abc-123"]));
        assert!(!a.contains(&"--no-session-persistence".to_string()));
        assert!(!a.contains(&"--model".to_string()));
    }

    #[test]
    fn parse_reads_result_and_session() {
        let out = br#"{"type":"result","is_error":false,"result":" Hallo Stefan. ","session_id":"s-1"}"#;
        assert_eq!(parse(out).unwrap(), ("Hallo Stefan.".to_string(), Some("s-1".to_string())));
    }

    #[test]
    fn parse_turns_an_error_result_into_err() {
        let out = br#"{"type":"result","is_error":true,"result":"Not logged in"}"#;
        assert_eq!(parse(out).unwrap_err(), "Not logged in");
        assert!(parse(b"not json").is_err());
    }
}
```

- [ ] **Step 2: Test laufen lassen, Scheitern prüfen**

Datei mit den Tests und `pub fn parse(_: &[u8]) -> Result<(String, Option<String>), String> { Err(String::new()) }` sowie `pub fn args(..) -> Vec<String> { vec![] }` anlegen, `mod claude_cli;` in `src-tauri/src/lib.rs` nach `mod claude;` eintragen.

Run: `cargo test -p coucou --lib claude_cli`
Expected: 4 FAIL (Assertions).

- [ ] **Step 3: Implementieren**

`src-tauri/Cargo.toml:23`: tokio-Features um `"process"` ergänzen:

```toml
tokio = { version = "1", features = ["net", "io-util", "sync", "time", "rt", "macros", "process"] }
```

`src-tauri/src/claude_cli.rs` (über den Tests):

```rust
// `claude -p` as a chat provider and as the voice brain (Linux fork). It runs
// on the user's own Claude Code login, so no API key. --safe-mode keeps the
// user's hooks (Coucou's own relay included), CLAUDE.md, plugins and MCP
// servers out; --tools "" leaves the model nothing to run. The prompt goes in
// on stdin, never as an argument.

use std::path::PathBuf;
use std::process::Stdio;
use std::sync::Mutex;
use std::time::Duration;

use serde_json::{json, Value};
use tokio::io::AsyncWriteExt;
use tokio::process::Command;

use crate::chat::{self, Chat, ChatContext, ChatReply};

pub const PROVIDER: &str = "claude-cli";
const TIMEOUT: Duration = Duration::from_secs(60);

/// The chat's Claude Code session, resumed turn after turn.
static CHAT_SESSION: Mutex<Option<String>> = Mutex::new(None);

pub fn args(system: &str, model: &str, resume: Option<&str>, persist: bool) -> Vec<String> {
    let mut a: Vec<String> =
        ["-p", "--safe-mode", "--tools", "", "--output-format", "json"].into_iter().map(String::from).collect();
    if !model.is_empty() {
        a.extend(["--model".into(), model.into()]);
    }
    if !system.is_empty() {
        a.extend(["--system-prompt".into(), system.into()]);
    }
    if let Some(id) = resume {
        a.extend(["--resume".into(), id.into()]);
    }
    if !persist {
        a.push("--no-session-persistence".into());
    }
    a
}

/// `--output-format json` → (answer, session id).
pub fn parse(stdout: &[u8]) -> Result<(String, Option<String>), String> {
    let v: Value = serde_json::from_slice(stdout).map_err(|e| format!("claude: unreadable answer ({e})"))?;
    let result = v.get("result").and_then(Value::as_str).unwrap_or_default().trim().to_string();
    if v.get("is_error").and_then(Value::as_bool) == Some(true) {
        return Err(if result.is_empty() { "claude: error".into() } else { result });
    }
    if result.is_empty() {
        return Err("claude: empty answer".into());
    }
    Ok((result, v.get("session_id").and_then(Value::as_str).map(str::to_string)))
}

/// A desktop launch often has a shorter PATH than the shell: ~/.local/bin too.
fn claude_bin() -> Option<PathBuf> {
    crate::platform::find_on_path("claude").or_else(|| {
        let p = PathBuf::from(std::env::var_os("HOME")?).join(".local/bin/claude");
        p.is_file().then_some(p)
    })
}

fn workdir() -> PathBuf {
    let home = std::env::var_os("HOME").map(PathBuf::from).unwrap_or_else(std::env::temp_dir);
    home.join(".local/share/coucou/brain")
}

pub async fn run(args: Vec<String>, prompt: &str) -> Result<(String, Option<String>), String> {
    let bin = claude_bin().ok_or("Claude Code (claude) not found")?;
    let dir = workdir();
    std::fs::create_dir_all(&dir).map_err(|e| format!("claude: {e}"))?;
    let mut child = Command::new(bin)
        .args(&args)
        .current_dir(&dir)
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .kill_on_drop(true)
        .spawn()
        .map_err(|e| format!("claude: {e}"))?;
    let mut stdin = child.stdin.take().ok_or("claude: no stdin")?;
    stdin.write_all(prompt.as_bytes()).await.map_err(|e| format!("claude: {e}"))?;
    drop(stdin);
    let out = tokio::time::timeout(TIMEOUT, child.wait_with_output())
        .await
        .map_err(|_| "claude: no answer within 60 s".to_string())?
        .map_err(|e| format!("claude: {e}"))?;
    if out.stdout.is_empty() {
        return Err(format!("claude: {}", String::from_utf8_lossy(&out.stderr).trim()));
    }
    parse(&out.stdout)
}

/// One chat turn through the user's Claude Code login.
pub async fn send(chat: &Chat, model: &str, query: String, context: Option<ChatContext>) -> Result<ChatReply, String> {
    let turn = chat.begin(PROVIDER);
    if turn.first {
        *CHAT_SESSION.lock().unwrap() = None;
    }
    let prompt = chat::plain_question(turn.first, context.as_ref(), &query);
    let resume = CHAT_SESSION.lock().unwrap().clone();
    let (text, session) = run(args(&chat::system_prompt(false), model, resume.as_deref(), true), &prompt).await?;
    if session.is_some() {
        *CHAT_SESSION.lock().unwrap() = session;
    }
    chat.commit(&turn, json!({ "role": "user", "content": prompt }), json!({ "role": "assistant", "content": text }), &prompt, &text);
    Ok(ChatReply { text })
}
```

**Wenn `turn.first` privat ist:** in `src-tauri/src/chat.rs` beim `Turn`-Struct `first` auf `pub first` setzen (einzige erlaubte Änderung dort außer Step 3b).

Step 3b – `src-tauri/src/chat.rs`, in `send` direkt nach `let model = model_for(settings, provider);`:

```rust
    if provider == crate::claude_cli::PROVIDER {
        return crate::claude_cli::send(chat, &model, query, context).await;
    }
```

Step 3c – `src-tauri/src/lib.rs`, neben `chat_send`:

```rust
/// One stateless question to `claude -p` (the voice brain). Nothing is kept.
#[tauri::command]
async fn brain_ask(system: String, query: String, model: String) -> Result<String, String> {
    claude_cli::run(claude_cli::args(&system, &model, None, false), &query).await.map(|(text, _)| text)
}
```

und `brain_ask,` in die Liste von `tauri::generate_handler![` eintragen.

Step 3d – `src/core/providers.ts`, in `PROVIDERS` nach `anthropic`:

```ts
  { id: "claude-cli", name: N_("Claude Code (subscription)"), accent: "#D97757", key: null, urlField: null, defaultModel: "sonnet", prefer: null },
```

Step 3e – `src/core/bridge.ts`, nach `approvalDecline`:

```ts
  // ── Voice (Linux fork) ────────────────────────────────────────────────────
  /** One stateless question to `claude -p --safe-mode`; throws with the reason. */
  brainAsk: (system: string, query: string, model: string) => callOrThrow<string>("brain_ask", { system, query, model }),
```

Neuer i18n-String `"Claude Code (subscription)"`: Eintrag in `src/i18n/strings.json` nach dem Muster der vorhandenen Einträge (Deutsch: „Claude Code (Abo)"; andere Sprachen englischen Text übernehmen). **Wenn ein Provider-Picker eine Modell-Liste per Befehl lädt** (`chat.ts` Picker): für `claude-cli` die feste Liste `["haiku", "sonnet", "opus"]` liefern – Stelle im Picker suchen, wo `ollama`/`lmstudio` ihre Modelle laden, und dort einen Zweig `if (p.id === "claude-cli") return ["haiku", "sonnet", "opus"];` ergänzen.

- [ ] **Step 4: Tests grün**

Run: `cargo test -p coucou --lib claude_cli && npm test`
Expected: 4 PASS; Frontend-Tests grün (i18n-Test findet den neuen String).

- [ ] **Step 5: Manuell**

`npm run tauri dev` → Chat → Anbieter „Claude Code (Abo)" wählen → „Wie heißt die Hauptstadt von Tirol?" → Antwort erscheint. Zweite Frage „Und wie viele Einwohner?" → Antwort bezieht sich auf Innsbruck (Session wird fortgesetzt).

- [ ] **Step 6: Commit**

```bash
git add src-tauri/src/claude_cli.rs src-tauri/src/chat.rs src-tauri/src/lib.rs src-tauri/Cargo.toml Cargo.lock src/core/providers.ts src/core/bridge.ts src/i18n/strings.json src/views/chat.ts
git commit -m "feat(linux): claude -p as chat provider and voice brain"
```

---

### Task 3: Push-to-talk-Shortcut mit Drücken und Loslassen

**Files:**
- Modify: `src-tauri/src/shortcuts.rs:60-71` (`ACTIONS`), `:82` (`description`), `:307-320` (Handler), `:323-330` (`dispatch`), `:524-530` (`wayland::on_event`)
- Modify: `src-tauri/src/portal.rs:230-239` (`Event`), `:513-528` (`message`), Fake-Portal-Test `:832`
- Modify: `src/core/shortcuts.ts:21-83` (`SHORTCUT_TEXT`, `ShortcutId`, `SHORTCUTS`)
- Modify: `src/island/shortcuts.ts:50` (`runGlobalShortcut`), `:192` (`registerShortcutHandlers`)
- Modify: `src/i18n/strings.json`
- Test: `tests/shortcuts.test.mjs`

**Interfaces:**
- Produces (Event an das Island-Fenster): `"shortcut"` mit `"pushToTalk"` beim Drücken; **neu** `"shortcut-release"` mit `"pushToTalk"` beim Loslassen.
- Produces (TS): `runGlobalShortcut` ruft für `pushToTalk` `host.voicePress()`; das Release-Event ruft `host.voiceRelease()`. `ShortcutHost` bekommt `voicePress(): void` und `voiceRelease(): void` (implementiert in Task 9).

- [ ] **Step 1: Failing Tests** – in `tests/shortcuts.test.mjs` ergänzen (beim vorhandenen Test „a global shortcut arrives as an event…", gleicher Host-Stub):

```js
test("push-to-talk hands press and release to the voice controller", () => {
  const seen = [];
  const host = { ...hostStub(), voicePress: () => seen.push("press"), voiceRelease: () => seen.push("release") };
  registerShortcutHandlers(host, () => {});
  emit("shortcut", "pushToTalk");
  emit("shortcut-release", "pushToTalk");
  emit("shortcut-release", "openChat"); // only push-to-talk has a release
  assert.deepEqual(seen, ["press", "release"]);
});
```

**Wenn es keinen `hostStub()` gibt:** den Host-Stub des bestehenden Tests „a global shortcut arrives as an event and opens the chat" (Zeile ~320) in eine Funktion `hostStub()` am Dateianfang herausziehen und beide Tests sie nutzen lassen.

Rust – im Fake-Portal (`portal.rs` ~:832) nach dem `Activated`-Signal zusätzlich senden:

```rust
                    let release = Message::new_signal(OBJECT, IFACE, "Deactivated")
                        .unwrap()
                        .append3(session.into_static(), list[0].0.clone(), 0u64)
                        .append1(PropMap::new());
                    ch.send(release).unwrap();
```

und im Test `the_whole_exchange_with_a_stand_in_portal` nach der Erwartung für `Event::Activated(..)` ergänzen:

```rust
        assert_eq!(recv(&rx), Event::Deactivated(want_first_id.clone()));
```

wobei `want_first_id` die ID ist, die der Test für `Activated` erwartet (die vorhandene `assert_eq!` für `Activated` zeigt sie; dieselbe ID verwenden).

- [ ] **Step 2: Scheitern prüfen**

Run: `node --import ./tests/setup.mjs --test tests/shortcuts.test.mjs`
Expected: FAIL (push-to-talk). Rust: `dbus-run-session -- cargo test -p coucou --lib portal -- --ignored` → Compile-Fehler `Event::Deactivated` fehlt.

- [ ] **Step 3: Implementieren**

`portal.rs`, `enum Event`, nach `Activated(String)`:

```rust
    /// A shortcut was let go (push-to-talk).
    Deactivated(String),
```

`portal.rs`, `message()`, nach dem `(IFACE, "Activated")`-Arm:

```rust
            (IFACE, "Deactivated") => {
                let Phase::Bound { session, .. } = &self.phase else { return };
                if let Ok((from, id)) = msg.read2::<Path, String>() {
                    if &*from == session.as_str() {
                        on_event(Event::Deactivated(id));
                    }
                }
            }
```

**Wenn das Portal Signale per Match-Regel abonniert** (Suche nach `add_match`/`match_rule` bzw. `member` in `portal.rs`): dort `Deactivated` genauso wie `Activated` eintragen.

`shortcuts.rs`, `ACTIONS` – am Ende anfügen:

```rust
    // Linux fork: hold to talk (press starts recording, release stops it).
    action("pushToTalk", "Ctrl+Alt+V", true, true),
```

`shortcuts.rs`, `description()` – Arm ergänzen:

```rust
        "pushToTalk" => n_("Hold to talk to Coucou"),
```

`shortcuts.rs`, Plugin-Handler (ersetzt den Körper von `.with_handler`):

```rust
        .with_handler(|app, shortcut, event| {
            let Some(registry) = app.try_state::<Registry>() else { return };
            let action = registry.by_id.lock().unwrap().get(&shortcut.id()).copied();
            let Some(action) = action else { return };
            if event.state == ShortcutState::Pressed {
                dispatch(app, action);
            } else if action == "pushToTalk" {
                dispatch_release(app, action);
            }
        })
```

`shortcuts.rs`, nach `dispatch`:

```rust
/// A held shortcut was let go. Only push-to-talk listens for this.
pub fn dispatch_release<R: Runtime>(app: &AppHandle<R>, action: &str) {
    let _ = app.emit_to(WINDOW_LABEL, "shortcut-release", action.to_string());
}
```

`shortcuts.rs`, `wayland::on_event`, vor `if let Event::Activated(id) = &event {`:

```rust
        if let Event::Deactivated(id) = &event {
            if id == "pushToTalk" {
                dispatch_release(app, "pushToTalk");
            }
            return;
        }
```

`src/core/shortcuts.ts`: in `SHORTCUT_TEXT` `pushToTalk: N_("Hold to talk to Coucou"),`; in `ShortcutId` `| "pushToTalk"`; in `SHORTCUTS` am Ende `def("pushToTalk", "Ctrl+Alt+V", true, true),`.

`src/island/shortcuts.ts`: `ShortcutHost` um

```ts
  /** Push-to-talk pressed / let go (src/island/voice.ts). */
  voicePress(): void;
  voiceRelease(): void;
```

ergänzen; in `runGlobalShortcut` einen Fall

```ts
    case "pushToTalk":
      host.voicePress();
      break;
```

und in `registerShortcutHandlers` nach dem `"shortcut"`-Listener:

```ts
  void onEvent<string>("shortcut-release", (action) => {
    if (action === "pushToTalk") host.voiceRelease();
  });
```

Die Klasse, die `ShortcutHost` implementiert (`src/island/island.ts`, Suche nach `wardrobeAnywhere()`), bekommt vorläufig leere Methoden `voicePress() {}` und `voiceRelease() {}`; Task 9 füllt sie.

`src/i18n/strings.json`: `"Hold to talk to Coucou"` (Deutsch: „Halten, um mit Coucou zu sprechen").

- [ ] **Step 4: Tests grün**

Run: `npm test && cargo test -p coucou --lib shortcuts && dbus-run-session -- cargo test -p coucou --lib portal -- --ignored`
Expected: alles PASS, inklusive des Tabellen-Gleichlauf-Tests Rust↔TS und des AltGr-Kollisionstests. **Wenn der AltGr-Test `Ctrl+Alt+V` ablehnt:** `Ctrl+Alt+B` in Rust und TS verwenden und in der Spec nachtragen.

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/shortcuts.rs src-tauri/src/portal.rs src/core/shortcuts.ts src/island/shortcuts.ts src/island/island.ts src/i18n/strings.json tests/shortcuts.test.mjs
git commit -m "feat(linux): push-to-talk shortcut with press and release"
```

---

### Task 4: `voice.rs` – Aufnahme, Transkription, Sprachausgabe, Check

**Files:**
- Create: `src-tauri/src/voice.rs`
- Modify: `src-tauri/Cargo.toml:24` (reqwest-Feature `multipart`)
- Modify: `src-tauri/src/lib.rs` (`mod voice;`, Befehle, `generate_handler!`)
- Modify: `src/core/bridge.ts`

**Interfaces:**
- Consumes: nichts aus anderen Tasks. Modell- und Stimmpfad kommen als Befehlsargumente aus dem Frontend (Task 5 liefert die Settings-Felder dafür).
- Produces (Rust): `voice::start() -> Result<(), String>`; `voice::stop() -> Option<PathBuf>`; `voice::ensure_server(model: &str) -> Result<(), String>`; `voice::transcribe(wav: PathBuf, base: &str) -> Result<String, String>` (async); `voice::speak(text: &str, voice_model: &str) -> Result<(), String>`; `voice::check(model: &str, voice_model: &str) -> Vec<String>`; `voice::record_args`, `voice::server_args`.
- Produces (Tauri): `voice_record_start() -> Result<(), String>`; `voice_record_stop(model: String) -> Result<Option<String>, String>` (None = zu kurz); `voice_speak(text: String, voice: String) -> Result<(), String>`; `voice_check(model: String, voice: String) -> Vec<String>`.
- Produces (TS): `Bridge.voiceRecordStart(): Promise<void>`, `Bridge.voiceRecordStop(model: string): Promise<string | null>`, `Bridge.voiceSpeak(text: string, voice: string): Promise<void>`, `Bridge.voiceCheck(model: string, voice: string): Promise<string[]>`.

- [ ] **Step 1: Failing Tests** (am Ende von `src-tauri/src/voice.rs`)

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::io::{Read, Write};
    use std::net::TcpListener;

    #[test]
    fn record_args_ask_for_16k_mono_s16() {
        let a = record_args(Path::new("/run/user/1000/x.wav"));
        assert_eq!(a, ["--rate", "16000", "--channels", "1", "--format", "s16", "/run/user/1000/x.wav"]);
    }

    #[test]
    fn server_args_bind_localhost_german() {
        let a = server_args("/m/ggml.bin", 4242);
        for pair in [["--model", "/m/ggml.bin"], ["--host", "127.0.0.1"], ["--port", "4242"], ["--language", "de"]] {
            assert!(a.windows(2).any(|w| w == pair), "{pair:?}");
        }
        assert!(a.contains(&"--no-timestamps".to_string()));
    }

    #[test]
    fn check_names_what_is_missing() {
        let missing = check("/nope/model.bin", "/nope/voice.onnx");
        assert!(missing.iter().any(|m| m.contains("/nope/model.bin")));
        assert!(missing.iter().any(|m| m.contains("/nope/voice.onnx")));
    }

    #[test]
    fn wait_for_port_waits_for_a_late_server() {
        let port = 41_877;
        std::thread::spawn(move || {
            std::thread::sleep(Duration::from_millis(600));
            let l = TcpListener::bind(("127.0.0.1", port)).unwrap();
            let _ = l.accept();
        });
        assert!(wait_for_port(port, Duration::from_secs(5)));
        assert!(!wait_for_port(41_878, Duration::from_millis(300)));
    }

    #[test]
    fn transcribe_posts_the_wav_and_reads_text() {
        let l = TcpListener::bind("127.0.0.1:0").unwrap();
        let base = format!("http://{}", l.local_addr().unwrap());
        let server = std::thread::spawn(move || {
            let (mut s, _) = l.accept().unwrap();
            let mut buf = vec![0u8; 65_536];
            let n = s.read(&mut buf).unwrap();
            let req = String::from_utf8_lossy(&buf[..n]).to_string();
            let body = r#"{"text":" Erlauben. "}"#;
            write!(s, "HTTP/1.1 200 OK\r\nContent-Type: application/json\r\nContent-Length: {}\r\n\r\n{body}", body.len()).unwrap();
            req
        });
        let dir = std::env::temp_dir().join(format!("coucou-voice-test-{}", std::process::id()));
        std::fs::create_dir_all(&dir).unwrap();
        let wav = dir.join("t.wav");
        std::fs::write(&wav, b"RIFF....WAVE").unwrap();
        let rt = tokio::runtime::Builder::new_current_thread().enable_all().build().unwrap();
        let text = rt.block_on(transcribe(wav.clone(), &base)).unwrap();
        assert_eq!(text, "Erlauben.");
        assert!(!wav.exists(), "the recording is deleted once read");
        let req = server.join().unwrap();
        assert!(req.starts_with("POST /inference "));
        assert!(req.contains("multipart/form-data"));
    }
}
```

- [ ] **Step 2: Scheitern prüfen**

Datei mit Tests anlegen, `mod voice;` in `lib.rs` nach `mod tray;` eintragen.
Run: `cargo test -p coucou --lib voice`
Expected: Compile-Fehler (Funktionen fehlen).

- [ ] **Step 3: Implementieren**

`src-tauri/Cargo.toml:24`:

```toml
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls", "multipart"] }
```

`src-tauri/src/voice.rs` (über den Tests):

```rust
// Push-to-talk for the Linux fork: pw-record captures, a local whisper-server
// (whisper.cpp, model loaded once) transcribes, piper + pw-play speak. Audio
// never leaves the machine; recordings live in $XDG_RUNTIME_DIR and are
// deleted once read.

use std::net::TcpStream;
use std::path::{Path, PathBuf};
use std::process::{Child, Command, Stdio};
use std::sync::Mutex;
use std::time::{Duration, Instant};

use serde_json::Value;

const WHISPER_SERVER: &str = "whisper-server";
const PIPER_NAMES: [&str; 2] = ["piper", "piper-tts"];
pub const PORT: u16 = 18_765;
const MIN_SECONDS: f64 = 0.4;
const SERVER_START: Duration = Duration::from_secs(30);

/// The running pw-record and when it started.
static RECORDING: Mutex<Option<(Child, Instant)>> = Mutex::new(None);
/// whisper-server, started on first use, kept for the session.
static SERVER: Mutex<Option<Child>> = Mutex::new(None);

fn runtime_dir() -> PathBuf {
    std::env::var_os("XDG_RUNTIME_DIR").map(PathBuf::from).unwrap_or_else(std::env::temp_dir)
}

fn wav_path() -> PathBuf {
    runtime_dir().join("coucou-voice.wav")
}

fn quiet(cmd: &mut Command) -> &mut Command {
    cmd.stdin(Stdio::null()).stdout(Stdio::null()).stderr(Stdio::null())
}

pub fn record_args(path: &Path) -> Vec<String> {
    let mut a: Vec<String> =
        ["--rate", "16000", "--channels", "1", "--format", "s16"].into_iter().map(String::from).collect();
    a.push(path.display().to_string());
    a
}

pub fn server_args(model: &str, port: u16) -> Vec<String> {
    vec![
        "--model".into(), model.into(),
        "--host".into(), "127.0.0.1".into(),
        "--port".into(), port.to_string(),
        "--language".into(), "de".into(),
        "--no-timestamps".into(),
    ]
}

pub fn start() -> Result<(), String> {
    let mut rec = RECORDING.lock().unwrap();
    if rec.is_some() {
        return Ok(());
    }
    let _ = std::fs::remove_file(wav_path());
    let child = quiet(Command::new("pw-record").args(record_args(&wav_path())))
        .spawn()
        .map_err(|e| format!("pw-record: {e}"))?;
    *rec = Some((child, Instant::now()));
    Ok(())
}

/// Stops the recording. None when there was none or it was too short.
pub fn stop() -> Option<PathBuf> {
    let (mut child, at) = RECORDING.lock().unwrap().take()?;
    // SIGINT lets pw-record finish the WAV header; a kill would leave it broken.
    unsafe { libc::kill(child.id() as libc::pid_t, libc::SIGINT) };
    let _ = child.wait();
    if at.elapsed().as_secs_f64() < MIN_SECONDS {
        let _ = std::fs::remove_file(wav_path());
        return None;
    }
    Some(wav_path())
}

pub fn wait_for_port(port: u16, limit: Duration) -> bool {
    let deadline = Instant::now() + limit;
    while Instant::now() < deadline {
        if TcpStream::connect_timeout(&([127, 0, 0, 1], port).into(), Duration::from_millis(200)).is_ok() {
            return true;
        }
        std::thread::sleep(Duration::from_millis(100));
    }
    false
}

/// Starts whisper-server unless it runs, then waits until it listens (the
/// model takes a few seconds to load the first time).
pub fn ensure_server(model: &str) -> Result<(), String> {
    let mut server = SERVER.lock().unwrap();
    let alive = server.as_mut().is_some_and(|c| matches!(c.try_wait(), Ok(None)));
    if !alive {
        let child = quiet(Command::new(WHISPER_SERVER).args(server_args(model, PORT)))
            .spawn()
            .map_err(|e| format!("{WHISPER_SERVER}: {e}"))?;
        crate::log::line("voice: whisper-server started");
        *server = Some(child);
    }
    drop(server);
    if wait_for_port(PORT, SERVER_START) { Ok(()) } else { Err(format!("{WHISPER_SERVER} did not start")) }
}

pub fn base_url() -> String {
    format!("http://127.0.0.1:{PORT}")
}

pub async fn transcribe(wav: PathBuf, base: &str) -> Result<String, String> {
    let bytes = tokio::fs::read(&wav).await.map_err(|e| format!("recording: {e}"));
    let _ = tokio::fs::remove_file(&wav).await;
    let part = reqwest::multipart::Part::bytes(bytes?).file_name("audio.wav").mime_str("audio/wav").map_err(|e| e.to_string())?;
    let form = reqwest::multipart::Form::new()
        .part("file", part)
        .text("response_format", "json")
        .text("language", "de");
    let response = reqwest::Client::builder()
        .no_proxy()
        .timeout(Duration::from_secs(60))
        .build()
        .map_err(|e| e.to_string())?
        .post(format!("{base}/inference"))
        .multipart(form)
        .send()
        .await
        .map_err(|e| format!("whisper: {e}"))?;
    let v: Value = response.json().await.map_err(|e| format!("whisper: {e}"))?;
    Ok(v.get("text").and_then(Value::as_str).unwrap_or_default().trim().to_string())
}

fn piper() -> Option<PathBuf> {
    PIPER_NAMES.iter().find_map(|n| crate::platform::find_on_path(n))
}

/// Speaks `text` in the background: piper writes a WAV, pw-play plays it.
pub fn speak(text: &str, voice_model: &str) -> Result<(), String> {
    let piper = piper().ok_or("piper not found")?;
    let out = runtime_dir().join("coucou-say.wav");
    let (text, voice_model) = (text.to_string(), voice_model.to_string());
    std::thread::spawn(move || {
        use std::io::Write;
        let Ok(mut child) = Command::new(piper)
            .args(["--model", &voice_model, "--output_file", &out.display().to_string()])
            .stdin(Stdio::piped()).stdout(Stdio::null()).stderr(Stdio::null())
            .spawn() else { return };
        if let Some(mut stdin) = child.stdin.take() {
            let _ = stdin.write_all(text.as_bytes());
        }
        if child.wait().is_ok_and(|s| s.success()) {
            let _ = quiet(Command::new("pw-play").arg(&out)).status();
        }
        let _ = std::fs::remove_file(&out);
    });
    Ok(())
}

/// What the voice feature still needs, in words Settings can show.
pub fn check(model: &str, voice_model: &str) -> Vec<String> {
    let mut missing = Vec::new();
    for bin in ["pw-record", "pw-play", WHISPER_SERVER] {
        if crate::platform::find_on_path(bin).is_none() {
            missing.push(bin.to_string());
        }
    }
    if piper().is_none() {
        missing.push("piper".into());
    }
    for file in [model, voice_model] {
        if !Path::new(file).is_file() {
            missing.push(file.to_string());
        }
    }
    missing
}

/// Stops whisper-server when Coucou quits.
pub fn shutdown() {
    if let Some(mut c) = SERVER.lock().unwrap().take() {
        let _ = c.kill();
        let _ = c.wait();
    }
}
```

`src-tauri/src/lib.rs` – Befehle neben `brain_ask`:

```rust
#[tauri::command]
fn voice_record_start() -> Result<(), String> {
    voice::start()
}

/// Stops the recording and transcribes it. None when it was too short.
#[tauri::command]
async fn voice_record_stop(model: String) -> Result<Option<String>, String> {
    let Some(wav) = voice::stop() else { return Ok(None) };
    tauri::async_runtime::spawn_blocking(move || voice::ensure_server(&model))
        .await
        .map_err(|e| e.to_string())??;
    voice::transcribe(wav, &voice::base_url()).await.map(Some)
}

#[tauri::command]
fn voice_speak(text: String, voice: String) -> Result<(), String> {
    voice::speak(&text, &voice)
}

#[tauri::command]
fn voice_check(model: String, voice: String) -> Vec<String> {
    voice::check(&model, &voice)
}
```

In `generate_handler!` `voice_record_start, voice_record_stop, voice_speak, voice_check,` eintragen. `voice::shutdown()` dort aufrufen, wo die App beim Beenden aufräumt (Suche in `lib.rs` nach `RunEvent::Exit` bzw. `ExitRequested`; **wenn es keinen solchen Zweig gibt:** in `.run(|app, event| …)` einen `if let tauri::RunEvent::Exit = event { voice::shutdown(); }` ergänzen).

`src/core/bridge.ts`, unter `brainAsk`:

```ts
  voiceRecordStart: () => callOrThrow<void>("voice_record_start"),
  /** The transcript, or null when the recording was too short. */
  voiceRecordStop: (model: string) => callOrThrow<string | null>("voice_record_stop", { model }),
  voiceSpeak: (text: string, voice: string) => call<void>("voice_speak", { text, voice }),
  /** What is missing for voice: commands or model files. Empty when ready. */
  voiceCheck: (model: string, voice: string) => call<string[]>("voice_check", { model, voice }),
```

- [ ] **Step 4: Tests grün**

Run: `cargo test -p coucou --lib voice`
Expected: 5 PASS.

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/voice.rs src-tauri/src/lib.rs src-tauri/Cargo.toml Cargo.lock src/core/bridge.ts
git commit -m "feat(linux): voice capture, local whisper transcription and piper speech"
```

---

### Task 5: Einstellungen „Sprache"

**Files:**
- Modify: `src-tauri/src/settings.rs:15` (`Settings`), `:124` (`Default`), Test `defaults()` ~:473
- Modify: `src/core/state.ts:100` (`Settings`), `~:171` (`DEFAULT_SETTINGS`)
- Modify: `src/settings/main.ts` (neuer Abschnitt, in `render()` ~:1321 anhängen)
- Modify: `src/i18n/strings.json`
- Test: `tests/settings.test.mjs` (falls vorhanden, sonst neuer `tests/voice-settings.test.mjs`)

**Interfaces:**
- Produces (Settings, camelCase in TS): `voiceEnabled: boolean` (false), `voiceWhisperModel: string`, `voicePiperVoice: string`, `voiceSpeakAnswers: boolean` (true), `voiceBrainModel: string` ("haiku").
- Defaults der Pfade: Rust berechnet `$HOME/.local/share/coucou/models/ggml-large-v3-turbo-q5_0.bin` und `$HOME/.local/share/coucou/models/de_DE-thorsten-medium.onnx`; TS-Default ist `""` (Rust liefert beim `boot` die gespeicherten bzw. Default-Werte).

- [ ] **Step 1: Failing Test** – Rust, im bestehenden `defaults()`-Test ergänzen:

```rust
        assert!(!s.voice_enabled);
        assert!(s.voice_speak_answers);
        assert_eq!(s.voice_brain_model, "haiku");
        assert!(s.voice_whisper_model.ends_with(".local/share/coucou/models/ggml-large-v3-turbo-q5_0.bin"));
        assert!(s.voice_piper_voice.ends_with(".local/share/coucou/models/de_DE-thorsten-medium.onnx"));
```

und ein neuer Test daneben:

```rust
    #[test]
    fn older_settings_files_load_with_voice_off() {
        let s: Settings = serde_json::from_str(r#"{"soundEnabled":true}"#).unwrap();
        assert!(!s.voice_enabled);
        assert_eq!(s.voice_brain_model, "haiku");
    }
```

- [ ] **Step 2: Scheitern prüfen**

Run: `cargo test -p coucou --lib settings`
Expected: Compile-Fehler (Felder fehlen).

- [ ] **Step 3: Implementieren**

`settings.rs`, am Ende von `pub struct Settings`:

```rust
    /// Linux fork: push-to-talk voice control. Off until turned on in Settings.
    pub voice_enabled: bool,
    /// whisper.cpp model file (ggml), per machine.
    pub voice_whisper_model: String,
    /// Piper voice (.onnx, its .onnx.json beside it).
    pub voice_piper_voice: String,
    /// Speak answers and confirmations, not only show them.
    pub voice_speak_answers: bool,
    /// Model for the voice brain (`claude -p --model`).
    pub voice_brain_model: String,
```

Hilfsfunktion über `impl Default for Settings`:

```rust
fn voice_file(name: &str) -> String {
    let home = std::env::var("HOME").unwrap_or_default();
    format!("{home}/.local/share/coucou/models/{name}")
}
```

In `Default::default()`:

```rust
            voice_enabled: false,
            voice_whisper_model: voice_file("ggml-large-v3-turbo-q5_0.bin"),
            voice_piper_voice: voice_file("de_DE-thorsten-medium.onnx"),
            voice_speak_answers: true,
            voice_brain_model: "haiku".into(),
```

`src/core/state.ts`, `interface Settings`:

```ts
  voiceEnabled: boolean;
  voiceWhisperModel: string;
  voicePiperVoice: string;
  voiceSpeakAnswers: boolean;
  voiceBrainModel: string;
```

`DEFAULT_SETTINGS`:

```ts
  voiceEnabled: false,
  voiceWhisperModel: "",
  voicePiperVoice: "",
  voiceSpeakAnswers: true,
  voiceBrainModel: "haiku",
```

`src/settings/main.ts`, neue Funktion nach `generalSection()` (gleiche Helfer `h`, `toggle`, `t`, `save` wie dort):

```ts
/** Linux fork: push-to-talk voice control. */
function voiceSection(): HTMLElement {
  const status = h("div", { class: "row-note" });
  const refresh = () => {
    void Bridge.voiceCheck(settings.voiceWhisperModel, settings.voicePiperVoice).then((missing) => {
      status.textContent = missing.length === 0
        ? t("Ready: hold the push-to-talk shortcut and speak.")
        : `${t("Missing:")} ${missing.join(", ")} — paru -S whisper-cpp piper-tts-bin`;
    });
  };
  const field = (value: string, set: (v: string) => void) => {
    const input = h("input", { type: "text", class: "text-field", value, spellcheck: "false" }) as HTMLInputElement;
    input.addEventListener("change", () => { set(input.value.trim()); void save(); refresh(); });
    return input;
  };
  refresh();
  return h("section", { class: "section" },
    h("h2", { text: t("Voice") }),
    h("div", { class: "row" }, h("label", { text: t("Voice control") }), toggle(settings.voiceEnabled, (v) => { settings.voiceEnabled = v; void save(); })),
    h("div", { class: "row" }, h("label", { text: t("Speak answers") }), toggle(settings.voiceSpeakAnswers, (v) => { settings.voiceSpeakAnswers = v; void save(); })),
    h("div", { class: "row" }, h("label", { text: t("Whisper model") }), field(settings.voiceWhisperModel, (v) => { settings.voiceWhisperModel = v; })),
    h("div", { class: "row" }, h("label", { text: t("Piper voice") }), field(settings.voicePiperVoice, (v) => { settings.voicePiperVoice = v; })),
    h("div", { class: "row" }, h("label", { text: t("Brain model") }), field(settings.voiceBrainModel, (v) => { settings.voiceBrainModel = v; })),
    status,
  );
}
```

**Wenn `generalSection()` andere Klassennamen für Abschnitt, Überschrift oder Textfelder nutzt** (`section`, `h2`, `text-field`, `row-note`): dessen Klassennamen übernehmen – keine neuen CSS-Regeln anlegen. In `render()` `voiceSection()` nach den vorhandenen Abschnitten anhängen, nur wenn `State.os === "linux"` bzw. die dort übliche OS-Abfrage Linux meldet (Suche nach `"linux"` in `main.ts`).

`src/i18n/strings.json`: `"Voice"` → „Sprache", `"Voice control"` → „Sprachsteuerung", `"Speak answers"` → „Antworten vorlesen", `"Whisper model"` → „Whisper-Modell", `"Piper voice"` → „Piper-Stimme", `"Brain model"` → „Modell für Antworten", `"Ready: hold the push-to-talk shortcut and speak."` → „Bereit: Push-to-talk-Taste halten und sprechen.", `"Missing:"` → „Fehlt:".

- [ ] **Step 4: Tests grün**

Run: `cargo test -p coucou --lib settings && npm test`
Expected: PASS (i18n-Test kennt alle neuen Strings).

- [ ] **Step 5: Manuell**

`npm run tauri dev` → Settings → Abschnitt „Sprache" zeigt „Bereit…" (nach Task 0). Modellpfad testweise auf `/x` ändern → „Fehlt: /x …". Zurücksetzen.

- [ ] **Step 6: Commit**

```bash
git add src-tauri/src/settings.rs src/core/state.ts src/settings/main.ts src/i18n/strings.json
git commit -m "feat(linux): voice settings section with readiness check"
```

---

### Task 6: Diktat im Chat (Mikrofon-Knopf)

**Files:**
- Modify: `src/views/chat.ts:204-216` (`buildPrompt`)
- Modify: `src/views/icons.ts` (Icon `mic`)
- Modify: `src/style.css` (nur neue Klasse `.mic-btn.recording`)
- Test: `tests/chat.test.mjs`

**Interfaces:**
- Consumes: `Bridge.voiceRecordStart`, `Bridge.voiceRecordStop(model)` (Task 4), `State.settings.voiceEnabled`, `State.settings.voiceWhisperModel` (Task 5).
- Produces: Knopf `.mic-btn` in `.chat-bar` zwischen Eingabefeld und Senden, nur sichtbar bei `voiceEnabled`. Klick 1 startet, Klick 2 stoppt; Transkript wird an `input.value` angehängt.

- [ ] **Step 1: Failing Tests** – in `tests/chat.test.mjs`:

```js
test("the mic dictates into the chat field when voice is on", async () => {
  State.settings = { ...State.settings, voiceEnabled: true, voiceWhisperModel: "/m.bin" };
  answers.voice_record_stop = "Wie spät ist es?";
  view = buildPrompt(() => {});
  const input = view.el.querySelector(".chat-input");
  const mic = view.el.querySelector(".mic-btn");
  input.value = "Frage:";
  mic.click();
  await flush();
  assert.ok(sent("voice_record_start"));
  assert.ok(mic.classList.contains("recording"));
  mic.click();
  await flush();
  assert.deepEqual(sent("voice_record_stop").args, { model: "/m.bin" });
  assert.equal(input.value, "Frage: Wie spät ist es?");
  assert.ok(!mic.classList.contains("recording"));
});

test("no mic without voice", () => {
  State.settings = { ...State.settings, voiceEnabled: false };
  view = buildPrompt(() => {});
  assert.equal(view.el.querySelector(".mic-btn"), null);
});
```

**Wenn `sent(cmd)` in `tests/tauri.mjs` eine andere Form zurückgibt** (z. B. nur die Argumente): die Assertions an diese Form anpassen, die Prüfinhalte bleiben gleich.

- [ ] **Step 2: Scheitern prüfen**

Run: `node --import ./tests/setup.mjs --test tests/chat.test.mjs`
Expected: FAIL („mic-btn" nicht gefunden).

- [ ] **Step 3: Implementieren**

`src/views/icons.ts`, Eintrag in `ICONS` nach dem Muster von `arrowUp`:

```ts
  mic: "M12 3a3 3 0 0 0-3 3v6a3 3 0 0 0 6 0V6a3 3 0 0 0-3-3zM5 11a7 7 0 0 0 14 0M12 18v3",
```

`src/views/chat.ts`, in `buildPrompt` die Zeile `const bar = h("div", { class: "chat-bar" }, input, send);` ersetzen durch:

```ts
  const bar = h("div", { class: "chat-bar" }, input, send);
  if (State.settings.voiceEnabled) {
    // Linux fork: dictation. Click to start, click again to stop and transcribe.
    const mic = h("button", { class: "send-btn mic-btn", title: t("Dictate") }, svg(ICONS.mic, 11));
    let recording = false;
    mic.addEventListener("click", () => {
      if (!recording) {
        recording = true;
        mic.classList.add("recording");
        void Bridge.voiceRecordStart().catch(() => {
          recording = false;
          mic.classList.remove("recording");
        });
        return;
      }
      recording = false;
      mic.classList.remove("recording");
      void Bridge.voiceRecordStop(State.settings.voiceWhisperModel).then((text) => {
        if (!text) return;
        input.value = input.value.trim() ? `${input.value.trim()} ${text}` : text;
        input.focus();
      }).catch(() => Sound.play("error"));
    });
    bar.insertBefore(mic, send);
  }
```

`src/style.css` (neue Regel, bestehende unverändert):

```css
.mic-btn.recording { background: #D97757; }
```

`src/i18n/strings.json`: `"Dictate"` → „Diktieren".

- [ ] **Step 4: Tests grün**

Run: `npm test`
Expected: PASS.

- [ ] **Step 5: Manuell**

Sprache in Settings an → Chat öffnen → Mikrofon → „Wie wird das Wetter morgen in Innsbruck" → Mikrofon → Text steht im Feld.

- [ ] **Step 6: Commit**

```bash
git add src/views/chat.ts src/views/icons.ts src/style.css src/i18n/strings.json tests/chat.test.mjs
git commit -m "feat(linux): dictation button in the chat"
```

---

### Task 7: Session-Register und Router (rein, ohne I/O)

**Files:**
- Create: `src/core/voice.ts`
- Test: `tests/voice.test.mjs`

**Interfaces:**
- Consumes: `ApprovalInfo` aus `src/core/state.ts`.
- Produces:

```ts
export interface VoiceSession { sessionId: string; name: string; cwd: string; status: string; recent: string[]; finalLine: string | null; updated: number; }
export interface HookLike { hook_event_name?: string; session_id?: string; cwd?: string; tool_name?: string; last_assistant_message?: string; coucou_agent?: string; }
export function trackSession(list: VoiceSession[], p: HookLike, now: number): VoiceSession[];
export type VoiceAction =
  | { kind: "approve"; decision: "allow" | "deny" }
  | { kind: "answer"; question: string; label: string }
  | { kind: "instruct"; sessionId: string; name: string; text: string }
  | { kind: "which-session"; names: string[] }
  | { kind: "say"; text: string }
  | { kind: "question"; text: string }
  | { kind: "ignore" };
export function route(transcript: string, pending: ApprovalInfo | null, sessions: VoiceSession[]): VoiceAction;
export function brainContext(sessions: VoiceSession[], pending: ApprovalInfo | null): string;
export const BRAIN_SYSTEM: string;
```

- [ ] **Step 1: Failing Tests** – `tests/voice.test.mjs`:

```js
// Voice router and session register (src/core/voice.ts): pure logic, no I/O.
// The approval cases are the safety net: only a bare yes/no answers a card.

import { test } from "node:test";
import assert from "node:assert/strict";
import { route, trackSession, brainContext } from "../src/core/voice.ts";

const card = { requestId: "r1", sessionId: "s1", pillId: "p", tool: "Bash", command: "npm test" };
const ask = {
  ...card,
  questions: [{ question: "Welche DB?", multiSelect: false, options: [{ label: "SQLite", description: "" }, { label: "Postgres", description: "" }] }],
};
const s = (name, sessionId) => ({ sessionId, name, cwd: `/c/${name}`, status: "Stop", recent: [], finalLine: null, updated: 1 });
const sessions = [s("finanzen", "a"), s("android-home", "b"), s("ha-docs", "c")];

test("a bare yes or no answers the card on screen, whatever the punctuation", () => {
  for (const t of ["Erlauben.", "ja", "Ja!", "okay", "Okay Coucou, erlauben"]) {
    assert.deepEqual(route(t, card, sessions), { kind: "approve", decision: "allow" }, t);
  }
  for (const t of ["Ablehnen.", "nein", "Stopp!"]) {
    assert.deepEqual(route(t, card, sessions), { kind: "approve", decision: "deny" }, t);
  }
});

test("a yes with a condition is never an approval", () => {
  const r = route("Ja, aber nur lesen.", card, sessions);
  assert.notEqual(r.kind, "approve");
});

test("without a card, a yes is just a question", () => {
  assert.deepEqual(route("ja", null, sessions), { kind: "question", text: "ja" });
});

test("a question card takes an option by number, word or label", () => {
  assert.deepEqual(route("Option zwei", ask, sessions), { kind: "answer", question: "Welche DB?", label: "Postgres" });
  assert.deepEqual(route("1", ask, sessions), { kind: "answer", question: "Welche DB?", label: "SQLite" });
  assert.deepEqual(route("Postgres.", ask, sessions), { kind: "answer", question: "Welche DB?", label: "Postgres" });
  assert.equal(route("Option fünf", ask, sessions).kind, "say");
});

test("multi-question or multi-select cards are left to the island", () => {
  const multi = { ...ask, questions: [{ ...ask.questions[0], multiSelect: true }] };
  assert.equal(route("eins", multi, sessions).kind, "say");
});

test("an instruction goes to the named session", () => {
  assert.deepEqual(route("Sag finanzen: Tests laufen lassen.", null, sessions),
    { kind: "instruct", sessionId: "a", name: "finanzen", text: "Tests laufen lassen." });
  assert.deepEqual(route("an Android Home, mach einen Build", null, sessions),
    { kind: "instruct", sessionId: "b", name: "android-home", text: "mach einen Build" });
  assert.deepEqual(route("sag fin lass die tests laufen", null, sessions),
    { kind: "instruct", sessionId: "a", name: "finanzen", text: "lass die tests laufen" });
});

test("an unknown or ambiguous name asks back", () => {
  assert.deepEqual(route("sag xyz: hallo", null, sessions), { kind: "which-session", names: ["finanzen", "android-home", "ha-docs"] });
  const two = [...sessions, s("ha-config", "d")];
  assert.equal(route("sag ha mach was", null, two).kind, "which-session");
});

test("an empty transcript is ignored", () => {
  assert.deepEqual(route("  ", card, sessions), { kind: "ignore" });
});

test("the register follows hook events and drops ended sessions", () => {
  let list = [];
  list = trackSession(list, { hook_event_name: "UserPromptSubmit", session_id: "a", cwd: "/home/stefan/Code/finanzen" }, 1);
  list = trackSession(list, { hook_event_name: "PreToolUse", session_id: "a", cwd: "/home/stefan/Code/finanzen", tool_name: "Bash" }, 2);
  list = trackSession(list, { hook_event_name: "Stop", session_id: "a", last_assistant_message: "Alle Tests grün." }, 3);
  assert.equal(list.length, 1);
  assert.equal(list[0].name, "finanzen");
  assert.equal(list[0].status, "Stop");
  assert.equal(list[0].finalLine, "Alle Tests grün.");
  assert.deepEqual(list[0].recent, ["UserPromptSubmit", "PreToolUse Bash", "Stop"]);
  list = trackSession(list, { hook_event_name: "SessionEnd", session_id: "a" }, 4);
  assert.equal(list.length, 0);
});

test("other agents and events without a session are not tracked", () => {
  assert.deepEqual(trackSession([], { hook_event_name: "Stop", session_id: "x", coucou_agent: "gemini" }, 1), []);
  assert.deepEqual(trackSession([], { hook_event_name: "StatusLine" }, 1), []);
});

test("the brain context names every session and the waiting card", () => {
  const c = brainContext(sessions, card);
  for (const n of ["finanzen", "android-home", "ha-docs", "npm test"]) assert.ok(c.includes(n), n);
});
```

- [ ] **Step 2: Scheitern prüfen**

Run: `node --import ./tests/setup.mjs --test tests/voice.test.mjs`
Expected: FAIL (Modul fehlt).

- [ ] **Step 3: Implementieren** – `src/core/voice.ts`:

```ts
// Linux fork: what a spoken sentence means. Pure: no I/O, no State, so the
// safety rules are tested on their own (tests/voice.test.mjs). The island
// shows every Claude Code session on one pill, so voice keeps its own register
// of the sessions that run, fed by the same hook events.

import type { ApprovalInfo } from "./state";

export interface VoiceSession {
  sessionId: string;
  name: string;
  cwd: string;
  status: string;
  recent: string[];
  finalLine: string | null;
  updated: number;
}

export interface HookLike {
  hook_event_name?: string;
  session_id?: string;
  cwd?: string;
  tool_name?: string;
  last_assistant_message?: string;
  coucou_agent?: string;
}

export type VoiceAction =
  | { kind: "approve"; decision: "allow" | "deny" }
  | { kind: "answer"; question: string; label: string }
  | { kind: "instruct"; sessionId: string; name: string; text: string }
  | { kind: "which-session"; names: string[] }
  | { kind: "say"; text: string }
  | { kind: "question"; text: string }
  | { kind: "ignore" };

const RECENT = 5;
const ALLOW = new Set(["erlauben", "erlaube", "ja", "okay", "ok", "zulassen", "ja erlauben"]);
const DENY = new Set(["ablehnen", "nein", "stopp", "stop", "verweigern", "lehne ab"]);
const NUMBERS: Record<string, number> = { eins: 1, ein: 1, eine: 1, zwei: 2, drei: 3, vier: 4 };
const FILLER = /^(?:(?:ähm|äh|also|hey coucou|okay coucou|ok coucou)(?=[\s,.!]|$)[\s,.!]*)+/;
const INSTRUCT = /^(?:sag|sage|an)\s+(.+)$/;

/** Lower case, no punctuation, single spaces, no leading filler. */
export function normalize(text: string): string {
  return text
    .toLowerCase()
    .replace(/[.,!?;:"„“”'()]/g, " ")
    .replace(/\s+/g, " ")
    .trim()
    .replace(FILLER, "")
    .trim();
}

function folder(cwd: string): string {
  return cwd.replace(/\/+$/, "").split("/").pop() ?? "";
}

/** The register after one hook event. Ended sessions leave it. */
export function trackSession(list: VoiceSession[], p: HookLike, now: number): VoiceSession[] {
  const id = p.session_id ?? "";
  const event = p.hook_event_name ?? "";
  if (!id || !event || p.coucou_agent) return list;
  if (event === "SessionEnd") return list.filter((s) => s.sessionId !== id);
  const old = list.find((s) => s.sessionId === id);
  const cwd = p.cwd || old?.cwd || "";
  const step = p.tool_name ? `${event} ${p.tool_name}` : event;
  const next: VoiceSession = {
    sessionId: id,
    name: folder(cwd) || old?.name || "Session",
    cwd,
    status: event,
    recent: [...(old?.recent ?? []), step].slice(-RECENT),
    finalLine: event === "Stop" ? (p.last_assistant_message ?? null) : event === "UserPromptSubmit" ? null : (old?.finalLine ?? null),
    updated: now,
  };
  return [...list.filter((s) => s.sessionId !== id), next];
}

function spoken(name: string): string {
  return name.toLowerCase().replace(/[-_]+/g, " ");
}

/** "sag <name>[:,] <text>": the longest full name first, then a unique first-word prefix. */
function instruction(rest: string, sessions: VoiceSession[]): VoiceAction {
  const clean = rest.replace(/^\s+/, "");
  const lower = clean.toLowerCase();
  const names = sessions.map((s) => s.name);
  const full = [...sessions]
    .sort((a, b) => b.name.length - a.name.length)
    .find((s) => new RegExp(`^${spoken(s.name).replace(/ /g, "[\\s_-]+")}[\\s:,]+`).test(lower));
  if (full) {
    const text = clean.replace(new RegExp(`^${spoken(full.name).replace(/ /g, "[\\s_-]+")}[\\s:,]+`, "i"), "").trim();
    return text ? { kind: "instruct", sessionId: full.sessionId, name: full.name, text } : { kind: "which-session", names };
  }
  const m = /^(\S+?)[:,]?\s+(.+)$/.exec(clean);
  if (!m) return { kind: "which-session", names };
  const word = m[1].toLowerCase();
  const hits = sessions.filter((s) => spoken(s.name).startsWith(word));
  if (hits.length !== 1) return { kind: "which-session", names };
  return { kind: "instruct", sessionId: hits[0].sessionId, name: hits[0].name, text: m[2].trim() };
}

function answerFor(norm: string, pending: ApprovalInfo): VoiceAction | null {
  const qs = pending.questions;
  if (!qs) return null;
  if (qs.length !== 1 || qs[0].multiSelect) return { kind: "say", text: "Bitte im Island beantworten." };
  const q = qs[0];
  const byLabel = q.options.find((o) => normalize(o.label) === norm);
  if (byLabel) return { kind: "answer", question: q.question, label: byLabel.label };
  const token = norm.replace(/^option\s+/, "");
  const n = /^\d+$/.test(token) ? Number(token) : NUMBERS[token];
  if (n === undefined) return norm.startsWith("option ") ? { kind: "say", text: "Diese Option gibt es nicht." } : null;
  const opt = q.options[n - 1];
  return opt ? { kind: "answer", question: q.question, label: opt.label } : { kind: "say", text: `Option ${n} gibt es nicht.` };
}

export function route(transcript: string, pending: ApprovalInfo | null, sessions: VoiceSession[]): VoiceAction {
  const norm = normalize(transcript);
  if (!norm) return { kind: "ignore" };
  if (pending) {
    if (pending.questions) {
      const a = answerFor(norm, pending);
      if (a) return a;
    } else if (ALLOW.has(norm)) {
      return { kind: "approve", decision: "allow" };
    } else if (DENY.has(norm)) {
      return { kind: "approve", decision: "deny" };
    }
  }
  const lowered = transcript.trim().replace(/^[\s,.!]+/, "");
  const withoutFiller = lowered.replace(new RegExp(FILLER.source, "i"), "");
  const m = INSTRUCT.exec(withoutFiller.toLowerCase());
  if (m) return instruction(withoutFiller.slice(withoutFiller.length - m[1].length), sessions);
  return { kind: "question", text: transcript.trim() };
}

export const BRAIN_SYSTEM =
  "Du bist Coucou, ein Begleiter für Claude-Code-Sessions. Antworte auf Deutsch in höchstens zwei kurzen Sätzen, " +
  "gesprochen, ohne Markdown. Nutze nur den Kontext unten; wenn er die Frage nicht beantwortet, sag das.";

export function brainContext(sessions: VoiceSession[], pending: ApprovalInfo | null): string {
  const lines = sessions.map(
    (s) => `- ${s.name} (${s.cwd}): Status ${s.status}; zuletzt ${s.recent.join(" → ")}${s.finalLine ? `; letzte Antwort: ${s.finalLine}` : ""}`,
  );
  const card = pending ? `\nWartende Freigabe: ${pending.tool} – ${pending.command}` : "";
  return `Laufende Sessions:\n${lines.join("\n") || "- keine"}${card}`;
}
```

**Wenn ein Test zum Präfix- oder Namensabgleich scheitert:** nur `instruction()` anpassen, nie die Freigabe-Regeln lockern.

- [ ] **Step 4: Tests grün**

Run: `node --import ./tests/setup.mjs --test tests/voice.test.mjs`
Expected: alle PASS.

- [ ] **Step 5: Commit**

```bash
git add src/core/voice.ts tests/voice.test.mjs
git commit -m "feat(linux): voice router and session register"
```

---

### Task 8: `session_send_text` – Text in das kitty-Fenster einer Session

**Files:**
- Modify: `src-tauri/src/platform/linux_focus.rs` (Modul `kitty`, ~:470-500; Tests ~:500)
- Modify: `src-tauri/src/platform/mod.rs:21` (Re-Export)
- Modify: `src-tauri/src/lib.rs` (Befehl)
- Modify: `src/core/bridge.ts`

**Interfaces:**
- Consumes: `session_window::lookup(session: &str) -> Option<Vec<u32>>`, `linux_focus::still_linked(owners: &[u32]) -> Vec<(u32, String)>`.
- Produces (Rust): `kitty::send_text_args(to: &str, shell: u32, text: &str) -> Vec<String>`; `linux_focus::send_session_text(owners: &[u32], text: &str) -> bool`.
- Produces (Tauri): `session_send_text(session_id: String, text: String) -> Result<(), String>` (Err „not reachable").
- Produces (TS): `Bridge.sessionSendText(sessionId: string, text: string): Promise<void>` (wirft bei Err).

- [ ] **Step 1: Failing Test** – in `mod tests` von `linux_focus.rs`:

```rust
    #[test]
    fn send_text_is_one_argument_with_one_return_at_the_end() {
        let a = kitty::send_text_args("unix:@kitty-42", 777, "echo \"$HOME\"\nrm -rf /\r");
        assert_eq!(&a[..6], ["@", "--to", "unix:@kitty-42", "send-text", "--match", "pid:777"]);
        assert_eq!(a[6], "--");
        assert_eq!(a[7], "echo \"$HOME\" rm -rf /\r");
        assert_eq!(a.len(), 8);
    }
```

- [ ] **Step 2: Scheitern prüfen**

Run: `cargo test -p coucou --lib linux_focus`
Expected: Compile-Fehler (`send_text_args` fehlt).

- [ ] **Step 3: Implementieren**

Im Modul `kitty` von `linux_focus.rs`, nach `focus_tab`:

```rust
    /// `kitten @ send-text` arguments: the text as one argument (no shell),
    /// line breaks flattened so nothing is submitted early, one Enter at the end.
    pub fn send_text_args(to: &str, shell: u32, text: &str) -> Vec<String> {
        let flat = text.replace(['\r', '\n'], " ").split_whitespace().collect::<Vec<_>>().join(" ");
        vec![
            "@".into(), "--to".into(), to.into(), "send-text".into(),
            "--match".into(), format!("pid:{shell}"), "--".into(), format!("{flat}\r"),
        ]
    }

    /// Types `text` into the session's kitty window, as if typed. False when
    /// the session does not run in kitty or remote control is off.
    pub fn send_text(chain: &[(u32, String)], text: &str) -> bool {
        let Some(at) = chain.iter().position(|(_, name)| name == "kitty") else { return false };
        let Some((shell, _)) = at.checked_sub(1).and_then(|i| chain.get(i)) else { return false };
        let kitty_pid = chain[at].0;
        let Ok(environ) = std::fs::read(format!("/proc/{shell}/environ")) else { return false };
        let Some(to) = listen_on(&environ) else { return false };
        let Some(kitten) = kitten(kitty_pid) else { return false };
        run_quietly(Command::new(kitten).args(send_text_args(&to, *shell, text)), Duration::from_secs(2))
    }
```

In `linux_focus.rs` (Modulebene, nach `focus_session_window`):

```rust
/// Linux fork: types `text` into the terminal a session runs in (kitty only).
pub fn send_session_text(owners: &[u32], text: &str) -> bool {
    let chain = still_linked(owners);
    !chain.is_empty() && kitty::send_text(&chain, text)
}
```

`platform/mod.rs:21`: `send_session_text` zur `pub use self::linux_focus::{…}`-Liste hinzufügen.

`lib.rs`, neben `open_session`:

```rust
/// Linux fork: a dictated instruction, typed into the session's kitty window.
#[tauri::command]
async fn session_send_text(session_id: String, text: String) -> Result<(), String> {
    #[cfg(target_os = "linux")]
    {
        let owners = session_window::lookup(&session_id).ok_or("not reachable")?;
        let ok = tauri::async_runtime::spawn_blocking(move || platform::send_session_text(&owners, &text))
            .await
            .map_err(|e| e.to_string())?;
        if ok { Ok(()) } else { Err("not reachable".into()) }
    }
    #[cfg(not(target_os = "linux"))]
    {
        let _ = (session_id, text);
        Err("not reachable".into())
    }
}
```

`session_send_text,` in `generate_handler!`.

`src/core/bridge.ts`:

```ts
  /** Types a dictated instruction into the session's kitty window. Throws "not reachable". */
  sessionSendText: (sessionId: string, text: string) => callOrThrow<void>("session_send_text", { sessionId, text }),
```

- [ ] **Step 4: Tests grün**

Run: `cargo test -p coucou --lib linux_focus`
Expected: PASS.

- [ ] **Step 5: Manuell**

Claude Code in kitty starten (neues kitty-Fenster nach Task 0). `npm run tauri dev`. In der Browser-Konsole des Islands (Tauri Devtools) `await window.__TAURI__.core.invoke("session_send_text", { sessionId: "<id aus coucou.log>", text: "sag nur hallo" })` → Text erscheint in der Session und wird abgeschickt.

- [ ] **Step 6: Commit**

```bash
git add src-tauri/src/platform/linux_focus.rs src-tauri/src/platform/mod.rs src-tauri/src/lib.rs src/core/bridge.ts
git commit -m "feat(linux): type dictated instructions into a session's kitty window"
```

---

### Task 9: Verdrahtung im Island (`src/island/voice.ts`)

**Files:**
- Create: `src/island/voice.ts`
- Modify: `src/island/island.ts` (`voicePress`/`voiceRelease` aus Task 3 füllen, `registerVoice` beim Start aufrufen – Suche nach `registerHookHandlers(`)
- Modify: `src/island/shortcuts.ts` (`Esc` bricht eine Vorschau ab – in `runIslandKey` bzw. dem Keydown-Handler, Suche nach `"Escape"`)
- Test: `tests/voice-island.test.mjs`

**Interfaces:**
- Consumes: `route`, `trackSession`, `brainContext`, `BRAIN_SYSTEM` (Task 7); `Bridge.voiceRecordStart/Stop`, `voiceSpeak`, `brainAsk`, `sessionSendText`, `approvalDecision`, `approvalAnswer` (Tasks 2, 4, 8, bestehend); `State.pendingApproval`, `State.noteMessage`, `State.settings.voice*`.
- Produces:

```ts
export interface VoiceHost { alert(view: "note"): void; setView(view: string): void; defaultView(): string; }
export function registerVoice(host: VoiceHost): VoiceController;
export interface VoiceController { press(): void; release(): void; cancel(): boolean; sessions(): VoiceSession[]; }
export const PREVIEW_MS = 2000;
```

- [ ] **Step 1: Failing Tests** – `tests/voice-island.test.mjs`:

```js
// Voice wiring (src/island/voice.ts) through the real bridge: what each spoken
// sentence makes Coucou invoke.

import { beforeEach, mock, test } from "node:test";
import assert from "node:assert/strict";
import { calls, emit, internals, sent } from "./tauri.mjs";
import { registerVoice, PREVIEW_MS } from "../src/island/voice.ts";
import { DEFAULT_SETTINGS, State } from "../src/core/state.ts";

let answers;
const plainInvoke = internals.invoke;
internals.invoke = async (cmd, args) => {
  const result = await plainInvoke(cmd, args);
  if (cmd in answers) return typeof answers[cmd] === "function" ? answers[cmd](args) : answers[cmd];
  return result;
};
// setImmediate, not setTimeout: some tests mock setTimeout.
const flush = () => new Promise((r) => setImmediate(r));
const views = [];
const voice = registerVoice({ alert: (v) => views.push(v), setView: (v) => views.push(v), defaultView: () => "overview" });

beforeEach(() => {
  calls.length = 0;
  views.length = 0;
  answers = {};
  State.settings = { ...DEFAULT_SETTINGS, voiceEnabled: true, voiceWhisperModel: "/m.bin", voicePiperVoice: "/v.onnx" };
  State.pendingApproval = null;
});

async function say(text) {
  answers.voice_record_stop = text;
  voice.press();
  voice.release();
  await flush(); await flush();
}

test("nothing happens while voice is off", async () => {
  State.settings.voiceEnabled = false;
  voice.press();
  assert.equal(sent("voice_record_start"), undefined);
});

test("a second press stops the recording when no release ever came", async () => {
  answers.voice_record_stop = "ja";
  voice.press();
  voice.press();
  await flush(); await flush();
  assert.equal(calls.filter((c) => c.cmd === "voice_record_start").length, 1);
  assert.ok(sent("voice_record_stop"));
});

test("'erlauben' allows the card on screen", async () => {
  State.pendingApproval = { requestId: "r1", sessionId: "s1", pillId: "p", tool: "Bash", command: "npm test" };
  await say("Erlauben.");
  assert.deepEqual(sent("approval_decision").args, { requestId: "r1", decision: "allow" });
});

test("an instruction waits for the preview, then is typed into the session", async () => {
  mock.timers.enable({ apis: ["setTimeout"] });
  emit("hook", { hook_event_name: "UserPromptSubmit", session_id: "a", cwd: "/home/stefan/Code/finanzen" });
  await say("sag finanzen: Tests laufen lassen");
  assert.equal(sent("session_send_text"), undefined);
  mock.timers.tick(PREVIEW_MS);
  await flush();
  assert.deepEqual(sent("session_send_text").args, { sessionId: "a", text: "Tests laufen lassen" });
  mock.timers.reset();
});

test("a press during the preview cancels the instruction", async () => {
  mock.timers.enable({ apis: ["setTimeout"] });
  emit("hook", { hook_event_name: "UserPromptSubmit", session_id: "a", cwd: "/home/stefan/Code/finanzen" });
  await say("sag finanzen: Tests laufen lassen");
  voice.press();
  mock.timers.tick(PREVIEW_MS);
  await flush();
  assert.equal(sent("session_send_text"), undefined);
  assert.equal(calls.filter((c) => c.cmd === "voice_record_start").length, 1, "the cancelling press does not record");
  mock.timers.reset();
});

test("a session that ended during the preview is not typed into", async () => {
  mock.timers.enable({ apis: ["setTimeout"] });
  emit("hook", { hook_event_name: "UserPromptSubmit", session_id: "a", cwd: "/home/stefan/Code/finanzen" });
  await say("sag finanzen: weiter");
  emit("hook", { hook_event_name: "SessionEnd", session_id: "a" });
  mock.timers.tick(PREVIEW_MS);
  await flush();
  assert.equal(sent("session_send_text"), undefined);
  assert.match(State.noteMessage ?? "", /nicht erreichbar/);
  mock.timers.reset();
});

test("a question goes to the brain with the session context and is spoken", async () => {
  emit("hook", { hook_event_name: "Stop", session_id: "a", cwd: "/home/stefan/Code/finanzen", last_assistant_message: "Fertig." });
  answers.brain_ask = "Die finanzen-Session ist fertig.";
  await say("Was macht Claude gerade?");
  await flush();
  const ask = sent("brain_ask").args;
  assert.ok(ask.query.includes("finanzen") && ask.query.includes("Was macht Claude gerade?"));
  assert.equal(ask.model, "haiku");
  assert.deepEqual(sent("voice_speak").args, { text: "Die finanzen-Session ist fertig.", voice: "/v.onnx" });
  assert.equal(State.noteMessage, "Die finanzen-Session ist fertig.");
});

test("Esc cancels a pending preview", async () => {
  mock.timers.enable({ apis: ["setTimeout"] });
  emit("hook", { hook_event_name: "UserPromptSubmit", session_id: "a", cwd: "/home/stefan/Code/finanzen" });
  await say("sag finanzen: weiter");
  assert.equal(voice.cancel(), true);
  mock.timers.tick(PREVIEW_MS);
  await flush();
  assert.equal(sent("session_send_text"), undefined);
  mock.timers.reset();
});
```

**Wenn `sent()` den letzten Aufruf in anderer Form liefert** (siehe Task 6): Assertions anpassen, Inhalte gleich lassen.

- [ ] **Step 2: Scheitern prüfen**

Run: `node --import ./tests/setup.mjs --test tests/voice-island.test.mjs`
Expected: FAIL (Modul fehlt).

- [ ] **Step 3: Implementieren** – `src/island/voice.ts`:

```ts
// Linux fork: push-to-talk wiring. Press starts pw-record, release (or a
// second press, where the desktop reports no release) transcribes, and the
// router (src/core/voice.ts) decides. Everything shows in the note view; no
// existing view changes.

import { Bridge } from "../core/bridge";
import { State } from "../core/state";
import { onEvent } from "../core/events";
import { BRAIN_SYSTEM, brainContext, route, trackSession, type HookLike, type VoiceAction, type VoiceSession } from "../core/voice";

export const PREVIEW_MS = 2000;
const NOTE_MS = 6000;

export interface VoiceHost {
  alert(view: "note"): void;
  setView(view: string): void;
  defaultView(): string;
}

export interface VoiceController {
  press(): void;
  release(): void;
  /** Cancels a pending instruction preview. True when there was one. */
  cancel(): boolean;
  sessions(): VoiceSession[];
}

export function registerVoice(host: VoiceHost): VoiceController {
  let sessions: VoiceSession[] = [];
  let recording = false;
  let preview: ReturnType<typeof setTimeout> | null = null;
  let noteTimer: ReturnType<typeof setTimeout> | null = null;

  void onEvent<HookLike>("hook", (p) => {
    sessions = trackSession(sessions, p, Date.now());
  });

  const settings = () => State.settings;

  function note(text: string, speak = true) {
    State.noteMessage = text;
    host.alert("note");
    if (noteTimer) clearTimeout(noteTimer);
    noteTimer = setTimeout(() => host.setView(host.defaultView()), NOTE_MS);
    if (speak && settings().voiceSpeakAnswers) void Bridge.voiceSpeak(text, settings().voicePiperVoice);
  }

  function cancel(): boolean {
    if (!preview) return false;
    clearTimeout(preview);
    preview = null;
    note("Abgebrochen.", false);
    return true;
  }

  async function act(action: VoiceAction) {
    switch (action.kind) {
      case "ignore":
        return;
      case "approve": {
        const pending = State.pendingApproval;
        if (!pending) return;
        await Bridge.approvalDecision(pending.requestId, action.decision);
        note(action.decision === "allow" ? "Erlaubt." : "Abgelehnt.");
        return;
      }
      case "answer": {
        const pending = State.pendingApproval;
        if (!pending) return;
        await Bridge.approvalAnswer(pending.requestId, { [action.question]: action.label });
        note(`Antwort: ${action.label}.`);
        return;
      }
      case "which-session":
        note(action.names.length ? `Welche Session? ${action.names.join(", ")}.` : "Keine Session läuft.");
        return;
      case "say":
        note(action.text);
        return;
      case "instruct": {
        note(`→ ${action.name}: ${action.text}`, false);
        preview = setTimeout(() => {
          preview = null;
          if (!sessions.some((s) => s.sessionId === action.sessionId)) {
            note(`${action.name} ist nicht erreichbar.`);
            return;
          }
          Bridge.sessionSendText(action.sessionId, action.text)
            .then(() => note(`An ${action.name} gesendet.`))
            .catch(() => {
              void navigator.clipboard?.writeText(action.text).catch(() => {});
              note(`${action.name} ist nicht erreichbar. Text liegt in der Zwischenablage.`);
            });
        }, PREVIEW_MS);
        return;
      }
      case "question": {
        note("Moment …", false);
        const query = `${brainContext(sessions, State.pendingApproval)}\n\nFrage: ${action.text}`;
        try {
          note(await Bridge.brainAsk(BRAIN_SYSTEM, query, settings().voiceBrainModel));
        } catch (err) {
          note(`Keine Antwort: ${String(err).replace(/^Error:\s*/, "")}`, false);
        }
        return;
      }
    }
  }

  async function finish() {
    recording = false;
    try {
      const text = await Bridge.voiceRecordStop(settings().voiceWhisperModel);
      if (!text) return;
      await act(route(text, State.pendingApproval, sessions));
    } catch (err) {
      note(`Sprache: ${String(err).replace(/^Error:\s*/, "")}`, false);
    }
  }

  return {
    press() {
      if (!settings().voiceEnabled) return;
      if (cancel()) return;
      if (recording) {
        void finish();
        return;
      }
      recording = true;
      note("Ich höre …", false);
      Bridge.voiceRecordStart().catch((err) => {
        recording = false;
        note(`Mikrofon: ${String(err).replace(/^Error:\s*/, "")}`, false);
      });
    },
    release() {
      if (recording) void finish();
    },
    cancel,
    sessions: () => sessions,
  };
}
```

**Wenn `onEvent` an anderer Stelle liegt** (Suche nach `export function onEvent` bzw. dem Import in `src/island/hooks.ts`): Importpfad von dort übernehmen. **Wenn `State.noteMessage` nach `host.alert("note")` nicht neu gezeichnet wird:** vor `host.alert` `State.notify()` aufrufen.

`src/island/island.ts`: beim Start (dort, wo `registerHookHandlers(` aufgerufen wird) `this.voice = registerVoice({ alert: (v) => this.alert(v), setView: (v) => this.setView(v as IslandViewName), defaultView: () => State.defaultView() });` – Feld `private voice!: VoiceController;` anlegen. Die leeren Methoden aus Task 3 werden:

```ts
  voicePress() { this.voice.press(); }
  voiceRelease() { this.voice.release(); }
```

`Esc`: im Keydown-Handler des Islands (Suche nach `"Escape"` in `src/island/shortcuts.ts`) als ersten Schritt: `if (host.voiceCancel?.()) return;` – dazu `voiceCancel?(): boolean;` in `ShortcutHost` und in `island.ts` `voiceCancel() { return this.voice.cancel(); }`.

- [ ] **Step 4: Tests grün**

Run: `npm test`
Expected: alle PASS, auch die bestehenden.

- [ ] **Step 5: Commit**

```bash
git add src/island/voice.ts src/island/island.ts src/island/shortcuts.ts tests/voice-island.test.mjs
git commit -m "feat(linux): wire push-to-talk to approvals, instructions and questions"
```

---

### Task 10: E2E auf wintermute und Doku

**Files:**
- Create: `../HANDOFF.md`
- Modify: `../CLAUDE.md` (Abschnitt „Fork (Gschirr)" anhängen)
- Modify: `../README.md` (Abschnitt „This fork" direkt unter der Überschrift)

- [ ] **Step 1: E2E-Checkliste** (`npm run tauri dev`, Sprache in Settings an, zwei Claude-Code-Sessions in kitty: `~/Code/finanzen`-Kopie oder Testordner `~/tmp/a`, `~/tmp/b`)

1. Chat-Mikrofon: „Wie wird das Wetter morgen" → korrekter Text im Feld.
2. In Session a `ls` als Bash-Freigabe auslösen → Taste halten, „erlauben" → Befehl läuft.
3. „ja aber nur lesen" bei offener Karte → keine Freigabe, Frage-Weg antwortet.
4. „sag b: schreib hallo in eine Datei" → Vorschau 2 s → erscheint in Session b, nicht in a.
5. „sag b: …" und während der Vorschau erneut drücken → „Abgebrochen.", nichts gesendet.
6. „Was macht Claude gerade?" → gesprochene Antwort nennt a und b.
7. Settings: Modellpfad auf `/x` → „Fehlt: /x"; Coucou läuft weiter; zurücksetzen.
8. KDE: Shortcut „Halten, um mit Coucou zu sprechen" erscheint in den Systemeinstellungen; Loslassen beendet die Aufnahme. **Wenn Loslassen nicht ankommt:** zweiter Druck beendet sie (Review Focus 1) – im HANDOFF vermerken.

Jeden Punkt mit ✅/🔴 im HANDOFF festhalten. **Wenn ein Punkt 🔴 ist:** Ursache mit `bug-investigation` klären, nicht raten.

- [ ] **Step 2: Doku**

`../CLAUDE.md` anhängen:

```markdown
## Fork (Gschirr)

<!-- Generelle Arbeitsregeln: ~/.claude/CLAUDE.md — wird automatisch geladen -->
- Eigener Fork mit Linux-Sprachsteuerung. Spec: `docs/superpowers/specs/2026-10-12-linux-voice-design.md`, Entscheidung: `docs/decisions/2026-10-12-linux-voice.md`, Stand: `HANDOFF.md`.
- Bewusste Abweichung: Freigaben per Sprache (nur fester Wortabgleich in `windows/src/core/voice.ts`).
- Upstream nachziehen: `git fetch upstream && git merge upstream/main`, danach `npm test` und `cargo test -p coucou --lib`.
```

`../README.md` unter dem Titelblock:

```markdown
> **This fork** (Gschirr/coucou) adds push-to-talk voice control to the Linux build: local whisper.cpp, Piper, `claude -p` and kitty. See `docs/superpowers/specs/2026-10-12-linux-voice-design.md`.
```

`../HANDOFF.md`: Current State (✅/⚠️/🔴 je Phase), Where We Left Off, offene TODOs nach Priorität (u. a. neuromancer, devbox-Bonus, allgemeiner Assistent), Key Decisions mit Datum, Test- und Build-Befehle.

- [ ] **Step 3: Commit und Push**

```bash
git add ../HANDOFF.md ../CLAUDE.md ../README.md
git commit -m "docs: fork notes, handoff and E2E results"
git push
```

---

### Task 11: Paket installieren (wintermute, dann neuromancer)

Hauptsession, mit Stefan.

- [ ] **Step 1: Branch in den Fork-`main` bringen** – über superpowers:finishing-a-development-branch (Merge `linux-voice` → `main`, Push).

- [ ] **Step 2: wintermute**

```bash
sudo snapper -c root create -d "vor coucou-git install"
cd ~/Code/coucou/linux/arch && makepkg -si
```

Coucou aus dem Startmenü starten, Settings → Sprache → „Bereit". E2E-Punkte 1, 2, 4 erneut prüfen. CHANGELOG-Eintrag (Paket `coucou-git`, Rollback `sudo pacman -Rns coucou-git`).

- [ ] **Step 3: neuromancer** (über SSH oder vor Ort; Task 0 Steps 1–5 und 7 dort wiederholen, kitty.conf kommt per `yadm pull`)

Pakete: `paru -Syu whisper-cpp piper-tts-bin ggml-vulkan` (Vulkan nur, wenn `vulkaninfo --summary` eine GPU zeigt; sonst ohne). Statt `ggml-large-v3-turbo-q5_0.bin` das kleinere Modell laden:

```bash
curl -L -o ~/.local/share/coucou/models/ggml-small.bin https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-small.bin
```

Repo klonen (`gh repo clone Gschirr/coucou ~/Code/coucou`), `makepkg -si`, in Settings → Sprache das Whisper-Modell auf `~/.local/share/coucou/models/ggml-small.bin` setzen. E2E-Punkte 1–3. Ergebnis im HANDOFF nachtragen, committen, pushen.
