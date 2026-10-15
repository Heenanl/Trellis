# Trellis Testing Notes

Last updated: 2026-10-08

This file records hands-on setup and user-testing feedback from Windows 11 with Docker Desktop using the production `compose.yaml` stack. Add future findings here while working on the `testing` branch.

## Test Summary

| Area | Result | Notes |
| --- | --- | --- |
| Docker prerequisite | Pass | Docker Desktop 28.4.0 and Compose 2.39.2 were installed. Docker had about 15.4 GiB of memory available. |
| Compose configuration | Pass | `docker compose config --quiet` completed successfully after `.env` was created. |
| Production image build | Pass with workaround | API, author, and frontend images built after temporary package-mirror overrides. No mirror changes remain in tracked Dockerfiles. |
| Container startup | Pass | Neo4j became healthy; API, author, frontend, and nginx remained running. |
| Web smoke test | Pass after recovery | `http://127.0.0.1:8080` returned HTTP 200 with the title `Trellis`. |
| Browser onboarding | Not tested | Provider key, account creation, and first-source workflows still require user testing. |
| Lesson generation | Not tested | Requires completed onboarding, an AI provider key, and token usage. |

## Blockers

### Package registry TLS failures

The initial production build could not reach standard package download hosts:

- Python failed against `files.pythonhosted.org` with `SSLV3_ALERT_HANDSHAKE_FAILURE`.
- npm failed against `registry.npmjs.org` with `ERR_SSL_SSL/TLS_ALERT_HANDSHAKE_FAILURE`, eventually surfacing the misleading `Exit handler never called!` error.
- The failures occurred from both Windows and clean Docker containers, so they appear to be caused by the local network or proxy rather than Trellis.

Temporary workaround used for this test:

- Python dependencies were built through `https://pypi.tuna.tsinghua.edu.cn/simple`.
- npm dependencies were built through `https://repo.huaweicloud.com/repository/npm/` with increased fetch retries.
- Temporary Dockerfile build arguments were removed after the images were built.

Suggested repository improvement: document optional, user-supplied `PIP_INDEX_URL` and `NPM_CONFIG_REGISTRY` build arguments without changing secure defaults. This would help users on corporate, filtered, or regionally constrained networks.

## Issues Found

### Local port forwarding became stale after Docker Desktop resumed

Observed behavior:

1. All Trellis containers reported `Up`, and Neo4j reported `healthy`.
2. Port `127.0.0.1:8080` accepted TCP connections.
3. HTTP requests received an empty response.
4. From inside the nginx container, both `frontend:3000` and `api:8000/docs` returned HTTP 200.
5. Recreating only the web container restored host access:

```powershell
docker compose up -d --no-build --force-recreate web
```

This is likely a Docker Desktop host-port forwarding issue rather than an nginx or application defect. It is still worth adding this recovery command to troubleshooting documentation because the visible symptom makes Trellis appear down while every container appears healthy.

### Limited service health reporting

Only Neo4j currently exposes a Compose health check. API, frontend, author, and web can show `Up` even when the complete user path is unavailable.

Suggested improvement: add lightweight health checks for each long-running service and make nginx readiness depend on healthy upstreams where practical.

### Build errors obscure the underlying network problem

The frontend's final error was `Exit handler never called!`, while verbose registry testing showed the real TLS handshake failure. A user following only the README may repeatedly retry without understanding the cause.

Suggested improvement: add a troubleshooting section with direct registry connectivity checks and explain that npm's exit-handler message may be secondary to network failure.

### Microsoft Foundry keys are accepted as OpenAI keys without endpoint validation

The setup screen accepted a Microsoft Foundry project API key under the OpenAI preset. Trellis then selected its standard OpenAI models and attempted to use the key as an OpenAI Platform credential. A Foundry key also requires the Foundry resource endpoint and model deployment names, which the current provider form does not collect.

Observed saved configuration:

- Provider and embedding provider: `openai`
- Mentor model: `openai/gpt-5.5`
- Extract model: `openai/gpt-5.5-mini`
- Embedding model: `openai/text-embedding-3-large`

Suggested improvement: add Microsoft Foundry as a distinct provider with endpoint and deployment-name fields. During setup, perform a small provider validation request and explain whether a failure concerns credentials, endpoint format, deployment name, quota, or model capability.

### OpenAI reasoning models rejected Trellis's fixed temperature

Before any provider request was sent, LiteLLM rejected `temperature=0.2` for the configured `gpt-5.5` and `gpt-5.5-mini` reasoning models. This caused HTTP 500 responses in learning and extraction flows and left the learning-roadmap experience waiting or retrying.

Local testing-branch fix: enable LiteLLM's unsupported-parameter filtering in the shared LLM wrapper, with a regression test confirming that the OpenAI provider mapper omits the unsupported temperature. The focused test passes in the API container.

### Learn button remains disabled with an empty map

Observed on 2026-10-13: the **Learn** navigation button was greyed out and could not be selected. The Explore view contained no concepts. The interface did not explain what condition enables Learn or direct the learner to the next required action.

