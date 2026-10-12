# Linux-Sprachsteuerung für Coucou (Fork Gschirr/coucou)

Stand: 12.10.2026 · Status: Spec freigegeben im Gespräch, schriftliche Prüfung offen

## Ziel

Stefan spricht per Push-to-talk mit Coucou und steuert damit die Claude-Code-Sessions, die lokal auf **wintermute** und **neuromancer** laufen (CachyOS, KDE Plasma 6, Wayland, kitty).

**Erfolg heißt:**
- Taste halten, „erlauben" sagen: die einzige offene Freigabe geht durch.
- Taste halten, „sag finanzen: lass die Tests laufen": der Text landet im kitty-Fenster der finanzen-Session.
- Taste halten, „Was macht Claude gerade?": Coucou antwortet gesprochen und im Island.
- Mikrofon-Knopf im Chat: diktierter Text steht im Eingabefeld.

## Rahmen

| Entscheidung | Wert |
|---|---|
| Fork-Modell | eigener Fork `Gschirr/coucou`, keine PRs upstream; `upstream` als Remote zum Nachziehen |
| Projektort | lokal `~/Code/coucou` (GUI und Mikrofon), nicht devbox |
| Spracherkennung | lokal, `whisper-server` aus `whisper-cpp` (extra) |
| „Gehirn" | `claude -p` mit Stefans claude.ai-Abo, kein API-Key |
| Sprachausgabe | lokal, Piper (`piper-tts-bin`, AUR) mit deutscher Stimme |
| Auslöser | Push-to-talk über globalen Shortcut, kein Wake-Word |
| Anweisung an Session | `kitten @ send-text` in das kitty-Fenster der Session |
| Sprache | Deutsch |

**Nicht in dieser Spec:** allgemeiner Assistent (Stretch Goal), devbox-Anbindung (Bonus, SSH-`RemoteForward` des Sockets), Wake-Word, ElevenLabs, Port der Mac-Swift-Voice-Logik.

## Abweichung von der Upstream-Regel

Upstream-`CLAUDE.md`: „Never … approve a Claude Code or Codex permission without an explicit click." Dieser Fork erlaubt Freigaben per Sprache. Das ist eine bewusste Entscheidung von Stefan. Die Schutzmaßnahmen stehen im Abschnitt Sicherheit. Alle anderen Upstream-Regeln gelten weiter (Hook blockiert Claude Code nie, `settings.json` nur mit Backup und Diff, 0 % CPU bei verstecktem Island, bestehende Views nicht umstylen).

## Architektur

Die Logik liegt im Frontend, weil Sessions (`AgentTask`, `src/core/state.ts:20`) und die offene Freigabe (`State.pendingApproval`, `state.ts:44`) dort liegen. Rust liefert einzelne Werkzeug-Befehle.

```
[Taste halten] ─► pw-record (WAV in $XDG_RUNTIME_DIR) ─► [Taste los] ─► whisper-server (HTTP, lokal)
                                                                              │ Transkript
                                                                              ▼
                                                            src/island/voice.ts (Router)
        ┌──────────────────────────────┬───────────────────────────────┴─────────────┐
 1. Freigabe-Weg                 2. Anweisungs-Weg                     3. Frage-Weg
 feste Wörter + Freigabe offen   „sag <session>: <text>"               alles andere
 → approval_decision /           → 2 s Vorschau mit Abbrechen          → brain_ask (claude -p mit
   approval_answer                → session_send_text                    Session-Kontext)
                                                                        → Island + voice_speak
```

Alle Pfade unten relativ zu `windows/`.

### Neue und geänderte Teile

