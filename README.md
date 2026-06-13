# Claude Code Token Optimization — The Complete Playbook

> **Cut your token bill by 39% before you even start coding.**
> Real numbers. Real configs. No BS.

<p align="center">
  <img src="https://img.shields.io/github/stars/tomerose/claude-code-token-optimizer?style=for-the-badge" alt="Stars">
  <img src="https://img.shields.io/github/license/tomerose/claude-code-token-optimizer?style=for-the-badge" alt="License">
  <img src="https://img.shields.io/badge/Claude%20Code-Compatible-blue?style=for-the-badge" alt="Claude Code">
  <img src="https://img.shields.io/badge/Token%20Savings-39%25-green?style=for-the-badge" alt="Token Savings">
  <img src="https://img.shields.io/badge/Platform-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey?style=for-the-badge" alt="Platform">
</p>

---

## The Problem Nobody Talks About

Every Claude Code session starts with **~24,000 tokens of invisible overhead** before you type a single word. That's your system prompt, skill descriptions, MCP tool schemas, CLAUDE.md files, memory injections, and hook outputs — all burned on every. single. turn.

If you're using 30+ skills, 5 MCP servers, and verbose CLAUDE.md files, you're paying for ~70% overhead and ~30% actual work. This guide shows you exactly how to flip that ratio.

```
┌─────────────────────────────────────────────────┐
│  BEFORE:  24,000 tokens overhead per turn        │
│  ████████████████████████████░░░░░░░░  (70%)     │
│  AFTER:   14,600 tokens overhead per turn        │
│  ██████████████░░░░░░░░░░░░░░░░░░░░  (39% saved) │
└─────────────────────────────────────────────────┘
```

## What You'll Get

| Area | Technique | Token Savings |
|------|-----------|:---:|
| Skills | 76 skills → 8 hot + 58 cold + 10 off | ~7,400/turn |
| Rules | 9 files → 1 trigger index | ~2,200/session |
| Context | Auto-compact at 50% window fill | Prevents overflow |
| Runtime | squeez: Bash output compression + dedup | ~84% per command |
| Documents | markitdown: PDF/Office → clean Markdown | Avoids PDF noise |
| Safety | deny rules for destructive commands | Zero-risk automation |

## The Four-Layer Defense

```
┌──────────────────────────────────────────────────────┐
│                 LAYER 1: FIXED OVERHEAD               │
│  skillOverrides │ trigger-index │ autoCompact         │
│  Cut the invisible tax you pay on every turn.         │
├──────────────────────────────────────────────────────┤
│                 LAYER 2: RUNTIME COMPRESSION           │
│  squeez hooks │ Bash dedup │ Read/Grep limits         │
│  Compress tool output BEFORE it enters context.       │
├──────────────────────────────────────────────────────┤
│                 LAYER 3: TOOLCHAIN                    │
│  markitdown │ PDF/Office → clean Markdown             │
│  Feed the LLM clean input. Never dump raw PDFs.       │
├──────────────────────────────────────────────────────┤
│                 LAYER 4: SECURITY                     │
│  deny rules │ bypassPermissions guardrails            │
│  Full automation with hard safety boundaries.         │
└──────────────────────────────────────────────────────┘
```

---

## Layer 1: Slash the Fixed Overhead

### 1.1 Skill Classification — `skillOverrides`

Claude Code loads ALL skill descriptions into context every turn. 60+ skills × ~150 tokens each = **9,000 tokens/turn** just for skills you never use.

The fix: categorize everything.

```json
// ~/.claude/settings.json
{
  "skillOverrides": {
    "session-maestro": "on",              // Always loaded (8 skills)
    "stop-slop": "on",
    "agent-reach": "on",
    "pdf": "on",
    "planning-with-files": "on",
    "impeccable": "on",
    "find-skills": "on",
    "claude-mem:mem-search": "on",

    "agent-skills:build": "user-invocable-only",  // /command works (58 skills)
    "agent-skills:test": "user-invocable-only",
    "deep-research": "user-invocable-only",
    "zhangxuefeng-perspective": "user-invocable-only",
    // ... 50+ more

    "algorithmic-art": "off",             // Never use (10 skills)
    "brand-guidelines": "off",
    "cartesia-api": "off",
    "update-config": "off"
  }
}
```

