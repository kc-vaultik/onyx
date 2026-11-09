# Plan: High-impact open-source integrations for Vaultik's Onyx (tablet-first)

## Issues to Address

Vaultik uses this Onyx fork as its internal knowledge and agent platform. Most of the team's
work is go-to-market and dealmaking: account and market research, intel reports, discovery and
closing calls, outreach and follow-ups, fundraising data rooms, decks, and data analysis. A lot of
it happens away from a laptop, on a Huawei MatePad.

The goal is to bring in the widely used open-source projects that add the most value to that work,
in order of value to effort, without copying features Onyx already has.

Measures of success:
1. Better answers over the documents we actually use (decks, financials, data-room PDFs, call
   transcripts).
2. Cheaper research on the open web that we host ourselves.
3. Using Onyx with voice and touch on the MatePad.
4. Onyx agents can reach the same tools (Notion, Supabase, Vercel, Gmail, etc.) that our other agent
   setups already use, and can kick off automations.

## Important Notes

**Already in Onyx. Turn these on; don't rebuild them:**
- **MarkItDown** (microsoft/markitdown) handles file text extraction
  (`backend/onyx/file_processing/extract_file_text.py`).
- **Unstructured** is an optional file parser (`backend/onyx/file_processing/unstructured.py`).
- **SearXNG** and **Exa** are web search providers
  (`backend/onyx/tools/tool_implementations/web_search/clients/`). SearXNG gives free self-hosted
  search right away.
- **Firecrawl** is used for open-URL crawling
  (`backend/onyx/tools/tool_implementations/open_url/firecrawl.py`).
- **Langfuse** and **Braintrust** tracing are in `backend/onyx/tracing/`. Turning on Langfuse is a
  config change.
- **MCP client** support is in `backend/onyx/server/features/mcp/` and
  `tools/tool_implementations/mcp`.
- A **memory** tool and a **Knowledge Graph** are in `backend/onyx/kg`, so mem0 and Graphiti are
  low priority.
- **Voice providers** are Azure, ElevenLabs and OpenAI (`backend/onyx/voice/providers/`). All three
  are paid cloud services; there is no self-hosted option.
- The **Gong, Fireflies, HubSpot, Salesforce, Gmail, Notion and Linear** connectors already exist.

**MatePad constraints:**
- HarmonyOS/Android, with no Google Play Services. The Tauri desktop app (`desktop/`) doesn't run
  there, so the **web app in the browser, installed as a PWA**, is the client.
- The plan needs touch-friendly chat, offline-tolerant loading, and voice input that doesn't rely
  on Google speech services. That last point is a strong reason for self-hosted speech-to-text.

**Star counts** are approximate, as of late 2026, and only used to rank options. Check them, and each
project's license (MIT/Apache is preferred because Onyx CE is MIT), before adding a dependency.

**Repo rules (CLAUDE.md):** database work goes under `backend/onyx/db`; raise `OnyxError`, not
`HTTPException`; no `response_model`; every Celery task has `expires=`; and new tasks go under
`background/celery/tasks/`.

## Ranked integrations

| # | Project (≈stars) | Where it plugs in | Vaultik value | Effort |
|---|---|---|---|---|
| 1 | **MCP reference + vendor servers** (modelcontextprotocol/servers, ≈75k) | Existing MCP client: a curated set of presets | Onyx agents can read and write Notion, Supabase, Vercel, GitHub and Drive, the same way our Claude agents do | Low |
| 2 | **Docling** (docling-project/docling, ≈45k) | New optional PDF/PPTX/XLSX extractor next to MarkItDown in `file_processing` | Keeps tables and layout in pitch decks, financials and data-room PDFs, which is the biggest single gain in answer quality for fundraising and intel work | Medium |
| 3 | **faster-whisper / WhisperX** (SYSTRAN/faster-whisper ≈15k, m-bain/whisperX ≈15k) | New self-hosted speech-to-text provider in `backend/onyx/voice/providers/`, served through the model server; can also transcribe uploaded call audio and video | MatePad dictation without Google; turns call recordings into searchable, speaker-labelled transcripts for discovery and closing notes | Medium |
| 4 | **Crawl4AI** (unclecode/crawl4ai, ≈55k) | New open-URL backend next to Firecrawl, plus an optional fetcher for the Web connector | Self-hosted, free crawling of JS-heavy sites into LLM-ready markdown for account research and intel reports | Low–Medium |
| 5 | **n8n** (n8n-io/n8n, ≈150k) | Two-way: n8n calls the Onyx chat/search API and MCP; Onyx triggers n8n webhooks through a custom "Automation" tool | Follow-up sequences, CRM updates and "brief me before this meeting" workflows built on Onyx answers | Low (mostly config and templates) |
| 6 | **Langfuse** (langfuse/langfuse, ≈20k), already integrated | Enable `tracing/langfuse_tracing_processor.py` against a self-hosted Langfuse | Shows cost and quality per agent, so we can tune research and outreach assistants | Very low |
| 7 | **SearXNG** (searxng/searxng, ≈20k), already integrated | Deploy the container and select it as the web search provider | Removes per-query search cost for research-heavy assistants | Very low |
| 8 | **browser-use** (browser-use/browser-use, ≈75k) | A tool that runs inside the existing Build sandbox (`server/features/build/sandbox`) | Multi-step browsing tasks such as portal lookups and filling forms on public sites. Higher risk, so keep it to the admin role | High |
| 9 | **Kokoro / Piper TTS** (hexgrad/kokoro, rhasspy/piper) | Self-hosted text-to-speech provider next to ElevenLabs | Hands-free read-outs of briefs on the tablet; lower priority than speech-to-text | Medium |