The disabled state may be a consequence of the source-extraction failure below, but that relationship has not yet been verified. Even when disabling Learn is intentional, the interface should explain that at least one successfully imported concept is required.

### Databricks Lakebase product link failed extraction

Observed on 2026-10-13 when importing this link:

<https://www.databricks.com/product/lakebase?scid=7018Y000001f8FGQAY&utm_medium=paid+search&utm_source=google&utm_campaign=23587488715&utm_adgroup=196202224969&utm_content=product+page&utm_offer=lakebase&utm_ad=797797815309&utm_term=databricks%20lakebase&gad_source=1&gad_campaignid=23587488715&gbraid=0AAAAABYBeAiFJRXg-OfMVH9bxmLKz2Y1s&gclid=CjwKCAjwrbfWBhAeEiwAc2e6PrUIXJeX1tar2IEOaUdMLyg7QAoIXPXm5u_xCZNUzxfJvlwNfGdbmRoCIOQQAvD_BwE>

Visible results:

- `HTTP 500` error notification.
- `No concepts found in 'Lakebase'.`
- `1 of 1 parts skipped after a failed extraction (part 1).`
- The graph remained empty and Learn remained disabled.

Verified cause: the API logs show `401 invalid_api_key` during concept extraction. Trellis had stored the Microsoft Foundry credential under the `openai` provider and sent it to the OpenAI Platform endpoint for `openai/gpt-5.5-mini`. A Foundry key is not valid at that endpoint; Foundry requires its resource endpoint and deployed model names. The link retrieval itself was not the reported failure stage.

Suggested improvement: validate provider credentials and endpoint routing during setup, identify the failed stage in the user-facing message, and classify authentication failures separately from page-fetch or extraction-content failures. Repeating the same request cannot succeed until the provider configuration is corrected.

### Error notifications ignore the selected English language

Observed on 2026-10-13: the interface was displayed in English, but extraction notifications appeared in Dutch, including `Er ging iets mis` and `1 van 1 delen overgeslagen na een mislukte extractie (deel 1).`

This mixed-language state makes errors harder to understand and suggests that some backend-generated messages are returned as display text instead of stable error codes translated by the frontend. All user-facing errors should follow the profile's selected interface language. Suggested improvement: return structured error codes and parameters from the API, translate them in the frontend, and use a localized generic fallback only for unknown errors.

### Model provider and API key cannot be changed after onboarding

Observed on 2026-10-13: after the initial setup, **My profile → Settings** has no control for changing the model provider, replacing an API key, or selecting a separate embedding provider. The authenticated backend settings endpoint supports these changes, but ordinary users cannot reach it through the application UI.

This leaves a failed provider configuration effectively permanent without developer tools or direct API use. Suggested improvement: add an owner-only AI provider card showing the active providers and masked key status, with controls to replace credentials and validate chat and embedding calls before saving. Never redisplay stored key values.

### Claude setup unexpectedly requires a second AI provider

Observed on 2026-10-13: after obtaining a Claude API key, the user could not simply switch Trellis to Claude. Anthropic does not provide an embedding model, so Trellis also requires a compatible embedding provider and a second API key. For the existing 3072-dimensional vector index, the available compatible choices are Gemini or OpenAI.

This requirement creates unexpected setup friction, especially when the user is trying to recover from an invalid provider configuration. Suggested improvement: explain before provider selection that Claude requires a separate embedding service, recommend compatible choices based on the current vector-index dimension, and present both key fields in one guided flow. Validate both services before saving and clearly identify which credential fails.

### Gemini capacity failures appear as generic HTTP 500 errors

Observed on 2026-10-13 after switching both generation and embeddings to Gemini: the credential successfully listed available models, and `gemini-embedding-2-preview` returned a valid 3072-dimensional embedding. Generation requests to `gemini-3.7-flash`, `gemini-3.8-flash`, `gemini-flash-lite-latest`, and `gemini-3.5-flash-lite` all returned Google `503 UNAVAILABLE` responses stating that the model was experiencing high demand.

Trellis displayed only `Er ging iets mis (HTTP 500)` and failed extraction after two nearly immediate attempts. This makes a temporary upstream capacity problem look like a broken key or application defect. Suggested improvement: preserve provider status classes, show a localized temporary-capacity message, use exponential backoff with jitter for 429/503 responses, and offer a delayed retry instead of marking the source as permanently failed.

The Learn mentor failed separately with Google `429 RESOURCE_EXHAUSTED` for `gemini-3.1-pro`: all applicable free-tier request and input-token limits were reported as `0`, with a provider retry delay of about 7 hours 54 minutes. Trellis again displayed only `Er ging iets mis (HTTP 500)`. The UI should distinguish an unavailable free-tier model from temporary capacity, show the provider's retry time, and recommend enabling billing or selecting a model with nonzero quota.

## Feature Suggestions

