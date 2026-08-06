# Cellopolis

*A mobile city-builder where cities grow from simple rules.*

Cellopolis is a mobile (Android / iOS) city-building game inspired by the classic DOS-era SimCity — not a remake, but a fresh take built on its own engine. You zone, connect, and power your city; the simulation does the rest, letting neighborhoods rise and fall from the bottom up. The name says it: the simulation core runs as a **cellular automaton**, where every map tile reacts to its neighbors — power floods across the grid, pollution spreads, land value ripples outward.

## Status

Early development — pre-alpha. Core simulation and tooling are being built.

## Design pillars

- **Field-based simulation** — the city is a stack of interacting map layers (power, traffic, land value, pollution, growth), not thousands of scripted agents. Robust, mobile-friendly, and true to the classic.
- **Simulation ≠ rendering** — the sim runs on plain data; the view only reads it and draws it.
- **Mobile-first** — touch-friendly controls, pinch-zoom and pan, and a UI designed for a phone rather than a ported desktop toolbar.

## Planned features (v1)

- Zoning: residential / commercial / industrial
- Roads and connectivity
- Power grid (flood-fill from power plants)
- Organic zone growth driven by R/C/I demand
- Budget and taxes

Later: pollution, land value, crime, disasters, and optional cosmetic agents (visible citizens and traffic).

## Tech

- **Engine:** Unity (C#)
- **Rendering:** Tilemap reading from the simulation grid
- **Performance:** plain managed arrays first; hot passes (diffusion, flood-fill) moved to Unity Jobs + Burst as needed

## Disclaimer

Cellopolis is an original work inspired by the city-building genre. It is **not affiliated with, endorsed by, or derived from** SimCity or Electronic Arts, and uses no SimCity code, assets, or trademarks.

## License

TBD