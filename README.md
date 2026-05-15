# Plow

## Purpose

This SEED installs the latest production build of [Plow](https://plow.co) on macOS. Plow is a personal AI assistant — calendars, messages, email, files, and whatever else you throw at it. Installing this SEED downloads the current `.dmg` from `https://plow.co/download`, replaces any existing `/Applications/Plow.app`, and launches the new build.

This SEED also installs [`seed-os-manager`](https://github.com/plow-pbc/seed-os-manager) as a hard prerequisite — its signed `seedctl` CLI is the TCC principal under which all Apple Events fired by this install and by downstream Plow SEEDs are attributed. Without it, macOS would silently block AppleScript run from an unidentified shell process (the `-1743` error mode). Downstream Plow SEEDs can rely on `seedctl` being at the conventional path `/Applications/Seed OS Manager.app/Contents/MacOS/seedctl` after this SEED runs (override with the `SEEDCTL` env var).

## Install

Tell any AI agent:

> Install `https://github.com/plow-pbc/plow-seed`

## License

MIT