Deliberately left out: Open WebUI, Dify and RAGFlow (they overlap with Onyx itself); mem0 and
Graphiti (Onyx already has memory and a knowledge graph; look again only if those fall short);
Ollama and vLLM (already reachable through LiteLLM as an LLM provider config, with no code needed).

## Implementation strategy

**Phase 0: switch on what's already there (config only, no code)**
- Deploy SearXNG and Langfuse alongside the existing docker compose / helm, and point the Onyx
  admin settings at them.
- Make the web app installable as a PWA (manifest, icons, service worker for the app shell) and
  check the chat and sidebar layouts at MatePad sizes, in landscape and portrait, following
  `web/AGENTS.md`. This is the only frontend work in Phase 0 and it's what enables everything
  after it on the tablet.

**Phase 1: highest value**
1. *MCP presets.* Add an admin-visible catalogue of pre-filled MCP server definitions (name, URL,
   auth type) that populates the existing MCP server form. Start with Notion, Supabase, Vercel,
   GitHub, Google Drive and Gmail. Ship no secrets; auth goes through the existing OAuth and
   user-token flows (`server/features/oauth_config`, `user_oauth_token`).
2. *Docling extractor.* Add a setting (env and admin) to route PDF/PPTX/XLSX through Docling when
   it's enabled, and fall back to MarkItDown on failure or timeout. Run Docling in the model server
   or a sidecar so the docprocessing workers stay light. Put the timeout inside the call: Celery
   time limits don't work in thread pools. Emit tables as markdown so chunking keeps rows together.
3. *Self-hosted speech-to-text.* Implement a faster-whisper provider against the existing voice
   provider interface, host it in the model server image (GPU optional), and expose it in the
   voice admin page. Then add a call-recordings ingestion path: audio and video uploaded as user
   files or through the file connector get transcribed (WhisperX adds speaker labels) and indexed
   like any other document.

**Phase 2**
4. *Crawl4AI backend.* Add it as a third open-URL provider next to Firecrawl and the Onyx crawler,
   chosen in the same admin settings, and offer it as an optional fetch strategy in the Web
   connector.
5. *n8n.* (a) Document and template n8n workflows that call the Onyx API and MCP server, such as a
   pre-meeting brief, follow-up drafting and a weekly intel digest. (b) Add a generic "Trigger
   Automation" custom tool, admin-configured with webhook URL and schema, so assistants can hand
   work to n8n. Reuse the custom tool framework (`tool_implementations/custom`) instead of adding a
   new tool type where possible.

**Phase 3 (only if Phases 1–2 prove out)**
6. Run browser-use inside the Build sandbox behind an admin-only permission, with domain
   allowlists.
7. Add a self-hosted text-to-speech provider (Kokoro or Piper).

**Cross-cutting**
- Every new integration ships disabled by default behind a setting, so the fork stays easy to
  rebase on upstream Onyx.
- Keep fork-only code isolated: new provider modules and new presets, with as few edits to shared
  files as possible. That keeps merge conflicts with upstream small.
- Track each phase's impact in Langfuse (Phase 0) by comparing answer quality and cost before and
  after.

## Tests

- **Docling:** an external-dependency unit test that runs a real table-heavy PDF and PPTX fixture
  through extraction and checks that table cells survive as markdown rows, and that a forced
  Docling failure falls back to MarkItDown.
- **Speech-to-text provider:** an external-dependency unit test that transcribes a short WAV
  fixture through the provider interface. An integration test that uploads an audio file as a
  user file and confirms the transcript is searchable.
- **Crawl4AI:** an external-dependency unit test that fetches a known public page through the new
  open-URL provider and checks for non-empty markdown.
- **MCP presets and the n8n automation tool:** an integration test that creates an MCP server from
  a preset through the API and lists its tools, and one that configures the automation tool
  against a local mock webhook and confirms an assistant call reaches it.
- **Tablet and PWA:** a Playwright test at a MatePad-like viewport (about 2560×1600 at DPR 2, both
  orientations) that checks the chat input, sidebar toggle and voice button are usable and the
  manifest is served.

## Open questions for the team

1. "MatePad": this plan assumes it means the Huawei MatePad tablet as the main client. If you meant
   something else (a project or product name), the PWA work in Phase 0 and the tablet test change.
2. Which CRM is the source of truth (HubSpot, Salesforce, Attio, Twenty)? It decides whether a CRM
   connector or MCP preset goes into Phase 1.
3. Is there a GPU host for the model server? Docling and Whisper run on CPU but are far faster on a
   GPU.
