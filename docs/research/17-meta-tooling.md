# Recherche #17 — Usage Meta créatif (Spark + Image/Wonder) et conformité judging

Part of #15 — Map : Concepteur PCB assisté par IA sur atopile (Hackathon Meta Global AI / Push the Frontier, deadline 4 nov 21h ET).
Ticket : #17 — Quelle combinaison Meta minimale et créative maximise Use of Meta AI Tooling (25%) tout en passant Stage One.

Date : 2026-10-26. Sources : primaires `dev.meta.ai` / `developer.meta.com` (URLs ci-dessous). Page Devpost du hackathon non résolvable au moment de l'enquête (404 sur les slugs `meta-global-ai-developer-hackathon` et `push-the-frontier`) — les critères Impact / Use of Meta AI Tooling / Demo / Technical Depth, crédits 150 $, vidéo 3 min, repo public, new-work-only sont repris de l'énoncé du ticket #17 + map #15 et marqués `[ticket]` quand non vérifiables en primaire. À revalider dès que l'URL Devpost officielle est connue (action #18).

---

## 1. Réponse courte — combinaison recommandée

**Cœur (obligatoire, à instrumenter) :**

1. `muse-spark-1.3` (Standard) — `plan -> document-plan -> .ato`. C'est le modèle attendu par la map.
2. `muse-image-1.0` — preview/rendu créatif (capture + maillage semi-transparent, boîte 3D sur demande).
3. `sam-3.1` — détourage/mesure/tracking du PCB, en amont de Muse Image.

**Appui (bon marché, différenciant) :**

4. `muse-voice-transcribe-1.0` — dictée du `plan` vocal + preuve voix-dans-la-boucle (optionnel, quasi-gratuit).
5. `Muse Glimmer-30B` (open-weight, Apache 2.0, local) — preuve open-weight self-hosted sans cluster + fallback offline pour la démo.

**À éviter / à isoler :**

- `muse-wonder-1.0` — uniquement 1–2 appels isolés si besoin frontier (refactor multi-fichiers bloquant). Coût 3–5× Spark, preview, Standard-only.
- `Muse Spark Open 816B` — citer la config, ne pas déployer pendant le hackathon (530 Go en NVFP4, 4× Blackwell Ultra ou 8× H200, ou 2–4× Mac Studio). Trop lourd pour 8 jours. Si le jury exige une preuve open-weight hébergée, passer par un hosted provider (Together/Fireworks) mais ce n'est pas du self-hosted.
- `muse-spark-1.3-contributor` — pour les itérations dev uniquement (prix /12), jamais pour les runs filmés (contenu réutilisable pour entraînement).

Pourquoi pas Wonder au cœur : même endpoints et mêmes requêtes que Spark (changement = un `model` ID), mais Wonder vise les tâches longues/complexes où le coût compte moins que la capacité, Spark vise le haut-volume/latence/coût avec mode Contributor disponible. Sources : choix Wonder-vs-Spark + pricing + availability.

---

## 2. Ce que dit la doc primaire (avec prix et endpoints)

### 2.1 Muse Spark 1.3

- Page modèle : https://dev.meta.ai/models/muse-spark/ — entraîné pour agentic workflows, 1M contexte, multimodal natif (image/vidéo/document), raisonnement avec effort réglable dont `max`.
- Docs : https://dev.meta.ai/docs/ — Base URL `https://api.meta.ai/v1`, auth Bearer `MODEL_API_KEY`, compatible OpenAI SDK / Anthropic SDK / OpenAI-compatible agents. `muse-spark-1.3` est le défaut des exemples.
- Versions : https://dev.meta.ai/docs/models — `muse-spark-1.3` (recommandé new work), `1.2`, `1.1`. Tous texte/image/vidéo/audio/PDF en entrée, texte en sortie, 1M tokens. Note : audio-in sur 1.3 dégradé — pour l'audio utiliser 1.2 ou Voice Transcribe.
- Reasoning : https://dev.meta.ai/docs/reasoning — effort `minimal` → `max`. `max` indisponible sur `muse-spark-1.3-contributor`.
- Pricing : https://dev.meta.ai/docs/pricing-rate-limits
  - Standard : input $1.25 / cached $0.15 / output $4.25 par 1M tokens.
  - Contributor : input $0.10 / cached $0.002 / output $0.20 par 1M tokens (contenu réutilisable pour entraînement).
  - Pas de surcharge long-contexte. Search grounding `$2.50 / 1 000 requêtes` en plus (sauf Image dont la recherche est incluse).
