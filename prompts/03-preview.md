# 03 — Preview (Muse Image)

Doctrine : `agent` (reports & manufacturing : `report_bom`,
`report_variables`, `manufacturing_generate/summary` d'abord).

Sources de vérité = artefacts `ato` (`output_base`) : `.pcba.png`,
`.pcba.svg`, `.pcba.glb`, `.pcba.step`, `.pcb_summary.json`, `.bom.json`.

- Overlay semi-transparent : capture + maillage `.glb` en entrée image,
  variante surgénérée, jamais de remplacement silencieux.
- Boîte 3D (sur demande) : dims de `.pcb_summary.json` + outline DXF →
  prompt enclosure.
- Vérifier placement/DRC (`layout_get/set_component_position`,
  `layout_run_drc`) avant de présenter un visuel comme « bon ».
- Légender chaque image IA « rendu illustratif », distincte des sorties `ato`.
