# Eigen versie: afhankelijkheden van de ArtCraft-servers

Status: fase 1 (inventaris). Er is nog niets omgebouwd. Peildatum 2026-10-17, upstream `storytold/artcraft` main.

Doel van de eigen versie: geen login, eigen fal.ai-sleutel, alles lokaal. Wijzigingen komen in
aparte modules (bijvoorbeeld `crates/desktop/artcraft/src/local/` en `frontend/libs/local-*`),
zodat upstream-updates blijven mergen.

Paden zijn relatief aan de repo-root. `D/` staat voor `crates/desktop/artcraft/src/`.

## Kern in één alinea

Alles draait om de **`media_token`** van hun server. Elke invoerafbeelding gaat eerst naar
`api.storyteller.ai`. Elk resultaat komt terug als een URL op hun CDN (`cdn-2.fakeyou.com`). De
galerij is de medialijst van je account op hun server. Omni (`/v1/omni_gen/*`) is de standaardroute
voor elke modaliteit. Zonder login kom je niet voorbij een loginvenster dat je niet kunt sluiten. Het
bestaande fal-pad werkt alleen voor **beeld**, en ook dat heeft nog hun server nodig: voor de invoer
(token → CDN-URL) en bij voltooiing (`handle_fal_complete` eist login en uploadt het resultaat naar
hun server).

## Overzicht per functie

Legenda omvang: **S** is minder dan een halve dag, **M** is 1 tot 2 dagen, **L** is 3 tot 5 dagen
(voor Claude Code, inclusief testen).

| Functie                 | Werkt nu                                               | Belangrijkste bestanden                                                                                                   | Voorgestelde vervanging                                                                                     | Omvang |
|-------------------------|--------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|--------|
| Login / sessie          | Alleen via hun server, blokkeert de app                | `frontend/libs/components/login-modal/src/lib/login-modal.tsx`, `D/services/storyteller/commands/*`, `MainApp.tsx`        | Login-modal en sessiecheck uitzetten met een vlag "lokale modus" en een vaste lokale gebruiker              | S      |
| Credits / abonnement    | Alleen via hun server (alleen weergave)                | `libs/state/credits`, `libs/state/subscription`, `TopBar.tsx`, `D/.../storyteller_get_credits_command.rs`                 | Verbergen in lokale modus. Kosten tonen via de bestaande `cost.rs` per fal-provider in `artcraft_router`    | S      |
| Model-catalogus         | Hun server, met hardcoded lijst als terugval           | `D/core/commands/generate/models/*/list_*_models_command.rs`, `libs/model-list/src/lib/lists/*`, `omni-gen-hooks.ts`      | Statische lijst met alleen fal-modellen die polling ondersteunen. Audio-lijst toevoegen                     | S      |
| Beeld                   | Standaard via hun server. Fal-pad half af              | `D/.../generate_image/generate_image_command.rs`, `.../artcraft_router/handle_api_providers.rs`, `handle_fal_complete.rs` | Invoer als data-URI of fal-storage-upload, voltooiing lokaal opslaan, `create_prompt` vervangen door SQLite | M      |
| Video                   | Alleen via hun server (fal niet aangesloten)           | `D/.../generate_video/generate_video_command.rs`, `.../providers/fal/` in `artcraft_router` (22 modules)                  | Fal-router-providers aansluiten zoals bij beeld                                                             | M      |
| Audio                   | Alleen via hun server                                  | `D/core/commands/generate/generate_audio_command.rs`, `PageAudio/CreateAudio.tsx`                                         | Fal `seed_audio_1p0` uit de router aansluiten. Andere audio-modellen zijn er niet                           | S–M    |
| 3D-mesh                 | Alleen via hun server                                  | `D/.../generate_mesh_command.rs`, `ImageTo3DExperience.tsx`, `ImageTo3DStore.ts`                                          | Fal-mesh-providers (hunyuan, meshy, rodin, tripo3d) aansluiten. GLB lokaal opslaan                          | M      |
| Worlds / splats         | Hun server, of WorldLabs (ook via hun server)          | `D/.../generate_splat_command.rs`, `enqueue_image_to_gaussian_command.rs`, `services/worldlabs/*`                         | Fal `triposplat` aansluiten. WorldLabs-pad later los van hun server maken                                   | M      |
| Achtergrond verwijderen | Alleen via hun server                                  | `D/.../image_bg_removal/artcraft/handle_generic_bg_removal_artcraft.rs`, `RemoveBackground.tsx`, `PageDraw.tsx`           | Nieuwe fal-provider (bijvoorbeeld `fal-ai/birefnet`). Invoer is al base64, dus eenvoudig                    | S      |
| 2D-canvas               | Tekenen lokaal. Genereren via hun server               | `libs/components/pagedraw/src/lib/PageDraw.tsx` (`uploadSceneSnapshot`), `BaseImageSelector.tsx`                          | Canvas-snapshot lokaal opslaan en als data-URI meesturen                                                    | S      |
| 3D-canvas (Three.js)    | Bewerken lokaal. Assets, opslaan, laden via hun server | `libs/components/pagescene/src/lib/adapter.ts`, `PageScene/useTauriPageSceneAdapter.tsx`, `sceneOutputAdapter.ts`         | Lokale implementatie van `PageSceneAdapter`. Demo-GLB's eenmalig binnenhalen en meeleveren                  | L      |
| Prompts / geschiedenis  | Takenlijst lokaal (SQLite). Prompttekst op hun server  | `_database/sql/artcraft_migrations/`, `D/.../get_task_queue_command.rs`, `generation-list/src/lib/prompts-cache.ts`       | Kolommen of tabel `prompts` in de bestaande SQLite-takendatabase                                            | M      |
| Media-bibliotheek       | Alleen via hun server                                  | `gallery-modal.tsx`, `useGalleryData.ts`, `MediaFilesApi.ts`, `D/.../media_file_delete_command.rs`                        | Tabel `media_files` in SQLite plus bestanden in `~/Artcraft/media/`, aangeboden via Tauri asset-protocol    | L      |
| Fal-sleutel             | Werkt, maar als platte tekst                           | `D/.../provider_set_api_key_command.rs`, `provider_credential_key.rs` → `~/Artcraft/credentials/fal.api_key.txt`          | Opslaan in de macOS-sleutelhanger (crate `keyring`) en een veld in de instellingen                          | S      |

