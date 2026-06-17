# RECON — Reconnaissance Phase 0 (fork Meetily)

> Note prise **avant toute modification de code**, conformément à `PLAN_FORK_MEETILY.md`.
> Base : `Zackriya-Solutions/meetily` — Tauri + Rust + Next.js. Branche : `feat/openai-endpoint`.

---

## Q1 — Version de Tauri & cible `tauri.conf.json`

- **Tauri 2.x** (confirmé par les clés `core:*` dans `permissions`, le bloc `capabilities`
  et `app.security.capabilities`).
- Config unique : **`frontend/src-tauri/tauri.conf.json`** (aucune autre instance hors `node_modules`).
- `productName: meetily`, `version: 0.4.0`, `identifier: com.meetily.ai`.
- `bundle.targets` : `deb, appimage, msi, nsis, app, dmg` → **`nsis` déjà présent** (per-user, utile au portable Windows / Chantier B).
- `bundle.windows.webviewInstallMode` : **non configuré** → défaut `downloadBootstrapper`
  (à passer en `fixedRuntime` au Chantier B).
- CSP `connect-src` : `'self' http://localhost:11434 http://localhost:5167 http://localhost:8178 https://api.ollama.ai`.

---

## Q2 — Custom OpenAI endpoint : stockage, lecture, forme d'URL  ⭐

- **Provider** : `LLMProvider::CustomOpenAI` (enum dans `summary/llm_client.rs:75`), clé string `"custom-openai"`.
- **Stockage** : ce n'est **pas** une colonne dédiée mais un **JSON `customOpenAIConfig`** dans la
  table SQLite `settings` (ligne `id='1'`).
  - Struct `CustomOpenAIConfig` (`summary/mod.rs:15`) : `endpoint`, `apiKey?`, `model`,
    `maxTokens?`, `temperature?`, `topP?`.
  - Save/Get : `SettingsRepository::save_custom_openai_config` / `get_custom_openai_config`
    (`database/repositories/setting.rs:322` / `:277`).
- **Lecture au moment du résumé** : `summary/service.rs:358` charge le config et passe `endpoint`
  à `generate_summary(...)`.
- **Construction d'URL** (`summary/llm_client.rs:173`, branche `CustomOpenAI`) :
  ```rust
  format!("{}/chat/completions", endpoint.trim_end_matches('/'))
  ```
  → le code **ajoute uniquement `/chat/completions`** ; il ne normalise pas `/v1`.
- **Forme d'URL attendue** : l'UI (placeholder `http://localhost:8000/v1`,
  `ModelSettingsModal.tsx:956`, libellé « Base URL of the OpenAI-compatible API ») demande à
  l'utilisateur d'inclure `/v1`. Comportement effectif :

  | Saisie utilisateur | URL finale envoyée | Résultat |
  |---|---|---|
  | `http://host:PORT/v1` | `http://host:PORT/v1/chat/completions` | ✅ OK |
  | `http://host:PORT/v1/` | `http://host:PORT/v1/chat/completions` | ✅ OK (trailing `/` géré) |
  | `http://host:PORT` (sans `/v1`) | `http://host:PORT/chat/completions` | ❌ 404 probable |
  | `http://host:PORT/v1/chat/completions` (URL complète) | `.../chat/completions/chat/completions` | ❌ double segment |

- **CSP non bloquante** : les requêtes de résumé partent du **backend Rust via `reqwest`**, pas du
  webview. La CSP `connect-src` ne s'applique donc pas à l'endpoint local custom, même sur un port
  non listé. **Aucune modif CSP requise pour le Chantier A.**
- La liste des modèles (`openai/openai.rs::get_openai_models`) est **codée en dur sur
  `api.openai.com`** ; pour `custom-openai`, le modèle est **saisi manuellement** dans l'UI
  (`ModelSettingsModal.tsx:233`). Pas d'impact sur le Chantier A.

**Conclusion Q2** : le câblage existe et fonctionne *si et seulement si* l'utilisateur saisit
exactement une base finissant par `/v1`. Le builder d'URL est **fragile** → cible du durcissement
(Chantier A).

---

## Q3 — Diarisation / Sortformer / ONNX

- **Aucune diarisation câblée** : 0 occurrence de `sortformer`, 0 de `diariz`. Les occurrences de
  `speaker` concernent les **enceintes / output devices**, pas les locuteurs.
- **Le runtime ONNX existe déjà** : `onnx` présent dans `frontend/src-tauri/Cargo.toml` et dans
  tout `parakeet_engine/` (ORT, pour la STT Parakeet).
- **Conséquence Chantier C** : la **Voie 2 (diarisation native ONNX côté Rust)** est techniquement
  envisageable (runtime déjà présent, pas de Python à embarquer → cohérent avec le portable Windows),
  mais **rien n'est branché ni exposé** aujourd'hui. Choix de la voie à trancher au Chantier C.

---

## Q4 — Chemins de données (critique pour le portable Windows)

- L'app passe par l'**API path de Tauri** (`app.handle().path()`, `resource_dir()`, app_data_dir) :
  - modèles Whisper & Parakeet : `set_models_directory(app_handle)` (`lib.rs:454-465`) ;
  - base SQLite : `database::setup::initialize_database_on_startup` (`lib.rs:497`) ;
  - templates : `resource_dir()` (`lib.rs:504`).
- **Aucun chemin absolu codé en dur** côté runtime. Les seuls `C:\meetings\...` trouvés
  (`analytics/analytics.rs:473-477`) sont des **fixtures de test**, pas des chemins de production.
- **À vérifier au Chantier B** : que app_data_dir sur Windows résout vers
  `%APPDATA%` / `%LOCALAPPDATA%` (jamais `Program Files`). A priori conforme (Tauri s'appuie sur `dirs`).

---

## Q5 — Capture audio système sous Windows

- **WASAPI loopback via `cpal`** — aucune permission OS requise (contrairement à macOS
  ScreenCaptureKit/TCC).
- Détection des devices : `audio/device_detection.rs` (patterns WASAPI ; `loopback` ligne 175).
- Implémentation de référence (ancienne) : `audio/core-old.rs:122`
  (`cpal::host_from_id(cpal::HostId::Wasapi)`, énumération des outputs incluant le loopback).
- Capture courante : `audio/capture/system.rs` et `audio/system_audio_stream.rs`.

---

## Synthèse pour la suite

| Chantier | État de départ | Action principale |
|---|---|---|
| **A — Endpoint OpenAI** | Câblé mais builder d'URL fragile (pas de normalisation `/v1`) | Normaliser l'URL (tolérer les 4 formes) + test + hint UI |
| **B — Portable Windows** | `nsis` présent, `webviewInstallMode` non réglé, chemins via API Tauri | `webviewInstallMode: fixedRuntime`, vérifier app_data_dir hors `Program Files` |
| **C — Diarisation** | Rien de câblé, mais runtime ONNX déjà présent | Choisir Voie 1 (post-traitement) vs Voie 2 (ONNX natif) |
