# race-viewer

A readable live and replay viewer for Vakaros-tracked sailboat regattas.

Wind-up course orientation, labeled marks and start line, a live-sorted running
order, tap-to-follow with speed and heading, replay with play/pause, and overall
series standings with per-race breakdowns on hover.

Single self-contained `index.html`. No build step, no dependencies.

## Run

Serve it over http — don't open the file directly, or the browser will block the
API calls it needs.

```
python3 -m http.server 8000
open http://localhost:8000
```

## Point it at a regatta

In `index.html`, near the top of the `<script>` block:

```js
const REGATTA_ID = 'cOwE4vej0GntZwctsVjL';
const DIVISION   = 'MC_Scow';
```

Both come from the Vakaros player URL:
`player.vakaros.com/watch/<REGATTA_ID>/<Division>` — replace spaces in the
division with underscores.

## Deploy

Any static host works, since there's no server side:

```
npx vercel deploy --prod
# or
npx netlify deploy --prod --dir .
# or drop index.html in an S3 bucket / GitHub Pages
```

## Controls

| Control | What it does |
| --- | --- |
| Race dropdown | Pick a race, or OVERALL for series standings |
| ⟲ | Replay the selected race from the gun |
| ▶ / ⏸ | Play or pause (8× speed) |
| Scrubber | Jump to any moment; ● LIVE returns to real time |
| Labels dropdown | Sail numbers, skipper names, or none |
| SPREAD / EXACT | Nudge overlapping boats apart, or show true GPS positions |
| ⛵+ / ⛵− | Boat marker size |
| Tap a boat or name | Follow it — name, position, speed, heading, recent track |
| Double-click map | Reset zoom and resume auto-fit |

See `CLAUDE.md` for API details and the non-obvious behaviour of the race data.
