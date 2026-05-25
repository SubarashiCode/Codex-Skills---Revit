# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **Codex Skills** repository — it contains reusable skill definitions that AI coding agents (OpenAI Codex, Claude, etc.) consume to gain domain expertise in **Autodesk Revit add-in development**. It is not a runnable application; there is no build, test, or lint pipeline.

Owner: Ahmed Abdalla / SubarashiCode (vendor ID `AHMD`). License: MIT.

## Structure

Each skill lives in its own directory (e.g. `revit-palettes/`) containing:

- `SKILL.md` — the skill definition: YAML front-matter (name, description, trigger conditions) followed by Markdown knowledge that agents inject into their context when the skill activates.
- `agents/<platform>.yaml` — platform-specific agent interface config (display name, short description, default prompt).

## How Skills Are Used

Skills are **not executed locally**. They are referenced by AI agent platforms. When editing or adding skills:

- The `description` field in the YAML front-matter controls **when** the skill triggers — it must list all relevant keywords and task types.
- The Markdown body is the **knowledge payload** — everything an agent needs to complete Revit add-in tasks without asking follow-up questions.
- Keep knowledge concrete: include code patterns, exact file paths, PowerShell commands, known GUIDs/thumbprints, and gotchas learned from real debugging sessions.

## Adding a New Skill

1. Create a directory named after the skill (kebab-case).
2. Add `SKILL.md` with YAML front-matter (`name`, `description`) and knowledge body.
3. Add `agents/openai.yaml` (and other platform configs as needed).
4. The skill description should be comprehensive enough to match all relevant user queries.

## Key Domain Knowledge Baked Into This Repo

The `revit-palettes` skill encodes hard-won lessons about:

- Revit 2025 targeting (`net8.0-windows`, x64, WPF, local Revit assembly references with `<Private>false</Private>`)
- Native `DockablePane` registration vs. floating WPF windows — when each is appropriate and why fallback silently losing docking is a mistake
- WPF layout bugs specific to Revit's narrow docked panes (scrollbar clipping, `MinWidth`, `ViewportWidth` binding pitfalls)
- Authenticode signing workflow for resolving "Unknown Publisher" — including the distinction between `.addin` manifest vendor fields (cosmetic) and Windows code-signing (what Revit actually checks)
- Deployment with `dotnet build -o` to a fresh folder when Revit locks loaded DLLs
