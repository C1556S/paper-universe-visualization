# Angular Paper Universe Visualization

This repository contains the implementation plan, architecture decisions, and recommended tech stack for an interactive scientific paper knowledge map ("paper universe"). It focuses on building an Angular 18 application with D3 force simulation for dynamic paper positioning and a performant SVG + Canvas hybrid rendering approach.

## Executive summary

Goal: build a highly interactive map of scientific papers where each paper is a node and citation relationships are links. Positions must be dynamically calculated using a D3 force simulation (not pre-rendered tiles). The UI should support smooth pan/zoom, selection, hover, drag, search and filter, and should scale to thousands of papers while remaining responsive.

Key decisions
- Use Angular 18 (Standalone Components) + TypeScript (strict) for app structure, DI, and reactive state.
- Use D3.js (v7+) for force simulation and for SVG manipulation where appropriate.
- Use an SVG primary interactive layer for nodes/links/labels and optional Canvas for background or very large node sets.
- Progressive paging for API: fetch page 1 and render quickly, fetch remaining pages in background (Option A chosen).
- Use RxJS for reactive data flow and OnPush change detection to maintain performance.

Why not Paperscape's tile-based approach
- Paperscape pre-renders tiles (bitmaps) of positions for fast tile-based navigation. That is great for static views but incompatible with dynamic force simulation and interactive node-level events.
- Tile system adds complexity (tile servers, label zones, caching) and reduces ability to dynamically update layout.
- For real-time layout, an SVG+D3 approach is simpler, more flexible, and better for node-level interactivity.

Design overview

- Data layer: REST API with pagination; PaperDataService merges pages transparently and exposes BehaviorSubject<Paper[]>.
- Computation layer: D3ForceService runs forceSimulation, emits tick events/positions via an Observable so renderer updates.
- Rendering layer: PaperSvgComponent uses D3 selections bound to the SVG DOM. Links drawn as <line>, nodes as <circle>, labels as <text>.
- Interaction layer: Zoom/Pan via D3.zoom on the SVG root; selection and hover handled by D3 events and shared SelectionService.
- Search/Filter layer: SearchService applies filters to Paper[] and highlights results; integration with backend search endpoint for robust queries.

Data model (recommended)

Paper:
- id: string
- title: string
- x: number | null
- y: number | null
- size: number
- color: string
- cites: string[] (array of target paper ids)
- metadata: {authors, year, journal, categories, ...}

Link (derived):
- source: string | Paper
- target: string | Paper
- value?: number

API contract (recommended)
- GET /papers?page=1&limit=1000 → { papers: [...], pagination: {page, totalPages, total, pageSize} }
- Support batch endpoints for locations/metadata: POST /papers/locations (ids)
- Search endpoints: /search?query=..., and specialized queries for author/title/cites/refs

Progressive loading strategy (Option A)
1. Fetch page 1 (e.g. first 1000 papers). Render immediately and start force simulation.
2. In background, fetch remaining pages (2..N) in sequence or in parallel with rate-limiting.
3. Merge new papers into the existing dataset; add nodes/links to simulation and restart/alpha( ) to re-balance layout.
4. Keep UI unaware of pagination: PaperDataService exposes a single Observable<Paper[]> that grows as pages load.

D3 force simulation patterns
- Use d3.forceSimulation(nodes)
  - forces: forceLink(links).distance(d), forceManyBody().strength(s), forceCenter(width/2, height/2), forceCollide(radius)
- For thousands of nodes tune:
  - alphaStart / alphaDecay — start cooler or warmer based on size
  - limit number of ticks or pause simulation when converged
  - consider running simulation in a Web Worker for 10k+ nodes
- When merging pages: add new nodes/links to the simulation via simulation.nodes(...) and simulation.force('link').links(...), then call simulation.alpha(0.3).restart()

Rendering strategy
- Use SVG for interactive node/link/label elements. SVG keeps DOM-level events simple and CSS styling easy.
- For very large node counts where rendering all SVG elements is too slow, use a hybrid:
  - Canvas for background rendering of many nodes (no DOM events) and SVG for highlighted/interactive subset.
  - Or use WebGL rendering libraries (e.g. regl, pixi.js) if scaling to 10k+ interactive nodes is required.
