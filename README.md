<img src="public/icon.svg" width="56" height="56" alt="Cloudflare Durable Objects presentation icon">

# Cloudflare Durable Objects presentation

A [Slidev](https://sli.dev/) slide deck that explains Cloudflare Durable Objects through practical examples, interactive Vue demos, and speaker notes. The content follows the Cloudflare Developers video [Durable Objects Finally Make Sense](https://www.youtube.com/watch?v=k4UXEfZf3sc).

## Run the presentation

Install dependencies and start the local Slidev server:

```bash
pnpm install
pnpm dev
```

Slidev starts at `http://localhost:3030`. Open `http://localhost:3030/presenter` to view speaker notes alongside your slides.

## Build and export

Build a static single-page app in `dist/`:

```bash
pnpm build
```

Export slides to PDF or PNG (install `playwright-chromium` first if prompted):

```bash
pnpm export
```

## Project structure

- `slides.md` — 13-slide presentation covering the mental model, HTTP + SQLite implementation, WebSocket hibernation concept and API, and partitioning patterns.
- `components/RoomRoutingDemo.vue` — Interactive simulator demonstrating deterministic `idFromName(roomId)` routing across edge locations.
- `components/HibernationConceptDiagram.vue` — High-level visual diagram showing how the runtime host (`Connection Manager`) keeps WebSockets open while an idle Durable Object sleeps at `$0` cost.
- `components/HibernationDemo.vue` — Interactive simulator showing WebSocket hibernation (`0 MB` idle memory) and automatic wake-and-broadcast with embedded SQLite.
- `style.css` — Light and dark theme variables and typography rules.
