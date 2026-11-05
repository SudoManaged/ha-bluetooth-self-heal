# Changelog

All notable changes. Versions follow [Semantic Versioning](https://semver.org/).

## 1.0.0 (2026-11-05)

First public release.

- When all chosen Bluetooth sensors have been unavailable for a set time, reloads the chosen Bluetooth adapter and
  waits for a sensor to come back; if none does, restarts Home Assistant (optional) or leaves a notification,
  which a later successful reload dismisses. Runs at most once per cooldown and retries after it.
- Every step is written to the logbook, attached to the automation.
- Tested live on Home Assistant 2024.11.0, 2026.3.3 and 2026.11.0.