Totaal grofweg 4 tot 6 weken als alles om moet. Een eerste bruikbare versie (geen login, beeld,
video en achtergrond verwijderen via fal, lokale galerij) is ongeveer 1,5 week.

## Gedeelde bouwstenen (hier zit het meeste werk)

### 1. De `media_token` als centrale sleutel

- **Rust:**
  - `D/core/commands/generate/generate_image/providers/artcraft_router/utils/map_media_files_to_urls.rs` heeft `map_media_file_tokens_to_cdn_urls`. Die roept voor elke token `/v1/media_files/file/{token}` aan.
- **Frontend:**
  - Uploads gaan via `UploadImageMedia` / `MediaUploadApi` naar `/v1/media_files/upload/image`.
  - De CDN-origin staat hardcoded in `frontend/apps/artcraft/app/src/api/GetCdnOrigin.ts:18` en `frontend/libs/api/src/lib/GetCdnOrigin.ts:3`.
- **Voltooiingsevents:** `D/.../events/notify_frontend_of_completion.rs` stuurt `media_token` plus `cdn_url` mee, met als terugval `cdn.artcraft.ai/placeholder`.

**Vervanging:** een lokale mediadienst die dezelfde vorm teruggeeft.

- **Token:** een eigen token, bijvoorbeeld `local_…`. De vorm blijft gelijk, zodat de rest van de frontend niet hoeft te veranderen.
- **URL:** een `asset://`- of `convertFileSrc`-URL naar een bestand in `~/Artcraft/media/`.
- **Index:** een tabel in de bestaande SQLite-database.
- **Ingrijpen op twee punten:**
  1. In de frontend: `StorytellerApiHostStore` / `GetCdnOrigin` en de media-API's laten omleiden naar Tauri-commands.
  2. In Rust: `map_media_file_tokens_to_cdn_urls` laten verwijzen naar lokale bestanden.
