# Sight Reader

A single-file HTML piano sight-reading trainer.

## Key facts
- Pure HTML/CSS/JS, no build step, no dependencies
- Serves at localhost:8080 via `python3 -m http.server 8080`
- Pitch detection via Web Audio API autocorrelation
- Staff drawn in SVG, redraws on every new question
- History stored in Supabase when signed in, localStorage otherwise

## Architecture
- SVG constants: LG=24 (line gap), NR=11 (note radius), mobile uses narrower viewBox
- Note tables: TREBLE_NATURALS and BASS_NATURALS, sp = steps from bottom staff line
- `redraw()` is the main draw function, calls drawStaffLines/drawTrebleClef/drawBassClef/drawNote
- `transitioning` flag prevents double-skipping after a correct note
- `historyCache` holds last-fetched data; `renderDailyView`/`renderSessionLog` read from it synchronously
- `db` (the Supabase client) is null when SUPABASE_URL is not configured; app falls back to localStorage
- Session saved to storage on Stop, grouped by day in Daily Summary view
- Local history is merged into Supabase on first sign-in via `mergeLocalHistory()`
- Row Level Security enforces per-user data isolation in Supabase
