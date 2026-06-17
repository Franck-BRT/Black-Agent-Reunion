# Plan de fork — Meetily adapté (transcription réunions 100% local)

> Document de kickoff pour Claude Code en **Plan Mode**.
> Base forkée : `Zackriya-Solutions/meetily` (Tauri + Rust + Next.js, MIT).
> Ne RIEN modifier avant d'avoir terminé la Phase 0 (reconnaissance).

---

## 1. Objectif

Adapter Meetily Community pour :
- Enregistrer l'audio des réunions (micro + audio système : Teams, navigateur, etc.).
- Transcrire en local (Whisper/Parakeet, déjà présent).
- **Ajouter la diarisation** (qui parle) — absente de la version Community.
- Brancher un **endpoint OpenAI-compatible** pour résumés/prompts (config, pas de dev).
- Produire un **build portable Windows** (sans droits admin) + un `.dmg` macOS standard.

## 2. Cibles & contraintes

| Cible | Packaging | Capture audio | Contrainte clé |
|-------|-----------|---------------|----------------|
| Windows | **Portable, sans install, sans admin** | WASAPI loopback (aucune permission) | C'est LA cible prioritaire pour le portable |
| macOS | `.dmg` standard (déjà produit upstream) | ScreenCaptureKit (permission TCC utilisateur) | Pas de portable demandé, pas de signature à gérer |

Résumés : endpoint OpenAI-compatible, base URL se terminant par `/v1` (ex. `mlx-openai-server`, LM Studio, llama.cpp server). **Pas de développement, c'est de la configuration.**

## 3. Règles de travail (garde-fous)

- Travailler sur une branche dédiée par chantier (`feat/openai-endpoint`, `feat/win-portable`, `feat/diarization`). Jamais sur `main`.
- Le repo utilise des **submodules** (whisper.cpp) : après clone, `git submodule update --init --recursive`. Ne jamais committer dans les submodules.
- Garder `upstream` configuré pour pouvoir resynchroniser plus tard.
- Ne pas casser le build existant : tout changement doit laisser `macOS .dmg` et le build de base fonctionnels.
- Préférer la config (`tauri.conf.json`, settings UI) au code quand c'est possible.

---

## PHASE 0 — Reconnaissance (à faire AVANT toute modif)

But : comprendre l'architecture réelle, pas supposer. Produire une note `RECON.md` avec les réponses.

**Commandes d'investigation à lancer :**
```bash
# Vue d'ensemble
fd -t f -e toml -e json | grep -iE 'tauri|cargo'
cat tauri.conf.json 2>/dev/null || fd tauri.conf.json

# Diarisation : qu'est-ce qui existe déjà ?
grep -ri 'sortformer' . --include='*.rs' --include='*.toml' --include='*.json'
grep -ri 'diariz'    . --include='*.rs' --include='*.ts' --include='*.tsx'
grep -ri 'onnx'      . --include='*.rs' --include='*.toml'
grep -ri 'speaker'   . --include='*.rs' --include='*.ts'

# Endpoint OpenAI : où est géré le custom endpoint ?
grep -ri 'openai'    . --include='*.rs' --include='*.ts'
grep -ri 'base_url\|baseUrl\|/v1' . --include='*.rs' --include='*.ts'

# Chemins de données (critique pour le portable Windows)
grep -ri 'app_data_dir\|data_dir\|AppData\|local_data' . --include='*.rs'

# Capture audio
grep -ri 'wasapi\|loopback\|screencapture\|cpal' . --include='*.rs'
```

**Questions auxquelles `RECON.md` doit répondre :**
1. Version de Tauri (1.x ou 2.x) ? Cible `tauri.conf.json` exacte.
2. Le custom OpenAI endpoint : où est-il lu/stocké ? Quelle forme d'URL attend-il (racine vs `/v1`) ?
3. Sortformer/ONNX : y a-t-il du câblage de diarisation déjà présent mais non exposé ? Ou rien ?
4. Où l'app écrit-elle ses données (modèles, SQLite, enregistrements) ? Chemins absolus codés en dur ?
5. Comment la capture audio système est-elle faite sur Windows aujourd'hui ?