- **Omvang:** L. Dit is de basis voor beeld, galerij, prompts en 3D.

### 2. Invoer naar fal

- `fal_client` kan nog niet uploaden naar fal-storage.
- `crates/lib/files` heeft al `file_to_base64_url`.
- **Eenvoudigst:** data-URI's meesturen. Fal accepteert die voor de meeste modellen.
- **Voor grote bestanden (video):** fal-storage-upload toevoegen aan `fal_client`. Omvang: S.
- Let op: het fal-beeldpad negeert nu de ruwe canvasbytes (`canvas_image_raw_bytes`) en gebruikt alleen tokens.

### 3. Voltooiing van fal-taken

In `D/core/threads/third_party_task_polling_thread/handlers/fal/handle_fal_complete.rs`:

| Regel | Wat er nu gebeurt                                                             | Wijzigen?                    |
|-------|-------------------------------------------------------------------------------|------------------------------|
| 79    | `.ok_or("No Storyteller credentials available")`: zonder login stopt het hier | Weg                          |
| 95    | `auto_download_task_urls`: slaat het resultaat lokaal op                      | Blijft                       |
| 124   | `upload_to_backend`: uploadt het resultaat naar hun server                    | Vervangen door lokale index  |
| 146   | `get_media_file`: haalt de CDN-URL op                                         | Vervangen door lokale index  |
| 181   | `notify_frontend_of_completion(storyteller_host…)`                            | Lokale URL en token meegeven |

Gevolg nu: **een fal-taak wordt in de frontend nooit afgerond zonder ArtCraft-login.** Omvang: S tot M.

### 4. Routering

- `D/core/commands/generate/omni/request.rs:29` `uses_artcraft()` geeft `true` bij provider `None`. Daardoor is Omni (hun server) overal de standaard.
- `handle_router.rs:50` kent alleen Fal als sleutelprovider.

**Vervanging:**

- In lokale modus de standaardprovider op Fal zetten (instelling `set_provider_order_command` bestaat al).
- Per modaliteit een fal-tak toevoegen.

**Waarschuwing:** deze fal-modellen eisen een webhook (`WebhookUrlRequired`) en zijn dus zonder eigen
server niet bruikbaar. Kies voor elk ervan een andere fal-variant of laat het weg.

- **Beeld:** flux_pro_1p1, flux_pro_1p1_ultra, nano_banana, seedream_4, seedream_4p5, seedream_5_lite
- **Video:** kling_2_1_master, kling_2_1_pro, kling_3p0_pro, seedance_1p0_lite, seedance_1p5_pro, sora_2, sora_2_pro

### 5. Opslag van gegevens en geheimen nu

- **App-map:** `~/Artcraft/` (`D/core/state/data_dir/app_data_root.rs`), met de submappen `assets`, `credentials`, `downloads`, `settings`, `state` en `temp`.
- **Takendatabase:** `~/Artcraft/state/tasks_v{N}.sqlite` (migraties in `_database/sql/artcraft_migrations/`).
- **Geheimen:** platte tekst in `~/Artcraft/credentials/`. Dat geldt voor de fal-sleutel, sessiecookies en de Sora-, Grok- en Midjourney-gegevens. Er is geen keyring-crate.
- **Polling:** het pollen van fal gaat al rechtstreeks naar `queue.fal.run`.

## Per functie in detail

### Login / sessie

- **Frontend:**
  - Bij het laden roept `login-modal.tsx:61-86` `storyteller_get_login_session_command` aan. Is er geen gebruiker, dan opent een schermvullende overlay zonder sluitknop.
  - `MainApp.tsx:153-160` roept `UsersApi.GetSession()` aan.
- **Rust:**
  - `D/services/storyteller/commands/*` heeft de login-challenge (`app.getartcraft.com/login/desktop`), wachtwoord-login en aanmelden.
  - `persist_storyteller_cookies_task.rs` kopieert de cookies.
  - `storyteller_activity_thread.rs` stuurt analytics naar `/v1/analytics/active_user_v2`.
