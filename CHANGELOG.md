# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.1.1] - 2026-02-06

### Fixed

- Relaxed prerequisite checks to only require `.planning/` directory (init, share, consolidate no longer fail when STATE.md, ROADMAP.md, or MILESTONES.md are missing)

## [0.1.0] - 2026-02-06

### Added

- `/gsd-teams:init` command for first-time team configuration
- `/gsd-teams:share` command to share planning state to team directory
- `/gsd-teams:consolidate` command to generate consolidated milestones and status
- `gsd-team-reporter` agent for parallel milestone extraction
- `gsd-team-sync` agent for bounded rsync operations
- Support for parallel sessions via slash notation (e.g., `ian/green`)
- Selective sync mode (full or shallow) for bandwidth-conscious sharing
- Stale session cleanup via `max_session_age_days` configuration
