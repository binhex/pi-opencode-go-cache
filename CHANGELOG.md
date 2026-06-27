# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.3] - 2026-06-27

### Fixed

- Corrected factual inaccuracies in the README and the extension's header comment about what Pi's built-in providers do for opencode-go, verified against `@earendil-works/pi-ai` 0.80.2:
  - Pi's `openai-completions` provider adds **zero** `cache_control` markers for opencode-go (not "at most one") — pi-ai only stamps `cache_control` when a model's compat sets `cacheControlFormat: "anthropic"`, and the opencode-go models don't.
  - Pi's `anthropic-messages` provider stamps **3** breakpoints (system prompt + last tool + last conversation message), not 1.
  - The "Why this and not `PI_CACHE_RETENTION=long`" section previously said Pi "drops" `cache_control` markers — it actually *adds* them, and only when `cacheControlFormat: "anthropic"` is configured.

### Changed

- Updated the "Verification" section to reflect the current opencode-go model roster: 13 models total (11 cacheable + 2 GLM skipped), up from the previously documented 11.

## [0.2.2]

- Wrap the `before_provider_request` hook in a try/catch so a bug in the extension can never break the LLM call.
- Skip cache stamping for GLM (Zhipu) models, which reject `cache_control` with "Extra inputs are not permitted".
- Update cache status immediately on model change via the `model_select` event.
- Drop startup notify; simplify footer status.

## [0.1.2]

- Initial public release.
