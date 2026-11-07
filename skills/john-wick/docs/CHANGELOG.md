# Changelog

## 2026-11-07

- Added John Wick, a cleanup sweep skill organized as passes that share one scope → sweep → confirm → report method. Sweeps report findings and change code only on request.
- First pass: implicit state machines. Covers flag soup, unowned status fields, presence-encoded phases, temporal coupling, counter phases, parser mode flags, async lifecycles, distributed machines, and bypassed explicit machines.
- Includes multi-language `rg` sweep signals, clustering by owner, state/transition reconstruction, confirmation and rejection criteria, severity, a report format, and a hand-off procedure with per-language explicit forms.