- Rate limits : Standard 3 000 RPM / 4M TPM ; Contributor 150 RPM / 3M TPM. Headers `x-ratelimit-*`. Background responses 600/min.
- Muse Code : Spark 1.3/1.2 (+contributor) dispo en pay-as-you-go et toutes subscriptions ; 1.1 API-only.

### 2.2 Muse Wonder 1.0 / Wonder Full

- Annonce : https://developer.meta.com/ai/resources/blog/build-with-muse-wonder/ (12 oct 2026) — $4 input / $20 output par 1M, cached $0.20. Wonder Full = subscription Muse Code only (Power/Maximum Usage), pas sur Model API.
- Docs : https://dev.meta.ai/docs/ — 1M contexte, `reasoning.effort: high`, `model: muse-wonder-1.0`, preview. Wonder Full = `max` + abonnement, non accessible via `muse-wonder-1.0` seul.
- Availability : https://dev.meta.ai/docs/models — Wonder `muse-wonder-1.0` sur Model API Standard-only + Muse Code ; Wonder Full Muse Code Power/Maximum only. Non dispo Corée du Sud. Contributor non dispo en pay-as-you-go pour Wonder.

### 2.3 Muse Image 1.0

- Page : https://dev.meta.ai/models/muse-image/ — $0.01/image, agentic (planifie, cherche des références web, génère/exécute du code pour layouts, auto-corrige). Trois primitives : generate / edit / compose (ancrage multi-références).
- Guide : https://dev.meta.ai/docs/image-generation
  - Responses API (conversationnel, multi-turn, `previous_response_id` ou `store:false` + replay `image_generation_call.id`) : `model: muse-image-1.0`, `input` texte + `input_image` (URL ou base64 ou `file_id` via Files API). Sortie `output[]` = `reasoning` (summary) + `message` + `image_generation_call` (base64 ou URL signée). Seul outil accepté : `image_generation` (`enable_image_search` / `enable_web_search` / `enable_shell`, `reasoning_strength: high|low`, `size: WxH` = ratio, `output_format: webp|png|jpeg`). `web_search` ou function tools → 400.
  - One-shot OpenAI-compatible : `POST /v1/images/generations` (text-to-image) et `POST /v1/images/edits` (image(s) → edit/compose, `images[]` Meta-specific en JSON brut, `multipart` via SDK). Params : `n` 1–10, `size`, `response_format: b64_json|url`, `output_format`, `reasoning_strength`, `tool_enablement`, `user` (end-user anti-abus). Streaming SSE `image_generation.completed` / `image_edit.completed`.
  - Facturation : $0.01 par image générée et retournée uniquement (échecs/filtrage non facturés). `usage` tokens informatif, non facturé. Recherche intégrée incluse.

### 2.4 SAM 3.1

- Page : https://dev.meta.ai/models/sam-3-1/ — un modèle pour détection + segmentation + tracking image/vidéo, zero-shot, sans entraînement, mêmes envelope/keys/billing que les autres modèles.
- Guide : https://dev.meta.ai/docs/media-segmentation/ + https://dev.meta.ai/docs/sam/overview + `/segmenting` + `/reading-segmentation` + `/client-libraries`.
  - `model: sam-3.1` sur `POST /v1/responses` (+ `POST /v1/files` pour clips réutilisés). Entrée : prompt noun-phrase courte (« yellow school bus », « delivery van ») ou prompts boîte/point ; image/vidéo. Sortie : boxes + masques pixel-précis, tracking vidéo avec identité.
  - Usage conseillé avec Spark : Spark (VLM) interprète la scène et décide quoi segmenter → SAM produit les masques.
- Pricing : image $2.50/1k images ; vidéo $0.20/1k frames, minimum 50 frames/requête. `usage.video_frames_processed` = frames facturées.

### 2.5 Muse Voice Transcribe 1.0

- Page : https://dev.meta.ai/models/muse-voice-transcribe/ — $0.18/heure ($3.00/1 000 min), streaming + fichier, diarization 20+ speakers, endpointing natif, biasing vocabulaire, 25 langues + code-switch, timestamps au niveau turn (pas au mot). STT seul : pas de TTS, pas de speech-to-speech, pas de détection émotion/événements.
- Endpoints : `wss://api.meta.ai/v1/asr/realtime` + `POST /v1/asr/transcribe`. Guide : https://dev.meta.ai/docs/speech-to-text. Limites : 256 streams concurrents, 16 000 streams/heure. Pas de Contributor, pas de headers `x-ratelimit-*` sur cette surface (1013 / 429 en cas de rejet).

### 2.6 Open weights — Spark Open vs Glimmer