| Mode | Context Load | `/skill-name` Works |
|------|:---:|:---:|
| `"on"` | Full description (150 tokens) | ✅ |
| `"user-invocable-only"` | Name only (10 tokens) | ✅ |
| `"off"` | Nothing (0 tokens) | ❌ |

**→ 9,000 → 1,780 tokens/turn (80% reduction)**

### 1.2 Rules On-Demand — Trigger Index

Instead of loading 9 rule files (281 lines) into every session, keep one lightweight trigger index:

```markdown
# ~/.claude/rules/trigger-index.md (15 lines, 300 tokens)

| Trigger Keywords | Rule File | When to Read |
|-----------------|-----------|-------------|
| Frontend/UI/Component | `frontend.md` | When modifying UI |
| Backend/API/Database | `backend.md` | When writing APIs |
| Auth/Login/Permission | `auth.md` | When touching auth |
```

Move full rule files to a non-auto-loaded location. When a trigger word matches, Claude reads the file on demand.

**→ 2,500 → 300 tokens/session (88% reduction)**

### 1.3 Auto-Compact Aggressively

Default auto-compact triggers at ~95% context fill — by then it's already too late.

```json
{
  "env": {
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "50"
  },
  "autoCompactEnabled": true
}
```

Compacts at 50% window fill instead of 95%. Keeps the context lean, responses fast, and prevents the dreaded "context window full" error.

---

## Layer 2: Runtime Compression with squeez

