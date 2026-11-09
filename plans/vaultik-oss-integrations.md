# Plan: Making Onyx great on the Huawei MatePad (open-source repos that help most)

## Issues to Address

The Vaultik team uses Onyx from a **Huawei MatePad**, in meetings, on the road and away from a
laptop. The work is mostly GTM and dealmaking: account and market research, intel reports,
discovery and closing calls, outreach and follow-ups, data rooms and decks.

The goal is to pick the most widely used open-source projects that make **Onyx useful on the
tablet itself**, and plan how to bring them in, in order of value to effort. The tablet should
handle:

1. **Get to Onyx securely** from anywhere, as an app-like experience rather than a browser tab.
2. **Capture by voice** (dictating questions and debriefs right after a call) without Google speech
   services.
3. **Capture with the camera and pen**: business cards, whiteboards, handwritten notes and M-Pencil
   sketches into Onyx.
4. **Share into Onyx** from any tablet app (articles, PDFs, emails) using the system share sheet.
5. **Read and present**: briefs and decks that are easy to read on a touch screen.

## Important Notes

**The fork that decides most of this plan is the HarmonyOS version on the MatePad:**
- **HarmonyOS 4.x and earlier (and EMUI)** can install Android APKs (sideloading, AppGallery,
  F-Droid). There are no Google Play Services, so nothing can rely on Google sign-in, Google speech
  or Firebase push.
- **HarmonyOS NEXT / 5+** runs **no Android APKs at all**, only native HarmonyOS apps. Here the
  **browser (Huawei Browser / ArkWeb) is the only way in**, so a PWA is the client.
- So every item below is tagged **[All]** (works through the browser on any version) or **[APK]**
  (only on HarmonyOS 4 and earlier). The [All] items come first.

**What already exists in this repo:**
- The **web app** (`web/`) is the universal client. I found no service worker, web app manifest or
  share-target setup in `web/src`, so the PWA work is new.
- The **desktop app** (`desktop/`) is a macOS-focused shell built on **Tauri 2.10**
  (`desktop/src-tauri/Cargo.toml`). Tauri 2 can build for **Android**, so the same shell could
  become an APK.
- **onyx-cli** (`cli/`, written in Go) already builds `linux/arm64` (`cli/README.md`). A static
  `CGO_ENABLED=0 linux/arm64` binary runs inside Termux unchanged.
- The **browser extension** (`extensions/chrome`) is Manifest V3 and uses `sidePanel`, which
  Firefox for Android doesn't support. A mobile port needs a popup or tab UI instead.
- **Voice providers** are Azure, ElevenLabs and OpenAI (`backend/onyx/voice/providers/`). All
  three are cloud services; none is self-hosted.
- **File extraction** uses MarkItDown and, optionally, Unstructured
  (`backend/onyx/file_processing/`). There's no dedicated OCR pipeline for photos.
- Web search (SearXNG, Exa), crawling (Firecrawl), MCP, memory and the knowledge graph already
  exist and are left alone here.

**Deliberately left out:**
- **On-device LLMs** (llama.cpp, MLC-LLM). Kirin NPUs aren't supported by these runtimes, CPU-only
  models are too slow and too weak for research work, and they wouldn't use Vaultik's indexed
  knowledge. The server does the LLM work; the tablet captures and displays.
- **tldraw.** Its licence needs a commercial key for production use. Excalidraw is MIT.

**Star counts** are approximate, as of late 2026, and only used to rank options. Check each
project's licence (MIT or Apache preferred, since Onyx CE is MIT) before adding a dependency.

**Repo rules (CLAUDE.md, `web/AGENTS.md`):** strict typing, database work under
`backend/onyx/db`, raise `OnyxError`, no `response_model`, every Celery task has `expires=`, and
follow the frontend standards in `web/AGENTS.md`.

## Ranked repos for the MatePad

