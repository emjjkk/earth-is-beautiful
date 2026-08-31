# Somewhere — a random-world-photo Svelte app

Fills the page with a random geotagged photo from Wikimedia Commons. A small
round pin button sits in the corner; hover it to see where the photo was
taken and who shot it, click it to open the original file page on Commons.

## How it works

- On load, it queries the Commons API for photos that carry a
  Wikidata "coordinate location" (`P625`) statement, picks one at random
  from the results, and preloads it before setting it as the background.
- The place name is resolved from the photo's lat/lon via OpenStreetMap's
  Nominatim reverse-geocoding API (free, no key required).
- The pin button shows a hover panel with the place name and photographer
  credit, and links to the photo's Commons page (`descriptionurl`).

## Run it locally

```bash
npm install
npm run dev
```

Then open the printed local URL. `npm run build` produces a static
`dist/` folder you can host anywhere (Netlify, GitHub Pages, etc.) since
everything runs client-side against public APIs — no backend needed.

## Notes

- Not every Commons photo has a location, so the search is scoped to ones
  that do; occasionally a search page will still come back empty and the
  app shows a "try another" prompt.
- Nominatim's public endpoint is rate-limited and meant for light use —
  fine for this kind of personal/demo app, but swap in your own geocoder
  if you expect real traffic.