---

## CHANTIER A — Endpoint OpenAI-compatible (le plus simple, valider en premier)

**Pourquoi en premier :** zéro/peu de code, valide immédiatement le pipeline résumé, donne une victoire rapide.

**Étapes :**
1. À partir de la RECON, localiser le champ "Custom OpenAI Endpoint" dans les settings.
2. Vérifier le comportement `/v1` : tester `http://localhost:PORT/v1` et `http://localhost:PORT`. Documenter laquelle fonctionne.
3. Si l'URL est mal normalisée (double `/v1` ou 404), corriger la construction d'URL côté code pour accepter les deux formes proprement.
4. Tester un résumé de bout en bout sur un transcript existant avec l'endpoint local.

**Done quand :** un résumé est généré via l'endpoint local `/v1`, sans cloud, et la forme d'URL attendue est documentée.

---

## CHANTIER B — Build portable Windows (sans admin)

**Étapes :**
1. **Format de bundle.** Dans `tauri.conf.json`, viser `nsis` (per-user, sans élévation) plutôt que `msi`. Idéalement, récupérer aussi le binaire + ressources brut depuis le dossier de build pour un vrai portable.
2. **WebView2.** Configurer `webviewInstallMode` en **fixed version** (runtime embarqué, ~150 Mo) pour garantir le fonctionnement sur un poste sans WebView2 et sans droits d'install. Télécharger le runtime fixed et le référencer.
3. **Chemins de données.** Vérifier (RECON Q4) que modèles, SQLite et enregistrements s'écrivent dans `%LOCALAPPDATA%` ou à côté de l'exe — JAMAIS dans `Program Files`. Corriger tout chemin en dur.
4. **Test sur machine sans admin.** Copier le dossier portable, lancer, faire une capture WASAPI loopback d'un onglet navigateur, vérifier qu'aucune élévation n'est demandée.

**Done quand :** le dossier portable tourne sur un Windows sans droits admin, capture l'audio système, et n'écrit nulle part qui exige une élévation.

---

## CHANTIER C — Diarisation (reconnaissance des personnes)

> Selon la RECON Q3, choisir UNE des deux voies. Commencer par valider la qualité avant d'intégrer proprement.

**Voie 1 — Post-traitement (le plus simple, recommandé pour démarrer) :**
1. Récupérer le WAV que Meetily enregistre déjà.
2. Passe de diarisation externe : `pyannote.audio` ou **WhisperX** (timestamps mot-à-mot + diarisation).
3. Fusionner les segments locuteurs avec le transcript existant → "Speaker 1 / Speaker 2".
4. Exposer le résultat dans l'UI.

**Voie 2 — Natif via Sortformer/ONNX (si câblage déjà présent) :**
1. Exploiter l'infra ONNX repérée en RECON pour faire la diarisation côté Rust, sans dépendance Python.
2. Plus propre et cohérent avec le portable Windows (pas de runtime Python à embarquer).

**Limite à acter :** la diarisation donne des labels anonymes ("Speaker 1"), pas les vrais noms. Mapping nom→locuteur = **phase 2** (mapping manuel post-réunion, ou voice fingerprinting plus tard).

**Done quand :** un transcript de réunion affiche des locuteurs distincts attribués correctement aux bons segments.

---

## 4. Ordre d'exécution recommandé

1. **Phase 0** (RECON.md) — obligatoire.
2. **Chantier A** (endpoint) — victoire rapide, valide le pipeline résumé.
3. **Chantier B** (portable Windows) — la contrainte structurante de ta cible.
4. **Chantier C** (diarisation) — le plus de R&D, à faire une fois la base stable.
5. Phase 2 (plus tard) : noms réels des locuteurs, exports, raffinements.

## 5. Instruction de démarrage pour Claude Code

> Commence par la Phase 0 uniquement. Lance les commandes d'investigation, remplis `RECON.md`, et **présente-moi tes conclusions + le plan détaillé du Chantier A avant d'écrire la moindre ligne de code.** N'enchaîne pas les chantiers sans validation.