- Annonce Spark Open : https://developer.meta.com/ai/resources/blog/build-with-muse-spark-open/ (15 oct 2026) — 816B MoE, 42B actifs/token, 1M contexte, texte/image/vidéo/audio, licence Muse Model License. BF16 ~1.6 To, NVFP4 ~530 Go. Runtimes SGLang/vLLM/TensorRT-LLM/Transformers/llama.cpp/MLX/Unsloth ; hosted Baseten/Fireworks/Groq/Modal/Nebius/OpenRouter/Together ; catalogs SageMaker/Garden/Foundry. Speculative decoding DFlash2 (SGLang) / DSpark (vLLM). Post-training TorchTitan/Miles/NeMo/Fireworks/Together/TRL.
- Page Glimmer : https://dev.meta.ai/models/muse-glimmer/ — 30B, Apache 2.0, single GPU/Mac, distillé de Spark, tool-calling fiable, état persistant, mémoire auto-gérée. Runtimes vLLM/SGLang/llama.cpp/ExecuTorch, distros HF/Ollama/Unsloth/LMStudio/Fireworks/Together/OpenRouter. Docs : https://dev.meta.ai/docs/muse-glimmer/ (prompting, quantization, spec-decode, vLLM, llama-cpp, ExecuTorch).
- Docs open-weight : https://dev.meta.ai/docs/ — Spark Open et Glimmer ne passent pas par Model API (pas de pricing/limits API), chacun expose son endpoint OpenAI-compatible via son runtime.

### 2.7 Capacités transverses utiles au projet EDA

- Tool calling : https://dev.meta.ai/docs/tool-calling — Responses API recommandée (raisonnement préservé, `previous_response_id`), parallèle + streaming args, tools dev + built-in `web_search` / `tool_search` combinables.
- Structured output : JSON valide selon schéma à chaque fois (pour `document-plan` BOM+contraintes).
- Prompt caching / token counting / file handling / image & video understanding / computer use / multi-agent orchestration / subagent fanout / GitHub agent / search grounding : https://dev.meta.ai/docs/cookbook et pages capacités. À mobiliser pour la preuve Technical Depth.
- Modèles listables : `GET /v1/models` (tri newest-first, champ `created`).

---

## 3. Appels API à instrumenter comme preuve (Stage One + judging)

Principe : chaque étape `plan -> document-plan -> .ato -> ato build -> preview` doit laisser une trace horodatée, rejouable, montrée dans la vidéo. Stocker dans `evidence/` (repo public, clé expurgée) : requête (modèle, effort, tools, schéma), `response.id`, `usage` (input/output/cached_tokens, frames), latence, `x-ratelimit-*`, artefacts générés.

| # | Étape projet | Appel canonique | Preuve à logger |
|---|--------------|-----------------|-----------------|
| P1 | `plan` → brouillon | `POST /v1/responses` `model: muse-spark-1.3`, `reasoning.effort: high`, tools `[]`, `store:true` | `resp.id`, prompt plan, `usage`, temps |
| P2 | `document-plan` validé | même endpoint + `structured output` JSON Schema BOM+contraintes + `tool_search`/`web_search` si grounding datasheet | schéma versionné, JSON validé, `cached_tokens`, citations search |
| P3 | `document-plan` → `.ato` | `POST /v1/responses` avec tools `ato_validate`, `ato_build_dryrun` (function calling parallèle), `previous_response_id: P2.id` | `tool_calls[]`, args, résultats d'exécution locale, `.ato` généré |
| P4 | Preview créative | `POST /v1/images/generations` puis `POST /v1/images/edits` (`model: muse-image-1.0`, `n:1–4`, `size`, `reasoning_strength: high`, `output_format: png`) + variante Responses API multi-turn (`previous_response_id`, `image_generation_call.id`) | prompts, `reasoning.summary`, `b64_json`/`url`, `created`, référence `file_id` |
| P5 | Détourage/mesure | `POST /v1/responses` `model: sam-3.1`, `input: [{input_text: "bare pcb"}, {input_image: file_id}]`, stream | boxes+masques, count, `video_frames_processed` si vidéo, overlay PNG |
| P6 | Voix (option) | `POST /v1/asr/transcribe` ou `wss .../realtime` `model: muse-voice-transcribe-1.0` | transcript + speakers + turns, secondes facturées |
| P7 | Catalogue | `GET /v1/models` | liste modèles activés pour la team |
| P8 | Billing | Dashboard Model API (screenshot) + `GET /v1/...` `usage` agrégés | spend/jour, preuve crédits 150 $ consommés sur la période hackathon |

