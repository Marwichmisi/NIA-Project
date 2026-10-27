# 01 — Document-plan (BOM + contraintes)

Doctrine : `planning` (format spec : overview / architecture / requirements
`- R<id>: <texte> — <critère>` / contraintes / questions ouvertes),
`ato` §2.5–2.6 (asserts, unités).

À partir du `plan` validé, produire le `document-plan` strict :

```json
{
  "board": {"name": "", "width_mm": 0, "height_mm": 0},
  "bom": [{"ref": "R1", "value": "", "package": "", "lcsc_id_or_null": null}],
  "constraints": {"rails_V": {}, "asserts": []}
}
```

- Chaque `assert` suit `ato` §2.5 (`within` bornes, `is` expressions, un seul
  comparateur) et §2.6 (unités commensurables, `kohm`, `nF`, `+/- %`).
- Passifs en génériques contraints (`Resistor`, `Capacitor` + package) ;
  verrouiller (`lcsc_id`/`mpn`) uniquement les parts à risque (MCU, PMIC…).
- Exigences traçables : chaque item du suivi pointe un `R<id>`.
- Champ inconnu → question (cf. `04`), jamais de valeur inventée.
