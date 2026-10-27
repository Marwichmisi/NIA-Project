# ato/demo — baseline

Provenance : copie verbatim de `examples/quickstart` d'atopile au tag
`v0.14.3` (même tag que `requirements-ato.txt`). Ne pas éditer à la main :
c'est l'agent (prompt `02-montage.md`) qui générera les futurs `.ato`.

```bash
pip install -r ato/requirements-ato.txt   # Python >=3.14,<3.15 requis
cd ato/demo && ato build -t default        # rapide, sans KiCad
# avec KiCad : ato build -t glb,step,3d-image,2d-image,mfg-data
```
