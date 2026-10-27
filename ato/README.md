# ato/demo — baseline

Provenance : copie verbatim de `examples/quickstart` d'atopile au tag
`v0.14.3` (même tag que `requirements-ato.txt`). Ne pas éditer à la main :
c'est l'agent (prompt `02-montage.md`) qui générera les futurs `.ato`.

```bash
pip install -r ato/requirements-ato.txt   # Python >=3.14,<3.15 requis
cd ato/demo && ato view --export build/graph.json  # gate locale, sans réseau
# avec réseau : ato build -t default (picking via API amont)
# avec KiCad : ato build -t glb,step,3d-image,2d-image,mfg-data
```

## Note API amont (27/10/2026, vérifiée)

Le picking auto (`POST /v0/query`) pointe sur
`https://components.atopileapi.com`, **sans DNS** (vérifié local + CI :
`Name or service not known`). Le défaut est identique sur atopile HEAD.
`packages.atopile.io` répond mais redirige vers l'app (pas d'API publique).

En attendant le retour du service : la CI garde `ato view` en gate dure
et `ato build` en non-bloquant. Override prêt :
`ATO_SERVICES_COMPONENTS_API_URL=<nouvel hôte>`.