| # | Repo (≈stars) | Runs on | What it gives the MatePad | Effort |
|---|---|---|---|---|
| 1 | **Serwist** (serwist/serwist), the maintained next-pwa successor | [All] | Makes Onyx an installable PWA: home-screen icon, full screen, cached app shell, **Web Share Target** so "Share → Onyx" works from any app | Medium |
| 2 | **faster-whisper / WhisperX** (SYSTRAN/faster-whisper ≈15k, m-bain/whisperX ≈15k) | [All], on the server, mic through the browser | Accurate dictation and post-call debriefs on the tablet with no Google services; WhisperX adds speaker labels to recorded calls | Medium |
| 3 | **Headscale** (juanfont/headscale ≈30k) + **Tailscale** client (tailscale/tailscale ≈25k) | Server [All]; client [APK] | Private, zero-trust access to a self-hosted Onyx from hotels and client sites without exposing it publicly. On HarmonyOS NEXT, fall back to HTTPS plus SSO | Low |
| 4 | **PaddleOCR** (PaddlePaddle/PaddleOCR ≈55k) | [All], on the server, camera through the browser | Photograph business cards, whiteboards and handwritten notes; get searchable text, plus a "contact card" extraction for follow-ups | Medium |
| 5 | **Excalidraw** (excalidraw/excalidraw ≈115k) | [All], in the browser | M-Pencil sketching inside Onyx (deal maps, org charts, account plans). The sketch is attached to chat and saved as a document | Medium–High |
| 6 | **Tauri 2 mobile**, reusing `desktop/` (tauri-apps/tauri ≈95k) | [APK] | A native Onyx app for HarmonyOS 4 and earlier: fewer browser quirks, an Android share intent, its own window | Medium |
| 7 | **Termux** (termux/termux-app ≈45k) | [APK] | Run `onyx-cli` on the tablet for terminal queries and scripting, with a keyboard cover. No code change needed | Very low |
| 8 | **Onyx extension on Firefox for Android** (port of `extensions/chrome`) | [APK] | "Ask Onyx about this page" while browsing prospects' sites on the tablet | Medium |
| 9 | **Karakeep**, formerly Hoarder (karakeep-app/karakeep ≈20k), optional | [All] web, [APK] app | Self-hosted read-it-later list. Only worth adding if the share target (#1) isn't enough; it would need a new Onyx connector | Medium |

Useful on the device but needing **no Onyx change**, so they're just recommended installs:
**LocalSend** (localsend/localsend ≈60k) for AirDrop-style laptop ↔ tablet file transfer without a
cloud (it has Android and community HarmonyOS builds), and **F-Droid** as the Google-free app store
on [APK] devices.

## Implementation strategy

**Phase 1: a good tablet client on every HarmonyOS version (#1, #3)**
- Add a web app manifest, icons and a Serwist service worker to `web/`. Cache only the app shell
  and static assets. Never cache API responses or chat content, because of tenant and permission
  data. Network-first for navigation, with an offline "reconnect" screen.
- Add a **Web Share Target** route that takes shared text, URLs and files. It creates a new chat
  pre-filled with the shared content, or uploads the files to a user "Inbox" project through the
  existing projects and user-file flow (`backend/onyx/server/features/projects`). Treat this as
  best effort, and check which share-target features ArkWeb actually supports. On [APK] devices,
  the Tauri share intent (Phase 3) covers what's missing.
- Do a **touch and tablet layout pass** at MatePad sizes in both orientations: bigger hit targets,
  a collapsible sidebar, a sticky input that sits above the on-screen keyboard, and a "reading
  mode" for long answers.
- Write a **Headscale/Tailscale deployment guide** in `deployment/`. It's ops-only: Onyx runs
  inside the tailnet and is reached by its tailnet hostname. On HarmonyOS NEXT, document the
  HTTPS + SSO path instead.

**Phase 2: voice and camera capture (#2, #4)**
- Add a **faster-whisper speech-to-text provider** against the existing voice provider interface,
  hosted in the model server (GPU optional) and selectable in the voice admin page. Make the mic
  button prominent on tablet layouts, and add a **"Debrief" mode**: dictate after a call, and the
  transcript plus a structured summary (attendees, next steps, risks) is saved to the user's
  projects.
- For uploaded call audio and video, run **WhisperX** with speaker labels as a user-file
  processing step, so recordings become searchable transcripts. Do it inside the existing user
  file processing worker; any new Celery task gets `expires=` and a timeout inside the task.
- Add **camera capture**: an "Add photo" action using the browser's camera capture input sends
  images through **PaddleOCR** (in a sidecar or the model server) into user files. A "business
  card" option extracts name, title, company, email and phone into a structured note the
  assistant can use for follow-up drafts. Raw images are still kept, so vision LLMs can use them
  too.

**Phase 3: pen and native extras (#5–#8)**
- Embed **Excalidraw** as a sketch panel in chat (lazy-loaded so it doesn't bloat the main bundle)
  with pen and pressure input. On save, attach a PNG to the message for vision models, and store
  the `.excalidraw` JSON in user files so the sketch can be edited again.
- **Tauri Android build of `desktop/`.** Add an Android target, fix up the macOS-only features
  (`macos-private-api`, tray icon) behind platform checks, and handle the share intent. The
  release is a sideloadable APK; no Play Store, because there are no Google services.
- **Termux:** document installing the existing `linux/arm64` onyx-cli build in Termux. No code
  change.
- **Firefox Android extension:** a Manifest V3 build variant of `extensions/chrome` that swaps the
  side panel for a popup or tab page, with `browser_specific_settings` for Gecko.

**Cross-cutting**
- Every server-side feature (speech-to-text, OCR, share inbox) ships **disabled by default** behind
  admin settings, and lives in new modules rather than edits to shared upstream files, so the fork
  keeps rebasing cleanly on upstream Onyx.
- Nothing may depend on Google Play Services, Firebase or Google sign-in.

## Tests

- **Playwright (main coverage):** at a MatePad-like viewport (about 2560×1600 at DPR 2, landscape
  and portrait, touch on), check that the manifest and service worker are served and installable;
  that chat input, sidebar toggle and mic button are usable; that a POST to the share-target route
  with a URL and with a PDF creates a pre-filled chat or an Inbox file; and that the Excalidraw
  panel opens and attaches its PNG. Lives in `web/tests/e2e`.
- **External-dependency unit tests:** transcribe a short WAV fixture through the faster-whisper
  provider; OCR a business-card image fixture and check the extracted fields; and process a short
  two-speaker audio fixture through the WhisperX path and check for speaker labels.
- **Integration test:** upload an audio file and a photo as user files and confirm both become
  searchable text.
- **Manual device check, every release:** install the PWA in Huawei Browser on the actual MatePad,
  test share-to-Onyx and the mic, and on [APK] devices sideload the Tauri build.

## Open questions

1. **Which HarmonyOS version is on the MatePad (Settings → About tablet)?** If it's NEXT/5+,
   items #3 (client side), #6, #7 and #8 drop out, and the PWA becomes the only client.
2. Which MatePad model? Is there an M-Pencil and a keyboard cover? That affects how much the pen
   and Termux items are worth.
3. Is there a GPU host for the model server? Whisper and PaddleOCR run on CPU but are much faster
   on a GPU.
