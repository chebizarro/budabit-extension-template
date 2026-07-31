# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - Unreleased

### Added

- `budabit-sdk` now ships host theme helpers (`applyHostTheme`, `seedHostThemeFallback`, `watchHostTheme`) in the published package. The `theme.ts` module existed in source and was exported from the main entry point, but was added to the repo after the `budabit-sdk@0.2.0` publish cut, so the npm tarball for 0.2.0 predates it and has no `dist/theme.js`. 0.3.0 includes it.
- Documented the host theme helpers in the SDK README.

### Changed

- Bumped `create-budabit-widget` and the scaffolded template's `budabit-sdk` dependency to `^0.3.0`.

## [0.2.0] - Unreleased

### Changed

- Renamed the project and remaining branding to Budabit.
- Updated `budabit-sdk` to the current extension bridge, typed action map, subscription lifecycle, manifest flags, worker bridge, and test mocks.
- Updated `create-budabit-widget` and generated projects to depend on `budabit-sdk@^0.2.0`.
- Added current lifecycle initialization, publishing scripts, CORS, Vitest exclusions, and defensive ESLint warnings.

## [0.1.0] - 2025-01-08

### Added

- Initial template release
- Monorepo structure with pnpm workspaces
- `@budabit/ext-shared` package with bridge and types
- `@budabit/ext-iframe` package with Svelte 5 demo app
- `@budabit/ext-worker` package (stubbed)
- `@budabit/ext-manifest` package with CLI generator
- `@budabit/test-utils` package with mocks
- Comprehensive documentation
- CI/CD workflow with GitHub Actions
- E2E tests with Playwright
- Unit tests with Vitest
- 95%+ coverage enforcement
- ESLint and Prettier configuration
- TypeScript strict mode
- Single-file build output

[0.1.0]: https://github.com/your-org/budabit-extension-template/releases/tag/v0.1.0
