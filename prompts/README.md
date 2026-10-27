# Prompts système — bibliothèque modifiable

Un fichier par rôle agent. Modifier un `.md` ici change le comportement de
l'agent **sans toucher au code**. Versionnés, relus avant chaque démo.

## Doctrine source (ne pas copier, citer)

La vérité technique vit dans `.agents/skills/atopile/` (skills officiels
atopile, verbatim, cachés à l'utilisateur). Chaque prompt ci-dessous cite ses
sections ; en cas de conflit, le skill officiel gagne.

| Prompt | Skills sources |
|---|---|
| `00-plan` | `planning` (spec IS design, front-load questions), `agent` (action bias, decision policy) |
| `01-document-plan` | `planning` (spec format, requirements), `ato` §2.5–2.6 (asserts, unités) |
| `02-montage` | `ato` §1–4 (process, patterns), `planning` (packages/parts, main.ato) |
| `03-preview` | `agent` (reports & manufacturing), `.pcba.*` artefacts |
| `04-questions` | `agent` (front-load, design_questions), `planning` Phase 1 |

## Correspondance outils officiels → notre runtime

L'agent officiel appelle des outils MCP ; le nôtre les mappe ainsi :

| Skill officiel | Notre runtime |
|---|---|
| `design_questions` | questions groupées du chat web (une salve, options recommandées) |
| `checklist_*`, `message_*` | suivi de tâches + statuts temps réel (style `agent-summary` : 1 ligne, présent, concret) |
| `project_read/edit/create_file` | outils fichiers du backend (jamais de commandes shell à l'utilisateur) |
| `parts_search/install`, `packages_search/install`, `stdlib_*`, `examples_*` | serveur MCP atopile (`src/atopile/mcp`) via backend, sinon CLI |
| `build_run`, `build_logs_search`, `design_diagnostics` | `ato build` + logs + diagnostics, sous-modules d'abord |
| `report_bom`, `report_variables` | BOM/variables lues depuis `output_base` (jamais inférées du source) |
| `manufacturing_*`, `layout_*` | targets `-t mfg-data/glb/step/...` + layout server, DRC après placement |
| `web_search` (datasheets) | recherche web backend avant de verrouiller un part à risque |

## Règles globales

- Pas de secrets dans ces fichiers.
- Ne jamais inventer de syntaxe `.ato` : `ato` skill §2 + §5 font foi.
- Toute sortie `.ato` montrée a un `ato build -t default` qui passe.
- Citer les `response.id` Meta dans la démo ; légender les images IA « illustratif ».