- Use D3 selections to join data and update attributes on tick events rather than Angular templates for per-node rendering.
- Use requestAnimationFrame for any manual DOM or Canvas updates.

Interaction patterns
- Zoom & pan: d3.zoom attached to SVG root. Update a top-level <g> transform for pan/zoom. Keep simulation coordinates in world coordinates.
- Click/hover: use quadtree spatial index (d3.quadtree) to find nearest node for hit testing instead of iterating all nodes.
- Drag: d3.drag connected to nodes to fix positions while user drags; on drag end optionally release or fix node.
- Selection: SelectionService exposes selected paper and provides APIs to highlight neighbors (cites/cited-by).

State management & performance
- Use OnPush change detection everywhere for components that receive data via Observables.
- Let D3 mutate the DOM; avoid Angular change detection for per-tick updates.
- Use BehaviorSubject for papers and links, derive observables for subsets (visible nodes, search results).
- Use quadtree + index maps for O(log n) hit detection and fast neighbor queries.

File structure (recommended)

src/app/
  features/paper-universe/
    components/
      paper-universe.component.ts  (container)
      paper-svg.component.ts       (SVG + D3 renderer)
      search-bar.component.ts
      paper-detail.component.ts
    services/
      paper-data.service.ts        (REST + pagination)
      paper-link.service.ts        (graph helpers)
      d3-force.service.ts          (simulation)
      zoom-pan.service.ts         (d3 zoom wrapper)
      selection.service.ts
      search.service.ts
    models/
      paper.model.ts
      link.model.ts

Implementation roadmap (broken into smaller tasks)

Phase 0 — Repo & scaffolding
- Task 0.1: Create repository and initialize package.json and Angular workspace
- Task 0.2: Install dependencies: Angular 18, d3, @types/d3, rxjs, @angular/cdk

Phase 1 — Core data plumbing
- Task 1.1: Create Paper model and Link model
- Task 1.2: Implement PaperDataService: fetch page 1, expose BehaviorSubject<Paper[]>, background load remaining pages (Option A)
- Task 1.3: Implement PaperLinkService: derive links from papers.cites

Phase 2 — D3 simulation
- Task 2.1: Implement D3ForceService: create simulation, add forces, expose tick Observable
- Task 2.2: Integrate with PaperDataService: build nodes/links and start simulation when page1 loaded
- Task 2.3: Handle incremental additions: add nodes/links and restart simulation

Phase 3 — SVG rendering & interactions
- Task 3.1: Create PaperSvgComponent: SVG container, <g id="zoom-root">
- Task 3.2: Bind nodes and links with D3 enter/update/exit
- Task 3.3: Attach d3.zoom to svg and update transform on events
- Task 3.4: Implement node hover (tooltip), click (detail panel), drag
- Task 3.5: Implement quadtree for hit-testing on canvas clicks

Phase 4 — Search, selection, and UI
- Task 4.1: SearchService with local filtering + backend search API
- Task 4.2: SelectionService to highlight neighbors
- Task 4.3: Detail panel and info view fetching metadata on demand

Phase 5 — Performance and polish
- Task 5.1: Tune force parameters per dataset size
- Task 5.2: Add Web Worker option for simulation if needed
- Task 5.3: Add Canvas hybrid rendering for background nodes if needed
- Task 5.4: Add instrumentation and profiling guide

Performance expectations
- Initial render (page 1, ~1000 nodes): 1–3s depending on network & machine
- Background merging of additional pages: non-blocking; each page merge may require a short re-layout
- Interactions (pan/zoom/hover) should remain 30+ FPS on modern desktops for a few thousand nodes
- For 10k+ interactive nodes consider WebGL or canvas hybrid

Repository files to commit
- README.md (this file)
- docs/architecture.md (expanded details)
- docs/implementation-roadmap.md
- src/ (starter skeleton)
- examples/ (minimal demo app)

Next steps I will perform (if you confirm):
1) Replace README.md in the repository with the focused implementation & tech stack content above and commit it.  
2) Create additional docs in docs/ as separate markdown files (architecture, roadmap, simulation tuning).  
3) Add starter Angular scaffold files and example D3ForceService and PaperSvgComponent templates.

Please confirm you want me to commit the README.md now and continue with the other files, or specify which of the next steps to prioritize.