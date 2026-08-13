# Sailboat race viewer

A single-file live/replay viewer for Vakaros RaceSense-tracked sailboat regattas.
Built because the official player (`player.vakaros.com`) makes it hard to see the
course, the marks, or who is actually leading.

Everything lives in `index.html` — no build step, no dependencies, no framework.
Open it in a browser and it runs. Keep it that way unless there's a strong reason
not to.

## Running it

It must be served over http(s), not opened as a `file://` path — iOS and some
desktop browsers sandbox local files and block the outbound API calls, which
surfaces as a "Network blocked" message.

```
python3 -m http.server 8000    # then open http://localhost:8000
```

## Data sources

Three external APIs, all reachable from the browser without a backend.

**1. Firebase anonymous auth** — the regatta document requires an authenticated
read. Anonymous sign-up is enough.

```
POST https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=<FB_KEY>
body: {"returnSecureToken":true}   ->  { idToken }
```

Token expires in ~1 hour; the code refreshes at 50 minutes and also clears the
token on any 401/403 so the next poll re-auths.

**2. Regatta document (Firestore REST)** — race list, stages, start times,
official finish order, participants, and the course/mark configuration.

```
GET https://firestore.googleapis.com/v1/projects/vakaros-racesense/databases/(default)/documents/regattas/<REGATTA_ID>?key=<FB_KEY>
Authorization: Bearer <idToken>
```

Firestore REST returns typed values (`stringValue`, `mapValue`, …); `unwrap()`
flattens them. `listCollectionIds` and most other paths are permission-denied —
only this document read is allowed, so don't go looking for subcollections.

**3. Telemetry (public, no auth)** — positions for boats and marks.

```
GET https://teleapi.regatta.app/telemetry/event/<REGATTA_ID>
    ?after=<ms>&before=<ms>&limit=<n>&division=<DIVISION>
```

Returns `{Fields:[...], Rows:[[...]]}` — columnar, not objects. Fields include
`ts, sn, division, sail_number, race_number, race_stage, role, latitude,
longitude, heading, sog`. `role` is `competitor` or `mark`.

Both `Fields` and `Rows` come back **null** (not an error) when the window is
empty or the timestamps are out of range. Always guard for that.

## Config

Top of the `<script>` block:

```js
const REGATTA_ID = '...';   // from the player URL: /watch/<REGATTA_ID>/<Division>
const DIVISION   = '...';   // spaces become underscores: "MC Scow" -> "MC_Scow"
```

Both are hardcoded. **A good next task is reading them from the query string**
(`?regatta=...&division=...`) so one deployment serves any regatta.

## Architecture

- `pollDoc()` every 30s — races, stages, official finishes, mark roles. Also
  populates `NAMES` from participants (`boatName` holds the skipper name in
  these events, not the boat's name).
- `pollTele()` every 3s — quick paint of the last 90s first, then history
  backfills in the background one page per cycle. Loading the whole race up
  front takes ~8MB and stalls on cellular; don't reintroduce that.
- `boats: Map<sail, {samples:[[ts,lat,lon,hdg,sog]], everUp}>` — `sampleAt()`
  interpolates between samples, including circular interpolation of heading.
- `markPos: Map<sn, [[ts,lat,lon]]>` — marks are time series too (see gotchas).
- `draw(t)` renders everything for time `t`; `computeOrder(t)` ranks the fleet.
- Live vs replay is one variable: `cursor` (null = follow the live edge).

## Gotchas — each of these was a real bug, don't undo them

**Marks move.** They're GPS trackers on buoys and the committee boat. Storing
only the latest fix, or a median over the whole window, puts marks where RC
parked them *after* racing — one device drifted 500m. Sample marks at the
render time, exactly like boats.

**The start line and finish line are different lines.** The committee boat
motors up the course to set the finish; the two line devices are ~200m apart at
the gun and ~795m apart at the first finish. The start line is therefore frozen
at gun-sequence geometry (`startMs + 30s`) rather than drawn between two
drifting devices. Windward marks and the gate stay time-sampled — those really
do get re-laid between races.

**Course rotation sign.** The view rotates so the windward mark is up. The
rotation matrix uses `+windAng`, not `-windAng`. The wrong sign looks fine on a
course that happens to point due north and is obviously broken on any other.

**Trackers go off air.** A boat that finishes often stops transmitting. Ranking
from live positions alone silently drops finishers from the standings. Build the
order from the official `finishes` array and fill in live positions, not the
reverse. Boats with no live fix get an `offair` flag and a `·` in the list.

**Batched uploads.** Boats' trackers report in batches, so at any instant some
are 20–60s behind. Holding a stale boat at its last position (faded) is correct;
dropping it makes the fleet count flicker.

**Zoom anchoring.** The transform is `screen = W/2 + world*scale*z + offset`.
Zoom-to-cursor has to account for the `W/2` term or the view creeps toward
center. Also: auto-fit re-centers every frame, so it must be frozen
(`manualView`) the moment the user zooms or pans. Double-click resets.

**Don't rebuild the leaderboard every frame.** It detaches the row under the
cursor and breaks hover. Only write `innerHTML` when the markup actually changed.

**Sail numbers may carry a `USA ` prefix** in the data. `shortSail()` strips it
for display; the raw value stays the key.

## Scoring

`overallStandings()` is low-point: finish position = points, lowest total wins,
ties broken by number of wins. A boat that raced earlier but is missing from a
later race gets `fleetSize + 1` (DNF). **No drop race is applied** — check the
event's sailing instructions before adding one.

## Ideas worth doing next

- Regatta/division from the query string (see Config above).
- Persist a favorite boat, highlighted on the map.
- Wind direction indicator derived from the fleet's upwind VMG.
- Distance-to-line and time-to-burn during the start sequence.
- Handle multi-lap courses explicitly; ranking currently assumes one beat plus a
  return leg.