Snippets minimaux (Python OpenAI SDK, `base_url=https://api.meta.ai/v1`) :

```python
# P1/P2/P3 — Spark
resp = client.responses.create(model="muse-spark-1.3",
    reasoning={"effort": "high"},
    input="Convertis ce plan en document-plan JSON (BOM+contraintes)…")
# P4 — Image one-shot + edit ancré
img = client.images.generate(model="muse-image-1.0", prompt="…", n=1)
edit = client.images.edit(model="muse-image-1.0", prompt="overlay semi-transparent…", image=[open("pcb.png","rb")])
# P4b — Image multi-turn
t2 = client.responses.create(model="muse-image-1.0", previous_response_id=t1.id, input="boîte 3D si demandée")
# P5 — SAM
with client.responses.stream(model="sam-3.1", input=[{"type":"message","role":"user","content":[
  {"type":"input_text","text":"bare pcb"},{"type":"input_image","image_url":"https://…/pcb.png"}]}]) as s:
    for e in s: ...
```

À montrer dans la vidéo : IDs de réponses à l'écran, dashboard avec appels P1–P6, `ato build` qui passe, overlay SAM→Image. C'est ce qui distingue « on a appelé une API » de « usage créatif et vérifiable ».

---

## 4. Idées au-delà du basique (Best Creative / Agentic)

Baseline qui ne suffit pas : un seul appel Spark qui crache du `.ato` + une image générique. Le jury Use-of-Meta-Tooling (25 % `[ticket]`) attend une combinaison.

**A. Pipeline SAM → Image ancré (recommandé, effet visuel fort)**
Capture `ato view` / SVG / KiCad → SAM (`"bare pcb"`, `"silkscreen"`) → masques → Muse Image `edit/compose` : carte détourée + maillage semi-transparent + boîte d'enclosure si demandée. Ancrage multi-images (face avant/arrière + rendu 3D) pour cohérence entre variantes. Répond directement à « capture + image maillée semi-transparente, boîte 3D si demandée » de la map.

**B. Spark multimodal critique visuelle**
Spark 1.3 lit screenshot/schéma (image understanding) ou clip (video understanding), pointe l'erreur (ex. empreinte, DRC), propose patch `.ato`, re-lance `ato build`. Boucle tool-calling avec `previous_response_id` + structured output. Montre le raisonnement long-horizon, pas juste de la génération.

**C. Document-plan grounded**
Spark + `web_search` grounding ($2.50/1k queries) pour vérifier références LCSC/JLCPCB, prix/stock, alternatives. Citations inline dans le document-plan. Prouve l'usage search au-delà du LLM nu.

**D. Voix-dans-la-boucle**
Dicter le `plan` → Voice Transcribe (diarization si 2𝑥, biasing « atopile, BOM, KiCad, gerber ») → Spark structure en `document-plan`. Dans la vidéo : 10 s de dictée → JSON. Coût négligeable, effet démo fort. Ne pas promettre de TTS Meta (inexistant).

**E. Multi-agent (Best Agentic)**
Orchestration : planner Spark → workers (BOM agent, contraintes agent, layout agent) en fanout, puis juge Spark qui fusionne. Pattern cookbook multi-agent-orchestration / subagent-fanout. Failure recovery visible (retry après `ato build` en erreur). Glimmer local en pré-vol offline, Spark cloud en prod. Si temps : tentative computer-use (Wonder/Spark) qui pilote le viewer — stretch, à ne filmer que si ça passe.

**F. Preuve open-weight (Technical Depth)**
Glimmer-30B en local (Ollama/llama.cpp, 1 GPU) qui rejoue les mêmes prompts Spark en mode dégradé + README `deploy/glimmer.md`. Spark Open cité avec config SGLang/vLLM mais marqué « non déployé — hors budget hackathon », avec lien HF. Honnête et vérifiable, mieux qu'un faux self-hosted.

Ordre de priorité si temps manque : A > B > C > F > D > E.

---

## 5. Plan crédits 150 $ `[ticket]` + ordres de grandeur (docs)

Hypothèses : 8 jours (27 oct–4 nov), dev en Contributor, runs filmés en Standard.

