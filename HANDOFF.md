# HANDOFF – Coucou-Fork mit Linux-Sprachsteuerung

<!-- Generelle Arbeitsregeln: ~/.claude/CLAUDE.md — wird automatisch geladen -->

Stand: 12.10.2026 abends · Branch `linux-voice` · Fork `Gschirr/coucou` (Upstream `Louis-CFM/coucou` als Remote `upstream`)

## Current State

- ✅ Design-Spec: `docs/superpowers/specs/2026-10-12-linux-voice-design.md`
- ✅ Entscheidung: `docs/decisions/2026-10-12-linux-voice.md`
- ✅ Plan (12 Tasks): `docs/superpowers/plans/2026-10-12-linux-voice.md`
- ✅ Task 0, Systemvorbereitung wintermute: whisper-cpp + ggml-vulkan (encode 160 ms auf der RX 7900 XT), piper-tts-bin (Binary `piper-tts`), Modelle in `~/.local/share/coucou/models/`, kitty Remote Control (yadm `52c148c`), CHANGELOG (yadm `5ae78f4`)
- ⚠️ Task 1: `strings.json` neu generiert (`c075b3f`, `npm test` 397/397 grün). `cargo test` linkt nicht, weil Systempakete fehlen. PKGBUILD-Änderung noch offen.
- 🔴 Tasks 2–11 offen

## Where We Left Off

Umsetzung läuft per Subagent-Driven Development. Das Ledger liegt in `.superpowers/sdd/2026-10-12-linux-voice/progress.md` (git-ignoriert, nur lokal auf wintermute). Dort stehen alle Rulings, auch Ruling 4/5: neue UI-Texte gehören in `windows/src/i18n/extra.json` mit allen 9 Nicht-Englisch-Sprachen, nicht in `strings.json`.

## Nächste Schritte (Priorität)

1. Fehlende Build-Pakete installieren (Stefan, sudo):
   `sudo pacman -S gtk-layer-shell libayatana-appindicator gst-plugins-good`
2. Task 1 fertigstellen: `cargo test -p coucou --lib` (aus `windows/`), dann PKGBUILD laut Plan, dann Review.
3. Tasks 2–9 per Subagent (Sonnet), Review nach jedem Task.
4. Task 10: E2E mit Stefan (alle manuellen GUI-Schritte gesammelt, Ruling 1), Hooks installieren, Doku.
5. Task 11: Merge nach `main` (nur nach Rückfrage), Paket auf wintermute, dann neuromancer.

## Key Decisions

- 12.10.2026: Eigener Fork statt PRs upstream.
- 12.10.2026: Push-to-talk statt Wake-Word. whisper.cpp lokal, `claude -p --safe-mode --tools ""` als Brain (Abo), Piper lokal, Anweisungen per `kitten @ send-text`.
- 12.10.2026: Freigabe per Sprache ist eine bewusste Abweichung von der Upstream-Regel „nur per Klick". Abgesichert durch festen Wortabgleich, nur auf die sichtbare Karte, nie über das Sprachmodell.

## Befehle

- Tests: `cd windows && npm test` · `cargo test -p coucou --lib`
- Dev: `cd windows && npm run tauri dev`
- Upstream nachziehen: `git fetch upstream && git merge upstream/main`
