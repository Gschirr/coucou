# Entscheidung: Linux-Sprachsteuerung im eigenen Fork

Datum: 12.10.2026 · Details: `docs/superpowers/specs/2026-10-12-linux-voice-design.md`

**Was:** Eigener Fork `Gschirr/coucou` mit Push-to-talk-Sprachsteuerung für lokale Claude-Code-Sessions auf wintermute und neuromancer. Spracherkennung lokal (whisper.cpp), Antworten über `claude -p` (Abo), Sprachausgabe lokal (Piper), Anweisungen per `kitten @ send-text`.

**Warum:** Upstream hat Sprache nur auf macOS (Apple-APIs). Stefan arbeitet auf CachyOS/KDE mit kitty und will Sessions per Sprache abfragen, freigeben und steuern, ohne API-Kosten und ohne Audio in die Cloud.

**Bewusste Abweichung:** Upstream erlaubt Freigaben nur per Klick. Dieser Fork erlaubt sie per Sprache, abgesichert durch festen Wortabgleich, nur bei genau einer offenen Freigabe, nie über das Sprachmodell, kein „Immer erlauben".

**Verworfen:**
- PRs an Upstream: Stefans Wahl war ein eigener Fork.
- REA (Reverse Engineering): unnötig, Coucou ist Open Source.
- Wake-Word „OK Coucou": mehr Aufwand, Dauer-Mikrofon, Fehlauslöser; Push-to-talk reicht.
- Anthropic-API-Key als Brain: Extra-Kosten; `claude -p` nutzt das Abo.
- Stop-Hook bzw. `claude --resume -p` für Anweisungen: nur am Session-Ende bzw. unsichtbar im Terminal.
- Audio-Aufnahme in der Webview (`getUserMedia`): WebKitGTK-Mikrofonrechte; `pw-record` ist einfacher.