- Spark Contributor : $0.10/$0.20 par 1M. 2 000 itérations × (10k in + 2k out) ≈ 20M in + 4M out ≈ $2.00 + $0.80 ≈ **$3**.
- Spark Standard (runs propres + vidéo) : 200 runs × (15k in + 3k out) ≈ 3M + 0.6M ≈ $3.75 + $2.55 ≈ **$6.30** (avec cache, nettement moins : cached $0.15).
- Wonder (2 runs secours) : 2 × (50k in + 10k out) ≈ $0.40 + $0.40 ≈ **$1**.
- Image : 300 images × $0.01 = **$3** (même avec `reasoning high` + search, prix fixe).
- SAM : 500 images × $2.50/1k = **$1.25** ; 20 clips × 200 frames × $0.20/1k = **$0.80**.
- Voice : 5 h × $0.18 = **$0.90**.
- Search grounding : 500 queries × $2.50/1k = **$1.25**.
- **Total estimé ≈ $12–20**, marge >80 % sur 150 $. Le risque n'est pas le dépassement mais la non-consommation visible : viser $30–50 de spend tracé (runs répétés, variantes images, cache désactivé sur 1 run témoin) pour une preuve billing crédible sans gaspillage.

Règles d'usage :
- Réclamer tôt (action #18 : documenter le lien dashboard + code hackathon, 1 team, 1 clé `MODEL_API_KEY` hackathon, rotation après l'événement).
- Dev = `muse-spark-1.3-contributor`, prod/vidéo = `muse-spark-1.3` (Standard, `reasoning: max` autorisé, contenu non réutilisé).
- Logger spend/jour (dashboard + `usage` agrégés), alerter à 50 % / 80 %.
- Respecter rate limits (Standard 3 000 RPM ; Contributor 150 RPM) avec backoff + jitter sur 429.

---

## 6. Conformité soumission (vidéo 3 min / repo public / new-work-only)

> Règles reprises du ticket/map `[ticket]` — à confronter à la page Devpost dès disponible.

- **Vidéo ≤3 min, script imposé :** 0:00–0:30 problème + ce qui est neuf ; 0:30–1:30 live `plan → document-plan (JSON à l'écran) → .ato` avec `response.id` visibles ; 1:30–2:30 `ato build` qui passe + preview SAM→Image (overlay + boîte) + dictée voix (10 s) ; 2:30–3:00 récap Meta tooling (logos modèles + IDs + dashboard billing) + URL repo + mention new-work 27 oct–4 nov. Voix humaine (pas de TTS Meta), sous-titres, pas de musique sous copyright, captions des appels API.
- **Repo public :** licence MIT atopile respectée (pas de fork du cœur sauf fallback #16), README avec quickstart `<5 min`, `ato build` reproductible, `evidence/` (logs expurgés, `response.id`, `usage`, screenshots dashboard), lien vidéo + Devpost, `SECURITY.md` (pas de clé), tag `hackathon-2026`.
- **New-work-only :** tout le code soumis commité entre 27 oct et 4 nov, `git log --since=2026-10-27 --oneline` montrable dans la vidéo ; cette branche `research/meta-tooling` est throwaway (recherche seule, jamais mergée dans `main`) ; socle prod dans #18 sur branche séparée. Aucun artefact pré-existant hors scaffold documenté.

---

## 7. Risques et suivis

- Devpost URL inconnue → action #18 : épingler l'URL officielle, vérifier critères/poids exacts, Best Creative/Agentic tracks, règle new-work.
- Crédits : processus de réclamation non décrit dans `dev.meta.ai/docs` (Help Center / dashboard) → action #18 : capturer l'écran de réclamation + solde initial.
- SAM vidéo : minimum 50 frames facturées même pour clip court → préférer images fixes sauf besoin tracking réel.
- Image : `moderation` param retourne 400 sur `muse-image-1.0` (non activé) — omettre le champ.
- Spark 1.3 audio-in dégradé → ne pas envoyer d'audio à Spark, passer par Voice Transcribe puis texte.

## Sources

- https://dev.meta.ai/docs/ (overview, endpoints, cookbook)
- https://dev.meta.ai/docs/models (versions, availability, Wonder-vs-Spark)
- https://dev.meta.ai/docs/pricing-rate-limits (prix, Contributor/Standard, rate limits)
- https://dev.meta.ai/models/muse-spark/ ; https://dev.meta.ai/models/muse-image/ ; https://dev.meta.ai/models/muse-glimmer/ ; https://dev.meta.ai/models/sam-3-1/ ; https://dev.meta.ai/models/muse-voice-transcribe/
- https://developer.meta.com/ai/resources/blog/build-with-muse-spark-open/ ; https://developer.meta.com/ai/resources/blog/build-with-muse-wonder/
- https://dev.meta.ai/docs/image-generation ; https://dev.meta.ai/docs/media-segmentation/ ; https://dev.meta.ai/docs/speech-to-text ; https://dev.meta.ai/docs/tool-calling