1. Add a cross-platform preflight command that checks Docker availability, memory, required ports, `.env`, package registry connectivity, and Compose interpolation.
2. Add a `docker compose` diagnostics command or script that prints service health, recent errors, internal upstream checks, and the public URL status.
3. Provide configurable package mirrors as documented build arguments for restricted networks.
4. Add an application health endpoint that checks API-to-Neo4j connectivity without exposing secrets.
5. Show application version and build information in Settings to make bug reports reproducible.
6. Add an exportable support bundle with redacted configuration, container states, and recent logs.
7. Add a Microsoft Foundry provider option supporting resource endpoint, chat/extraction deployment names, and embedding deployment name/dimension.

## Additional Product Feedback

### Highest priority

1. **Validate the complete AI setup before onboarding finishes.** Test one small chat completion and one embedding, then report credential, endpoint, deployment, quota, and model-capability failures separately. This would have prevented the Foundry key from appearing successfully configured.
2. **Explain the provenance of topic-only imports.** A topic entered without links is generated from the selected model's existing knowledge; it is not researched on the internet. Label the resulting source as AI-generated, show the model and generation date, and prompt the learner to add authoritative sources before treating it as grounded material.
3. **Make every long-running AI action recoverable.** Lesson generation already exposes stages and cancellation. Apply the same durable job pattern to topic generation, extraction, study-guide writing, transcription, and roadmap updates so a refresh does not lose progress and failures retain a retryable error record.
4. **Expand cost controls beyond lessons.** Show a rough estimate before topic generation, transcription, study-guide rewriting, and bulk extraction. Add optional daily or monthly budgets and a confirmation threshold for unusually expensive operations.
5. **Add learner-data portability and deletion controls.** Provide a human-readable export of concepts, relationships, sources, conversations, goals, memory, and lesson history, plus scoped deletion for a source, conversation, profile, or all learner data. Show exactly which context will be sent to the model provider before a request when practical.

### Next priority

6. **Add concept maintenance tools.** Let users merge likely duplicates, rename concepts, edit prerequisite relationships, preview the impact of deletion, and undo recent graph changes. Extraction from multiple sources will otherwise make the map progressively harder to trust.
7. **Surface evidence and disagreement.** For important explanations and prerequisite links, show which stored excerpts support them, distinguish model-generated explanation from sourced claims, and flag conflicting sources instead of silently presenting one answer.
8. **Track source freshness and versions.** Store retrieval date and content hash for links, allow a source to be refreshed, summarize what changed, and let the learner decide whether concepts or guides should be regenerated.
9. **Support an embedding-degraded mode.** If embeddings fail, keep basic keyword search and non-vector learning features available, display a persistent warning, and provide a focused repair action rather than making the whole setup appear broken.
10. **Add a short diagnostic check before a learning path begins.** Let learners optionally demonstrate what they already know, then skip or shorten familiar material. Keep the result editable and avoid treating one answer as proof of mastery.
11. **Improve accessibility testing.** Add automated keyboard, focus-order, contrast, reduced-motion, and screen-reader checks for onboarding, source review, roadmap navigation, mentor chat, and interactive lessons. Include a non-canvas list alternative for the graph and roadmap.
12. **Provide a source-quality review queue.** Flag sources with missing authorship, stale retrieval dates, extraction gaps, low citation coverage, or apparent conflicts. Keep quality judgments transparent and learner-controlled rather than automatically rejecting material.

### Product-positioning feedback

- The strongest differentiator is not generic AI tutoring; it is the learner-owned graph, source-grounded lessons, editable memory, and self-hosted data. Onboarding and empty states should demonstrate that loop immediately.
- The product should clearly distinguish three kinds of knowledge: learner-provided sources, model-generated explanations, and observed learning history. Mixing them weakens trust even when the content is correct.
- A small, reliable first success is more valuable than exposing every advanced option during setup. A guided flow of one source, three reviewed concepts, one mentor exchange, and one quick check would demonstrate the core experience with lower cost and fewer failure points.

## User-Testing Improvements

1. Give testers a short scenario checklist: complete onboarding, add one topic, add one URL or transcript, generate a study guide, ask the mentor a question, create a lesson, and verify a backup.
2. Ask testers to record expected behavior, actual behavior, exact error text, browser, operating system, and whether the failure survives a refresh or container restart.
3. Clearly label actions that can consume AI tokens and show an estimated cost before first use.
4. Include a visible setup-progress indicator and a retry path for provider-key validation.
5. Add a first-run diagnostics screen that distinguishes browser, nginx, API, database, and AI-provider failures.
6. Provide a feedback link that pre-fills version and environment details while excluding API keys and passwords.

## Recommended Next Test Pass

1. Complete browser onboarding with a low-cost provider configuration.
2. Verify login, logout, and password reset.
3. Import one source of each practical type, including a failed or blocked URL.
4. Exercise mentor chat continuity across a page reload.
5. Generate and validate one interactive lesson on desktop and phone-sized layouts.
6. Create, verify, and restore a backup in a disposable test environment.
7. Restart Docker Desktop and reboot Windows, confirming that `http://localhost:8080` recovers automatically.

## Future Test Entry Template

```text
Date:
Environment:
Area tested:
Expected:
Actual:
Reproduction steps:
Severity: blocker | major | minor | suggestion
Workaround:
Evidence/logs:
Status: open | resolved | cannot reproduce
```