| Teil | Datei | Aufgabe |
|---|---|---|
| Aktion `pushToTalk` | `src-tauri/src/shortcuts.rs` (`ACTIONS` :60, `description()` :82) | neue Aktion; Druck und Loslassen getrennt als Events `shortcut` und `shortcut-release` |
| Loslassen unter Wayland | `src-tauri/src/portal.rs` (`Event` :230, `message()` :513), `shortcuts.rs` `wayland::on_event` :524 | Signal `Deactivated` des GlobalShortcuts-Portals auswerten |
| Loslassen unter X11 | `shortcuts.rs:310` | Filter `!= Pressed` für `pushToTalk` aufheben |
| `voice.rs` (neu) | `src-tauri/src/voice.rs` | `voice_record_start` / `voice_record_stop` (Prozess `pw-record`), `voice_transcribe` (WAV per multipart an `whisper-server`, Kindprozess mit geladenem Modell, Start beim ersten Gebrauch), `voice_speak` (`piper` → `pw-play`), `voice_check` (fehlende Befehle melden) |
| `brain_ask` (neu) | `src-tauri/src/claude_cli.rs` | `claude -p --output-format json` mit Prompt + Kontext, Timeout 60 s |
| Chat-Anbieter `claude-cli` | `src-tauri/src/chat.rs` (`send` :200), `src/core/providers.ts:24` | zusätzlicher Zweig, nutzt `claude_cli.rs`; damit läuft auch der normale Chat über das Abo |
| `session_send_text` (neu) | `src-tauri/src/platform/linux_focus.rs` (kitty-Teil :484) | gleiche Suche nach `KITTY_LISTEN_ON` und Shell-PID wie `focus-window`; `kitten @ --to <listen_on> send-text --match pid:<shell_pid> -- "<text>\r"` |
| Router (neu) | `src/island/voice.ts` | drei Wege, Anzeige im Island, Aufruf der Tauri-Befehle |
| Shortcut-Fall | `src/island/shortcuts.ts:50` | `pushToTalk` → `voice.ts` |
| Mikrofon-Knopf | `src/views/chat.ts:214` (`chat-bar`) | gleiche Aufnahme-Kette; Ergebnis ins Eingabefeld, kein Router |
| Einstellungen | `src-tauri/src/settings.rs:15/:124`, `src/core/state.ts:100/:171`, `src/settings/main.ts` | `voiceEnabled`, `voiceWhisperModel`, `voicePiperVoice`, `voiceSpeakAnswers`; neuer Abschnitt „Sprache" mit Status von `voice_check` |

Freigaben laufen über die vorhandenen Befehle `approval_decision` / `approval_answer` (`lib.rs:424-455`, intern `pipe::answer` :344 und `pipe::answer_question` :356). Der Phone-Link nutzt denselben Weg (`phone_link/mod.rs:66-84`).

### Router-Regeln (`voice.ts`)

