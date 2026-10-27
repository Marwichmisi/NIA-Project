# 02 — Montage (.ato)

Doctrine : `ato` §1 (process 4–5), §4 (patterns), §5 (dépannage dans
l'ordre : pragmas → imports → `new` → `~`/`~>` → références → asserts →
unités → picking → sur/sous-contraintes) ; `planning` (structure projet).

- Arborescence : `main.ato` (intégration, jamais de `_package` brut) ;
  `packages/<nom>/<nom>.ato` (wrappers IC génériques, interfaces stdlib) ;
  `parts/` (bruts installés) ; pas d'`ato.yaml` dans `packages/`.
- Réutiliser d'abord : `packages_search/install` (registre), `stdlib_list/
  get_item` avant toute `interface` custom, `examples_search/read_ato`.
- Local manquant : `parts_search` → datasheet via recherche web →
  `parts_install(create_package=true)` → wrapper affiné en place, passifs
  de découplage dedans, `assert` datasheet.
- Câbler aux interfaces (`~`, `~>` + pragma), toujours les `.reference`
  des bus logiques ; `Bridge`/`for`/`trait` seulement avec leur pragma.
- Valider par cibles : wrappers/sous-modules d'abord (en parallèle si
  indépendants), `main.ato` en dernier ; ne montrer que du build vert
  (`ato build -t default`, puis targets KiCad).
- Interdit : toucher grammaire/solver/exporters (pas de fork), inventer
  des features langage, fabriquer des résultats de build.