- **Vervanging:** een build-vlag of instelling `local_mode`.
  - De modal en de sessiechecks overslaan.
  - Een vaste lokale gebruiker teruggeven.
  - De activity-thread niet starten.
  - Upstream-code blijft staan, alleen de aanroep wordt omgeleid.

### Credits, abonnement en kosten

- **Credits en abonnement:**
  - `credits-state.ts`, `subscription-state.ts` en `TopBar.tsx` pollen de server.
  - Stripe-aankopen gaan via `/v1/stripe_artcraft/*`.
- **Kostenschattingen:** `estimate_*_cost_command` → `/v1/generate/cost_estimate/*` of `/v1/omni_gen/cost/*`.
- **Vervanging:**
  - Alles rond credits verbergen.
  - Voor kosten de `cost.rs` per fal-provider in `crates/api_clients/artcraft/artcraft_router` gebruiken. Die is al lokaal.

### Beeld

- **Standaard:** Omni (`/v1/omni_gen/generate/image`).
- **Legacy-modellen:** recraft_3, flux_pro_kontext_max, flux_dev_juggernaut en flux_pro_1 gaan naar de oude endpoints `/v1/generate/image/*`.
- **Fal-pad** (`handle_api_providers.rs`):
  - regel 59: gebruikt `storyteller_host`
  - regel 67 en 147: tokens → CDN-URL's
  - regel 87 en 153-178: `create_prompt_record` → `/v1/prompts/create` (gaat door als dit faalt)
  - regel 93: indienen bij fal gaat wel rechtstreeks
- **Frontend:**
  - `PromptBoxImage.tsx:433` uploadt referentiebeelden eerst.
  - `PromptBox3D.tsx:424` roept `OmniGenApi.generateImage` rechtstreeks via HTTP aan.

### Video

- **Route:** `generate_video_command.rs` gaat naar Omni, of naar `handle_artcraft_video_via_router.rs` met `RouterClient::Artcraft`.
- **Niet aangesloten:** de 22 fal-videoproviders in de router.
- **Native Sora en Grok:** gebruiken voor invoer en resultaat ook hun server (`services/sora/*`, `services/grok/*`).
- **Characters:** `CharactersApi` (`/v1/characters/*`) is alleen server. Voorlopig uitzetten.

### Audio

- Alleen Omni.
- In de router zit `seed_audio_1p0` (fal, polling).
- De audiomodellenlijst komt alleen van de server.

### 3D-mesh en worlds/splats

- **Commands:** `generate_mesh_command.rs` en `generate_splat_command.rs` gaan naar Omni.
- **Fal in de router (niet aangesloten):** 12 mesh-providers en `triposplat`.
- **WorldLabs:** het pad haalt de invoer op via `get_media_file` en uploadt het resultaat naar hun server.
- **Frontend:** `ImageTo3DExperience.tsx` uploadt eerst. `useSessionMediaByClass.ts` toont `/v1/media_files/{mesh,splat}/list`.

### Achtergrond verwijderen

- `handle_generic_bg_removal_artcraft.rs` → `/v1/generate/image/remove_background`, met upload van base64.
- Er is nergens een fal-pad.
- Het frontend-command accepteert al `base64_image`, dus dit is de makkelijkste om om te bouwen.

### 2D-canvas

- Tekenen is lokaal.
- Bij genereren wordt de snapshot geüpload via `/v1/image_studio/scene_snapshot` (`PageDraw.tsx:897,931`). Daarna volgt `GenerateImage` met `canvasImageMediaToken`.

### 3D-canvas (Three.js scene-editor)

- **Wat al lokaal is:** bewerken, skyboxes (`/resources/skybox/*`) en primitieve vormen.
- **Nette scheidingslaag:** `PageSceneAdapter` (`libs/components/pagescene/src/lib/adapter.ts:111-282`), geïmplementeerd in `useTauriPageSceneAdapter.tsx`:
  - `saveScene`: uploadt naar `/v1/media_files/upload/new_scene`
  - `loadScene`: CDN
  - `listUserMediaFiles` / `listFeaturedMediaFiles`
  - `getMediaUrlByToken`
