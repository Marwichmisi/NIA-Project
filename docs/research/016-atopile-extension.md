# Recherche #16 — Extension atopile sans fork + visualisation / preview 3D

> Part of #15 (map). Ticket : `Marwichmisi/NIA-Project#16`.
> Date : 2026-10-27. Source primaire : `atopile/atopile` @ `619eda7` (clone depth-1 du 27/10/2026), `docs.atopile.io`, code `ato`, exporters, serveur, MCP.
> Branche : `research/atopile-extension` (throwaway, aucun code principal modifié).

## Réponse courte (oui/non)

- **Extension sans fork : OUI.** Sur-couche viable par : CLI `ato` en sous-processus, API Python (`atopile.buildutil` + `atopile.config` + `faebryk.*`), serveur FastAPI (`src/atopile/server`), MCP (`src/atopile/mcp`), LSP (`ato lsp start`), registry packages (`ato.yaml` + `ato dependencies`), layout-server standalone. Aucun ne demande de modifier atopile.
- **Preview visuelle/3D native : OUI.** `ato` produit déjà : graphe interactif (`ato view`), SVG 2D (`.pcba.svg`), rendu PNG 3D (`.pcba.png`), GLB (`.pcba.glb`), STEP (`.pcba.step`), Gerber/ZIP, DXF, BOM CSV+JSON, `variables.json`, `pcb_summary.json`, netlist `.net`, layout `.kicad_pcb` visualisable via KiCanvas / GerberViewer / ModelViewer / StepViewer / layout-editor WebGL2 / VSCode / `app.atopile.io`.
- **Muse Image : à brancher CÔTÉ SUR-COUCHE, pas dans ato.** `ato` fournit les captures + maillages sources ; `muse-image`/`wonder` génère la couche créative (maillage semi-transparent, boîte d'enclosure). Aucun fork requis pour ça.
- **Fork requis UNIQUEMENT si** : grammaire ANTLR, solver Zig, nouveau target `muster` first-class, comportement layout-editor/LSP/serveur au cœur, ou outputs hors kicad-cli.

## 1. Points d'extension sans fork (fichiers / APIs exacts)

### 1.1 CLI `ato` en sous-processus (recommandé pour la démo hackathon)

Entrée : `src/atopile/cli/cli.py` — `app = typer.Typer(...)`, `ato = "atopile.cli.cli:main"` (`pyproject.toml` `[project.scripts]`).

| Commande | Fichier | Usage sur-couche |
|---|---|---|
| `ato build [<entry>] -b <build> -t <target> -x <excl> [--frozen] [--keep-*]` | `src/atopile/cli/build.py` (`discover_projects`, `_run_build_queue`, `_build_all_projects`) | Build sandboxé par projet ; `-t` sélectionne `bom`, `glb`, `step`, `3d-image`, `2d-image`, `mfg-data`, `default`, `all` sans toucher au code |
| `ato view [--export graph.json] [--serve] [--port 8765]` | `src/atopile/cli/view.py` (`view()`, `_serve_visualizer()`) | Graphe logique/schéma-bloc ; `--export` donne le JSON, `--serve` ouvre le visualiseur |
| `ato mcp start [--http] [--debug]` | `src/atopile/cli/mcp.py` + `src/atopile/mcp/mcp_server.py` (`run_mcp()`) | Expose l'agent atopile à notre backend (stdio ou streamable-http) |
| `ato serve backend [--port 8501] [-w <workspace>]` / `ato serve frontend` | `src/atopile/cli/serve.py` → `src/atopile/server/server.py` (`create_app()`, `run_server()`) | Réutilise le dashboard existant ou l'embarque en iframe |
| `ato lsp start` | `src/atopile/cli/lsp.py` → `src/atopile/lsp/lsp_server.py` (`LSP_SERVER`, pygls) | Diagnostics/complétion/hover/goto/format dans VSCode ou Monaco custom |
| `ato validate <file.ato>` | `src/atopile/cli/cli.py` (`validate()`) | Check syntaxe avant génération IA |
| `ato create / ato dependencies add|sync / ato inspect` | `src/atopile/cli/create.py`, `install.py`, `inspect_.py` | Scaffolding projet + gestion registry |
| `ato dev worktree <suffix>` | `src/atopile/cli/dev.py` | Isole les essais sans polluer `main` |

Contrat sortie : exit `0` succès / `1` échec build / `130` annulé (`cli.py::main()`).

### 1.2 API Python (même process, sans fork)

- `src/atopile/buildutil.py` : `build(app?, ctx?) -> fabll.Node`, `init_app() -> fabll.Node` (montage minimal jusqu'à `post-instantiation-setup`, utilisé par `ato view`), `BuildStepContext` / `BuildContext`, `run_build_targets()`, `generate_build_id()` / `generate_build_timestamp()`.
- `src/atopile/config.py` : `config.apply_options(entry, selected_builds, include_targets)`, `config.select_build(name)`, `ProjectConfig.from_path()`, `BuildTargetConfig` (`address` alias `entry`, `build_type`), `BuildTargetPaths` (`layout`, `output_base`, `netlist`, `kicad_project`, `documentation`), `BuildType.ATO | PYTHON`, `ServicesConfig` (registry URLs, cf §1.6).
- `src/atopile/compiler/build.py` : `build_file(g, tg, import_path, path)`, `build_stage_2(...)`, `Linker`, `StdlibRegistry(tg).get("Resistor")`, `import_fabll_type(path, symbol)` (voie `BuildType.PYTHON` : entry `*.py:*` = sous-classe `faebryk.core.node.Node`).
- `src/atopile/address.py` : `AddrStr` (`file.ato:Entry::instance.path`), `get_file/get_entry_section/get_instance_section/from_parts`.
- `src/atopile/layout.py` : `SubPCB`, `has_subpcb`, `in_sub_pcb` (multi-PCB).
- `src/faebryk/` : `faebryk.library._F` (stdlib `Resistor`, `Capacitor`, …), `faebryk.core.node/graph`, `faebryk.core.solver.solver.Solver`, `faebryk.core.graph_export.export_graph_to_json` (utilisé par `ato view`), `faebryk.exporters.*` (cf §2.2).
- Pattern éprouvé (copié de `src/atopile/mcp/tools/cli.py::build_project`) : `config.apply_options(...)` → `config.select_build(...)` → `buildutil.build(ctx=BuildStepContext(...))` avec `ATO_BUILD_ID` + `BuildHistory`.

Contrainte : `requires-python = ">=3.14,<3.15"`, `antlr4-python3-runtime==4.13.2` strict, `ziglang==0.15.1` (cf §4).

### 1.3 Serveur FastAPI existant (à consommer, pas à forker)

`src/atopile/server/server.py` : `create_app()` / `run_server()` ; routers montés dans `create_app()` (~l.636-649).

| Domaine | Fichier route | Endpoints clés |
|---|---|---|
| builds | `routes/builds.py` | `POST /api/build`, `GET /api/build/{id}/status`, `/api/builds/active|history|queue`, `/api/summary` |
| artefacts | `routes/artifacts.py` + `domains/artifacts.py` | `GET /api/bom`, `/api/bom/targets`, `/api/variables`, `/api/resolve-location` |
| manufacturing | `routes/manufacturing.py` + `domains/manufacturing.py` (`BuildOutputs`) | `gerbers, bom_json/csv, pick_and_place, step, glb, kicad_pcb, kicad_sch, pcb_summary` ; download + `CostEstimate` |
| fichiers | `routes/files.py` | sert `.kicad_sch` (`application/x-kicad-schematic`), `.kicad_pcb`, build outputs ; `file_watcher.py` surveille `*.kicad_sch` |
| layout | `routes/layout.py` → `layout_server/server.py::create_layout_router` | `GET /api/layout/render-model`, `/footprints`, `POST /execute-action`, `WS /ws/layout` ; `LayoutService` + `PcbManager.get_render_model()` |
| websocket/logs | `routes/websocket.py`, `routes/logs.py` | `WS /ws/state` (events `builds_changed`, `bom_changed`, `open_3d/open_kicad/open_layout`), `WS /ws/logs` streaming |
| packages/parts/problems | `routes/packages.py`, `parts*.py`, `problems.py`, `tests.py` | `GET /api/packages/summary`, parts-search JLCPCB, problèmes/ERC |

Note `DATA_FLOW.md` : pattern dominant = `WS event vide → refetch HTTP`. Notre front Next.js peut donc soit interroger le REST directement, soit écouter le WS.

### 1.4 MCP (agent atopile prêt à l'emploi)

`src/atopile/mcp/mcp_server.py` : `FastMCP("atopile", stateless_http=True)`, `mcp.run(transport="stdio"|"streamable-http")`.

Outils (`src/atopile/mcp/tools/*.py`, `util.py::MCPTools.register`) :

- `cli.py` : `build_project(absolute_project_dir, target_name_from_yaml) -> BuildResult`, `search_and_install_jlcpcb_part(...)`, `install_package(...)`.
- `library.py` : `get_library_modules_or_interfaces(include_modules, include_interfaces)`, `inspect_library_module_or_interface(name) -> NodeInfo`.
- `packages.py` : `find_packages(query)`, `inspect_package(identifier, version?)` via `PackagesAPIClient`.
- `project.py` : `find_project_from_filepath(absolute_path_to_file)`.

Branchement : lancer `ato mcp start --http` comme sidecar, ou importer `run_mcp()` / appeler les fonctions `handle_*` des domains depuis notre FastAPI.

### 1.5 LSP / VSCode (édition `.ato` sans fork)

- `src/atopile/lsp/lsp_server.py` : `LSP_SERVER` (pygls, `textDocumentSync=Full`, `MAX_WORKERS=5`) — diagnostics, complétion (`dot/new/import`), hover, goto-def, actions (auto-import), références, formatage.
- `src/vscode-atopile/` : extension (`package.json` → commandes `atopile.*`), webviews, `resources/kicanvas`, `resources/model-viewer/model-viewer.min.js`, `resources/templates`.
- `src/ui-server/` : dashboard Vite/React de référence (cf §2.3).

### 1.6 Registry packages (import `.ato` + dépendances)

- `ato.yaml` : `dependencies: [{type: registry, identifier: atopile/buttons, release: 0.4.0}]` (cf `examples/esp32_minimal/ato.yaml`), `builds.<nom>.entry: "file.ato:Module"`, `paths.src/layout`.
- Client : `src/faebryk/libs/backend/packages/api.py::PackagesAPIClient` (`get_package`, `query_packages`, `get_all_packages`), base `config.project.services.packages.url = https://packages.atopileapi.com` ; composants `https://components.atopileapi.com` (`config.py::ServicesConfig`).
- CLI : `ato dependencies add <id>`, `ato sync`, `ato package ...` ; MCP : `install_package`, `find/inspect_package`.
- Sur-couche : générer `ato.yaml` + `.ato` avec `import` registry, puis `ato sync && ato build`. Ne jamais éditer le cache `.ato/modules`.

### 1.7 Layout-server standalone (édition PCB sans fork)

`src/atopile/layout_server/` : `server.py` + `pcb_manager.py` + `models.py` + `frontend/` (WebGL2 : `renderer.ts`, `painter.ts`, `editor.ts`, `layout_client.ts`) + `README.md`.
Lancement : `python -m atopile.layout_server path/to/board.kicad_pcb` → `http://localhost:8100` (zoom/pan/select/drag `R` rotate / `F` flip / undo-redo). Embeddable en iframe ou via `create_layout_router(service)` dans notre backend.

## 2. Visualisation : ce que `ato` permet déjà

### 2.1 `ato view` (schéma-bloc / graphe)

`src/atopile/cli/view.py` : `init_app()` → `export_graph_to_json(app.instance, out.json)` (`faebryk.core.graph_export`) → soit `--export graph.json`, soit `--serve` (serveur `http.server` sur `--port 8765`, ouvre le navigateur).
Front : `src/atopile/visualizer/web/` (Vite + React + Tailwind + `three/`, `workers/`, `stores/`, `components/Inspector|Minimap|ExportMenu`). Si `dist/` absent, fallback JSON seul (message explicite dans `_serve_visualizer`). Idéal pour debug graphe, pas pour schéma KiCad.

### 2.2 Exporters fichier (ිles clés pour preview + Muse Image)

Cœur : `src/faebryk/exporters/pcb/kicad/artifacts.py` (via `kicadcliwrapper` → `kicad-cli`, tag `Tags.REQUIRES_KICAD` dans `build_steps.py`) + `src/faebryk/exporters/bom|documentation|parameters|power_tree`.

| Artefact | Target `muster` (`src/atopile/build_steps.py`) | Fichier sortie (`output_base.*`) |
|---|---|---|
| BOM picking | `bom` → `write_bom` (JLCPCB CSV) + `write_json_bom` (`.bom.json` + `build_id`) | `.bom.csv`, `.bom.json` |
| GLB 3D (avec tracks/zones/pads/soldermask/silkscreen) | `glb` / `3d-model` + `glb-only` (rapide, sans rebuild) → `export_glb` | `.pcba.glb` |
| STEP 3D | `step` → `export_step` (`subst_models`, `KIPRJMOD`) | `.pcba.step` |
| Rendu PNG 3D | `3d-image` → `export_3d_board_render` (1000×1000, `quality=high`, `rotate=335,0,-45`) | `.pcba.png` |
| SVG 2D | `2d-image` → `export_svg` (`F.Cu,F.Paste,F.SilkS,F.Mask,Edge.Cuts`) | `.pcba.svg` |
| Gerber + drill + ZIP | `mfg-data` → `export_gerber` (+ Excellon) | `.gerber.zip` |
| DXF outline | `mfg-data` → `export_dxf` (`Edge.Cuts`, mm) | `.pcba.dxf` |
| Pick&place (+JLCPCB), testpoints, summary | `mfg-data` → `export_pick_and_place`, `convert_..._jlcpcb`, `export_testpoints`, `export_pcb_summary` | `.pick_and_place.csv`, `.jlcpcb_pick_and_place.csv`, `.testpoints.json`, `.pcb_summary.json` (dims + stackup) |
| Netlist, manifest, variables, power-tree, datasheets | `default` (`bom`, `manifest`, `variable-report`, `datasheets`) | `.net`, `manifest.json`, `.variables.json`, `power_tree.md`, `documentation/datasheets` |
| Layout KiCad | `update_pcb` / `post_pcb_checks` → `LayoutSync`, `check_design`, ERC/DRC | `elec/layout/<build>/<build>.kicad_pcb` + `.kicad_pro`, `.kicad_sch` (si existant, servi par `domains/manufacturing.py::BuildOutputs`) |

Sélecteur : `ato build -t glb -t step -t 3d-image -t 2d-image -t mfg-data` (DAG `muster`, `build_steps.py::Muster.select/topologically_sorted`). `default` = BOM+manifest+variables+datasheets ; `all` = `default` + `mfg-data` + `3d-models`.

### 2.3 Viewers existants (à réutiliser / embarquer)

- `src/ui-server/src/components/` : `KiCanvasEmbed.tsx` (charge `vendored/kicanvas.js`, schématique + PCB), `GerberViewer.tsx`, `ModelViewer.tsx` (`model-viewer.min.js`, slider `soldermaskOpacity` — précédent de semi-transparence), `StepViewer.tsx` (`three` + `OrbitControls` + `occt-import-js` WASM), `manufacturing/BuildReviewCard.tsx` (GLB + KiCanvas côte à côte), `manufacturing/ManufacturingPanel.tsx` (Gerber + GLB).
- `layout_server/frontend` : éditeur WebGL2 (board edges, tracks, vias, zones, footprints).
- `visualizer/web` : graphe 3D du circuit logique.
- Externe : extension VSCode (`open_3d/open_kicad/open_layout` via WS), `app.atopile.io` (workspace navigateur, cf `docs.atopile.io` : « docs live in the editor now »).

Pas de schéma SVG généré par `ato` au sens `kicad_sch` synthétisé : le `.kicad_sch` n'existe que si le projet en fournit un ; la vérité schématique côté atopile = graphe `ato view` + netlist `.net` + variables.

## 3. Branchement Muse Image (capture + maillage semi-transparent, boîte 3D)

Principe : `ato` = source technique, `muse-image`/`wonder` = couche créative. Tout se fait dans notre Next.js/FastAPI, zéro fork.

1. **Collecte post-build** (après `ato build -t all` ou ciblé) : lire `output_base` → `.pcba.png` (vue orbitale 335°), `.pcba.svg` (2D), `.pcba.glb` (maillage), `.pcba.step` (CAO), `.pcb_summary.json` (dims X/Y, stackup), `.bom.json` (composants), `.kicad_pcb` (fallback KiCanvas).
2. **Preview immédiate (sans IA)** : afficher `.pcba.png` + `<ModelViewer src=.glb>` + `<StepViewer>` + `<KiCanvasEmbed>` + `<GerberViewer>`. Répond au « oui preview 3D » du ticket sans appeler Meta.
3. **Maillage semi-transparent** : (a) natif gratuit — `ModelViewer` expose déjà `soldermaskOpacity`/`setAlphaMode(BLEND)` : slider d'opacité ; (b) IA — envoyer `.pcba.png` (+ capture GLB orbitale via canvas) à `muse-image` avec prompt « semi-transparent mesh overlay / X-ray exploded traces » ; le modèle retourne une image stylisée qu'on affiche à côté de la capture brute (traçabilité démo).
4. **Boîte 3D / enclosure** : extraire l'outline (`Edge.Cuts` via `.pcba.dxf` + dims `pcb_summary.json` + hauteur max des composants BOM) → prompt `muse-image`/`wonder` « minimal enclosure box for <W×H×D>mm board, mounting holes, USB cutout » → afficher rendu à côté du STEP. Optionnel : convertir le rendu en fond de carte démo, pas en CAO fab.
5. **API Meta** : `muse-spark` génère `plan → document-plan → .ato` ; `muse-image` ne reçoit que des PNG/GLB-captures + dims, jamais le `.ato` brut (limite la fuite IP + coût tokens). Mettre en cache les captures par `build_id` (`generate_build_id()`).

## 4. Limites exigeant un fork (à éviter sauf fallback démo)

1. **Grammaire `.ato`** : `src/atopile/compiler/parser/AtoLexer.g4` + `AtoParser.g4` (+ générés `AtoLexer.py/AtoParser.py`, runtime `antlr4-python3-runtime==4.13.2` strict, `tools/atopile_mkdocs_plugin`). Nouveau mot-clé / syntaxe = fork + régénération ANTLR.
2. **Solver / sémantique** : `src/faebryk/core/solver/*` + `src/faebryk/core/zig/*` (`.zigversion` 0.15.1, `ziglang==0.15.1` via `scikit-build-core` forké atopile). Nouvelles équations/unités/checks cœur = fork + toolchain Zig.
3. **Nouveau target first-class** : `src/atopile/build_steps.py::muster` (`MusterTarget`, `dependency_dag`, `Tags.REQUIRES_KICAD`). Sans fork on ne peut que *sélectionner* (`-t/--exclude-target`) ou envelopper par script ; ajouter un target (ex. `enclosure`) = éditer `build_steps.py`.
4. **Sorties hors kicad-cli** : tout export non listé §2.2 (ex. vrai `.kicad_sch` synthétisé, PDF schéma, images custom angle/résolution autre que `export_3d_board_render`) = étendre `artifacts.py` (fork) ou post-traiter côté sur-couche (préféré : Blender/`occt` sur le STEP/GLB).
5. **Comportement serveur/LSP/layout internes** : auth `fastapi-github-oidc`, `domains/*`, `problem_parser.py`, handlers `lsp_server.py`, moteur `layout_server/pcb_manager.py`, `kicad_plugin/`, télémétrie `posthog==7.8.0` (`telemetry.py`). Consommer via REST/WS/MCP ; changer le comportement = fork.
6. **Fronts embarqués** : `visualizer/web/dist`, `layout_server/frontend/dist`, `ui-server/dist`, `vscode-atopile/resources/*` — rebuild npm/webpack ; customiser sans fork = iframe + CSS overlay côté Next.js.
7. **Compatibilité** : `requires-python >=3.14,<3.15`, `uv.lock`, `dockerfiles/`. Le sandbox démo doit installer `kicad-cli` séparément sinon les targets `REQUIRES_KICAD` sont skippés avec warning (`buildutil.py::_check_kicad_cli`).

Licence : MIT (`LICENSE`) — fork autorisé mais non nécessaire ; préférer l'extension (cf `AGENTS.md` / map #15).

## 5. Recommandation sur-couche (hackathon 27/10–04/11)

- Backend FastAPI : `POST /plan → document-plan` (muse-spark) → génère `ato.yaml` + `.ato` (imports registry) → `ato sync` → `ato build -t default` (rapide, sans KiCad) puis `-t glb,step,3d-image,2d-image,mfg-data` si KiCad présent → sert artefacts + `GET /api/bom|variables` (proxy ou lecture `output_base`).
- Front Next.js/shadcn : étapes `plan → document-plan → montage (.ato)` + onglets preview (PNG/SVG/GLB/STEP/KiCanvas/Gerber) + boutons « mesh semi-transparent » et « boîte 3D » → `muse-image`.
- Agent : `muse-spark` orchestre ; `ato mcp start --http` en sidecar pour `build_project` / `inspect_library` / `find_packages` ; fallback CLI si MCP indisponible.
- Exemple démo : `examples/esp32_minimal` ou `examples/led_badge` (cf `examples/`).

## Sources

- Repo : `https://github.com/atopile/atopile` (`619eda7`, 27/10/2026) — `src/atopile/cli/{cli,build,view,mcp,serve,lsp}.py`, `src/atopile/{buildutil,build_steps,config,address,layout}.py`, `src/atopile/compiler/build.py`, `src/atopile/compiler/parser/{AtoLexer,AtoParser}.g4`, `src/atopile/mcp/mcp_server.py`, `src/atopile/mcp/tools/{cli,library,packages,project}.py`, `src/atopile/server/{server,DATA_FLOW.md,routes/*,domains/manufacturing.py}`, `src/atopile/lsp/lsp_server.py`, `src/atopile/layout_server/{server,pcb_manager,README.md}`, `src/atopile/visualizer/web/`, `src/faebryk/exporters/pcb/kicad/artifacts.py`, `src/faebryk/exporters/bom/{jlcpcb,json_bom}.py`, `src/faebryk/libs/backend/packages/api.py`, `src/ui-server/src/components/{KiCanvasEmbed,ModelViewer,StepViewer,GerberViewer,manufacturing/*}.tsx`, `src/vscode-atopile/`, `examples/{esp32_minimal,led_badge}/ato.yaml`, `pyproject.toml`, `.zigversion`, `LICENSE` (MIT).
- Docs : `https://docs.atopile.io/` (workspace `app.atopile.io`), `https://packages.atopile.io`.
