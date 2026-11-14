# UMRB Build Status

A map of Montana Mesonet buildout across the Upper Missouri River Basin, one grid cell per station.

https://mesonet.climate.umt.edu/umrb/

## Run it

A static page with no build step. Serve the repo root and open http://localhost:8000.

```sh
python -m http.server 8000
```

## Deploy

GitHub Pages publishes `main` (repo root). `mesonet.climate.umt.edu/umrb/` proxies it, so pushing `main` deploys to production.
A nightly workflow (`.github/workflows/preview.yml`) commits `assets/og-card.png` to `main`. Pull before you push.

## URL parameters

| Parameter | Values | What it does |
|---|---|---|
| `internal` | `1` | Internal view, with build progress split in two (also `view=internal`) |
| `cat-public` | `active`, `building`, `contracted`, `candidate-26`, `candidate-27`, `unassigned` | Statuses shown in the public view |
| `cat-internal` | `active`, `structures`, `ground`, `contracted`, `candidate-26`, `candidate-27`, `unassigned` | Statuses shown in the internal view |
| `overlays` | `tribal`, `counties`, `hucs` | Reference layers turned on |
| `stations` | `off` | Hide station dots |
| `labels` | `on`, `off` | Grid cell labels |
| `legend` | `open`, `collapsed` | Legend state |
| `theme` | `light`, `dark`, `high-contrast` | Color theme |
| `kbd` | `off` | Turn off the `/` search key |
| `lng`, `lat`, `zoom` | numbers | Map camera |
| `cell` | cell id, e.g. `H-8` | Open that cell |
| `station` | station id, e.g. `aceabsar` | Open that station |

Lists take spaces, `+` or commas. A parameter at its default is left out of the address.

## Data

- Station status: `GET https://mesonet2.climate.umt.edu/api/v2/stations/status/live?type=json`, read on every load. It is a live read of the AirTable "Station Status" view.
- Overlays: FlatGeobuf files under `https://data.climate.umt.edu/mesonet/fgb/`, loaded when turned on.
- Grid cells and the Montana outline: vendored in `data/`. `data/README.md` has the regeneration recipe.

## Tests

```sh
node --check app.js
npx html-validate@9 index.html

# Browser harness. Installs are ephemeral, never committed.
npm init -y && npm i --no-save playwright @axe-core/playwright
npx playwright install chromium
mkdir -p fixtures && curl -fsS -o fixtures/status.json \
  'https://mesonet2.climate.umt.edu/api/v2/stations/status/live?type=json'
node consumer-verify.mjs

# Kit harness, from an mco-web-style checkout beside this repo
node tools/verify/axe-matrix.mjs --config ../mesonet-umrb-build/verify.config.mjs
node tools/verify/keyboard.mjs   --config ../mesonet-umrb-build/verify.config.mjs
```

## Kit

`mco-web-style` v1.0.0, pinned with SRI in `index.html`.

Design rules: mco-web-style/HOUSE-STYLE.md