[squeez](https://github.com/claudioemmanuel/squeez) (90.7% compression, open source) wraps every Bash command through a 4-stage pipeline:

```bash
npm install -g squeez
squeez setup --host=claude-code
```

What it does to your tool outputs:

| Command | Before | After | Savings |
|---------|--------|-------|:---:|
| `git log --oneline` (500 commits) | Full 500 lines | Summary + last 20 | 90% |
| `npm list` (200 packages) | All deps printed | Deduped + truncated | 95% |
| Same file read twice | Full content again | "same as call #3" | 99% |
| `ls` large directory | 300 files | Capped at 120 lines | 60% |

```ini
# ~/.claude/squeez/config.ini
persona = ultra              # Maximum compression
max_lines = 120              # Trim output >120 lines
adaptive_intensity = true    # Auto-escalate as context fills
dedup_min = 2                # Cross-call dedup after 2nd call
read_max_lines = 300         # Truncate file reads
grep_max_results = 100       # Cap grep results
```

---

## Layer 3: Clean Document Input

Never feed raw PDFs or Office documents to Claude — PDF parsing produces massive noise tokens.

```bash
pip install "markitdown[all]"
markitdown lecture.pdf -o lecture.md
markitdown exam.docx -o exam.md
```

[markitdown](https://github.com/microsoft/markitdown) (152k ★, Microsoft official) converts PDF/Word/PPT/Excel/EPUB/HTML into clean, structured Markdown. Feed the MD to Claude instead of the original file.

---

## Layer 4: Security Guardrails for Full Automation

`bypassPermissions` gives you zero prompts — but you NEED hard boundaries:

```json
{
  "permissions": {
    "allow": [
      "Bash(*)", "PowerShell(*)", "WebFetch(*)",
      "Edit(*)", "Write(*)", "Read(*)", "Glob(*)", "Grep(*)",
      "Skill(*)", "Agent(*)", "Task(*)", "Workflow(*)"
    ],
    "deny": [
      "Bash(rm -rf /*)",
      "Bash(dd if=*)",
      "Bash(sudo rm *)",
      "PowerShell(format *)",
      "PowerShell(diskpart *)",
      "PowerShell(shutdown *)",
      "PowerShell(bcdedit *)",
      "PowerShell(reg delete HKLM*)",
      "PowerShell(Remove-Item C:\\Windows\\System32*)"
    ],
    "defaultMode": "bypassPermissions"
  }
}
```

Deny rules override bypassPermissions. Daily operations are instant; system-destroying commands are blocked.

---

## Results: Before vs After

| Metric | Before | After | Improvement |
|--------|:-----:|:----:|:---:|
| Skills overhead | 9,000 tokens | 1,780 tokens | ↓80% |
| Rules overhead | 2,500 tokens | 300 tokens | ↓88% |
| Bash output (avg) | ~2,000 tokens | ~320 tokens | ↓84% |
| Repeated file read | ~1,500 tokens | ~13 tokens | ↓99% |
| **Total per-turn overhead** | **~24,000 tokens** | **~14,600 tokens** | **↓39%** |
| Safety prompts | Yes | Zero | Eliminated |

In a 30-turn session: **~282,000 tokens saved**. At Anthropic Sonnet pricing ($3/MTok input), that's **~$0.85 saved per session**. At 20 sessions/month: **$17/month** — and if you're on DeepSeek or using a cheaper provider, the proportional savings are even larger relative to your total bill.

---

## Quick Start (5 Minutes)

```bash
# 1. Classify your skills (copy the skillOverrides block above)
#    → Paste into ~/.claude/settings.json

# 2. Move rules, create trigger index
#    → Move ~/.claude/rules/*.md to a non-loaded directory
#    → Create ~/.claude/rules/trigger-index.md

# 3. Auto-compact
#    → Add CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50 to settings.json env

# 4. Install squeez
npm install -g squeez
squeez setup --host=claude-code

# 5. Install markitdown
pip install "markitdown[all]"

# 6. Add deny rules
#    → Copy the deny block above into ~/.claude/settings.json
```

Restart Claude Code. Done.

---

## Tools Used

| Tool | Stars | What It Does |
|------|:---:|------|
| [squeez](https://github.com/claudioemmanuel/squeez) | ★ | Tool output compression (4-stage pipeline) |
| [markitdown](https://github.com/microsoft/markitdown) | 152k | Microsoft's file→Markdown converter |
| [PowerToys](https://github.com/microsoft/PowerToys) | 134k | Windows productivity (FancyZones, Run, OCR) |
| [agent-skills](https://github.com/addyosmani/agent-skills) | 57k | Production-grade AI coding skills |
| [stop-slop](https://github.com/hardikpandya/stop-slop) | 10k | Remove AI writing patterns |

---

## Who This Is For

- **Heavy Claude Code users** who feel context filling too fast
- **Developers on a budget** who want to maximize every token
- **Windows users** tired of Linux-only optimization guides
- **Anyone running 20+ skills** who only uses 5 of them
- **Teams** that want to standardize Claude Code configs

---

## Philosophy

> The best token optimization is the one you don't notice.
> No workflow changes. No cognitive overhead. Just less noise in the context.

- **Configuration over code** — Everything here is JSON config, not custom scripts
- **Measure everything** — Every number in this guide was measured on real sessions
- **Safety first** — Automation doesn't mean recklessness
- **Tool-agnostic principles** — The categorization method works for any LLM coding agent

---

## Contributing

Found a better compression tool? A smarter skill classification? Open an issue or PR.

---

## Sponsor

<p align="center">
  <b>If this guide saved you tokens, buy me a coffee ☕</b>
</p>

<p align="center">
  <table>
    <tr>
      <td align="center"><b>WeChat</b></td>
      <td align="center"><b>Alipay</b></td>
    </tr>
    <tr>
      <td align="center">
        <img src="images/wechat-qr.jpg" width="200" alt="WeChat Pay"><br>
        <sub>WeChat Pay</sub>
      </td>
      <td align="center">
        <img src="images/alipay-qr.jpg" width="200" alt="Alipay"><br>
        <sub>Alipay</sub>
      </td>
    </tr>
  </table>
</p>

> **NOTE:** Replace `images/wechat-qr.jpg` and `images/alipay-qr.jpg` with your actual payment QR code images. The placeholder files in this repo are for reference only.

---

## License

MIT © 2026 — Share it. Improve it. Ship it.

---

## Star History

If this helps you, [star the repo](https://github.com/tomerose/claude-code-token-optimizer) — it helps others find it too.

<p align="center">
  <sub>Made with 🔧 by a developer who got tired of watching tokens burn</sub>
</p>