Normalisierung: Kleinschreibung, Satzzeichen weg, Füllwörter am Anfang („äh", „also", „okay coucou") weg.

1. **Freigabe-Weg**, nur wenn eine Freigabe oder Frage offen ist:
   - Transkript besteht nur aus `erlauben | ja | okay | zulassen` → `allow`.
   - nur aus `ablehnen | nein | stopp | verweigern` → `deny`.
   - bei AskUserQuestion: `option <n>` / `<n>` (Zahlwort oder Ziffer) oder exakter Options-Label → `approval_answer`.
   - Freier Satz mit Zusatz („ja, aber …") ist **keine** Freigabe und fällt durch zu Weg 2 oder 3.
2. **Anweisungs-Weg**: Muster `sag <name>[:,] <text>` oder `an <name>[:,] <text>`. `<name>` wird gegen `AgentTask.name` abgeglichen (exakt, sonst eindeutiger Präfix).
3. **Frage-Weg**: alles andere. Kontext für `claude -p`: pro Session Name, Status, die letzten 5 Schritte, `finalLine`, offene Freigabe. Die Antwort wird kurz gehalten (Systemprompt: max. 2 Sätze, Deutsch).

## Sicherheit

- „Erlauben" wirkt nur bei **genau einer** offenen Freigabe. Bei mehreren: gesprochene Rückfrage „Welche Session?", die nächste Aufnahme gilt als Antwort mit Session-Namen.
- Die Freigabe-Entscheidung trifft nur der lokale Wortabgleich. `claude -p` kann nie eine Freigabe auslösen.
- „Immer erlauben" gibt es per Sprache nicht.
- Das Island zeigt den erkannten Text auf der Freigabe-Karte. Klick auf Ablehnen bleibt möglich.
- Anweisungen gehen nur an einen eindeutig erkannten Session-Namen, nach 2 s Vorschau mit Abbrechen.
- Audio verlässt den Rechner nie. Die WAV wird nach dem Transkribieren gelöscht. Nur Transkript und Session-Kontext gehen über `claude -p` an Anthropic.

## Fehlerbehandlung

Jeder Fehler: sichtbar im Island, kurz gesprochen (falls Piper geht), Eintrag in `~/.local/share/coucou/coucou.log`.

| Fall | Verhalten |
|---|---|
| `whisper-server`, `piper`, `pw-record` oder Modell fehlt | Abschnitt „Sprache" zeigt den fehlenden Teil und den Installationsbefehl; Sprachfunktion bleibt aus, Rest von Coucou läuft |
| whisper-server-Prozess tot | beim nächsten Tastendruck einmal neu starten |
| Aufnahme < 0,4 s oder leeres Transkript | still verwerfen |
| Session nicht in kitty / kein `KITTY_LISTEN_ON` | Meldung „Session nicht erreichbar", Text in die Zwischenablage |
| Session-Name unklar | Rückfrage mit den Namen der laufenden Sessions |
| `claude -p` scheitert oder Timeout 60 s | Fehlermeldung, keine Wiederholung |
| Coucou-Relay fehlt auf einem Rechner | Hook endet sofort (Upstream-Regel), Claude Code fragt im Terminal |

## Systemvoraussetzungen

- Pakete: `whisper-cpp` (extra), `piper-tts-bin` + deutsche Stimme (z. B. `de_DE-thorsten-medium`), `pipewire` (vorhanden).
- Whisper-Modell pro Rechner: wintermute `large-v3-turbo-q5_0`, neuromancer `small`. Wahl in den Einstellungen.
- kitty (per yadm, beide Rechner): `allow_remote_control socket-only` und `listen_on unix:@kitty-{kitty_pid}`.
- Vor der Installation: `sudo snapper -c root create -d "vor coucou"`, danach Eintrag in `~/Dokumente/system/CHANGELOG.md`.

## Build und Installation

- Eigenes `linux/arch/PKGBUILD` mit `source` aus `Gschirr/coucou` und den Abhängigkeiten oben; Installation `makepkg -si`.
- Entwicklung: `cd windows && npm install && npm run tauri dev`.

## Tests

- **Router** (`windows/tests/voice.test.mjs`, `node --test` wie die bestehenden Tests, `npm test`): Transkript + Zustand → erwarteter Weg. Pflichtfälle: „erlauben" bei 1 offenen → allow; bei 2 offenen → Rückfrage; bei 0 offenen → Frage-Weg; „ja aber nur lesen" → keine Freigabe; „Option zwei" → Antwort Index 2; „sag finanzen: Tests laufen lassen" → send-text finanzen; unbekannter Name → Rückfrage; Präfix „fin" bei eindeutiger Session → finanzen.
- **Rust** (`cargo test`): Portal-`Deactivated` → Event; Aufbau des `kitten`-Befehls inkl. Escaping; multipart-Request an einen Fake-whisper-Server; `voice_check` bei fehlendem Befehl.
- **Manuell (E2E auf wintermute)**:
  1. Diktat im Chat ergibt korrekten deutschen Text.
  2. Echte Freigabe einer lokalen Session per „erlauben".
  3. Anweisung landet im richtigen kitty-Fenster bei zwei offenen Sessions.
  4. Frage „Was macht Claude gerade?" wird gesprochen beantwortet.
  5. Ohne `piper` startet Coucou normal und zeigt den Hinweis.

## Phasen

| Phase | Inhalt | Prüfung |
|---|---|---|
| 1 Fork und Build | PKGBUILD aus dem Fork, Installation wintermute, Hooks | lokale Session erscheint im Island |
| 2 Pipeline und Diktat | Push-to-talk, `voice.rs`, Mikrofon-Knopf, Anbieter `claude-cli` | Diktat steht im Chat; Chat antwortet über das Abo |
| 3 Freigaben per Sprache | Router Weg 1 | Router-Tests grün; echte Freigabe |
| 4 Anweisungen und Fragen | Weg 2 und 3, Piper-Ausgabe | E2E-Checkliste komplett |
| 5 neuromancer | Installation, kleineres Modell, kitty.conf per yadm | E2E-Punkte 1–3 auf neuromancer |