- **Demo-personages en props:** `demoAssets/*Items.ts` hebben lokale thumbnails, maar de GLB's zelf zijn server-`media_id`'s.
- **Vervanging:**
  - Een tweede adapter, `useLocalPageSceneAdapter`.
  - Scènes als JSON in `~/Artcraft/scenes/`.
  - De gebruikte demo-GLB's eenmalig downloaden en in `frontend/apps/artcraft/public/resources/` zetten (alleen voor eigen gebruik).

### Prompts / geschiedenis

- **Lokaal:** de takenlijst (`get_task_queue_command`, SQLite).
- **Op hun server:**
  - Prompttekst: `/v1/prompts/{token}` en `/v1/prompt/batch` via `prompts-cache.ts`, `lightbox-modal.tsx` en `desktopMediaActions.ts`.
  - Media-URL's bij taken: `GetMediaFileByToken` en `GetMediaFilesByBatchToken` in `useDesktopGenerationFeed.ts`.
- **Vervanging:** een migratie die prompt, model en parameters bij de taak opslaat, plus een lokaal command `get_prompt`.

### Media-bibliotheek

- **Op hun server:**
  - Galerij: `GalleryModalApi.ListUserMediaFiles` → `/v1/media_files/list/user/{username}`.
  - Mappen en tags: `/v1/folders/*` en `/v1/tags/*`.
  - Verwijderen: `media_file_delete_command`.
  - Deellinks: `getartcraft.com/media/{token}`.
- **Al lokaal:**
  - Downloads (`download_url_command`).
  - Moodboards (localStorage).
  - Videobewerkerprojecten (IndexedDB).
- **Vervanging:** de lokale mediadienst uit bouwsteen 1 met list, delete en tags. Mappen en tags kunnen later.

## Externe hosts

- **Hun diensten (vervangen of uitzetten):**
  - `api.storyteller.ai`, `upload.storyteller.ai`, `cdn.storyteller.ai`, `cdn-2.fakeyou.com` en `frontend-cdn.fakeyou.com`
  - `api.fakeyou.com`, `studio.storyteller.ai`, `style.storyteller.ai` en `hax.storyteller.ai`
  - `app.getartcraft.com`, `getartcraft.com`, `cdn.artcraft.ai` en `storage.googleapis.com/vocodes-public`
- **Tracking (uitzetten in de eigen versie):**
  - `www.googletagmanager.com` en `www.clarity.ms` (`frontend/apps/artcraft/index.html`)
  - `us.i.posthog.com` (`index.tsx:41`)
  - Discord-presence in `D/core/threads/discord_presence_thread.rs`
- **Overige derden (onschuldig):** Google Fonts, `api.frankfurter.dev` (valuta), `www.gravatar.com` en `cdn.brandfetch.io`.
- **Tauri-allowlist:** `crates/desktop/artcraft/tauri.conf.json` regels 27-90. `queue.fal.run` en `fal.media` moeten erbij als de frontend fal-URL's rechtstreeks laadt.

## Dode code die van pas komt

- Frontend-wrappers in `frontend/libs/tauri-api/src/lib/fal/*` roepen commands aan die niet geregistreerd zijn in `D/lib.rs`: `SetFalApiKey`, `GetFalApiKey`, `FalBackgroundRemoval`, `FalHunyuanImageTo3d` en `FalKlingImageToVideo`. Die zijn een goed vertrekpunt.
- Ook niet geregistreerd: `enqueue_image_to_3d_world_command` en `enqueue_image_to_video_command`.

## Voorgestelde volgorde voor fase 2

1. **Lokale modus en login weg.** Login, credits en analytics uit. Het fal-pad voor beeld afmaken: invoer als data-URI en `handle_fal_complete` zonder login. Daarna werkt beeldgeneratie met een eigen sleutel. Ongeveer 2 dagen.
2. **Lokale mediadienst en galerij.** Tabel in SQLite en bestanden in `~/Artcraft/media/`. Ongeveer 3 tot 4 dagen.
3. **Achtergrond verwijderen en video via fal.** Ongeveer 2 dagen.
4. **Sleutel naar de macOS-sleutelhanger en een instellingenveld.** Ongeveer een halve dag.
5. **Audio, mesh, splats en de 3D-scene-adapter.** Ongeveer 1 tot 2 weken.
