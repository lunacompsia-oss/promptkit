# PromptKit -- Production-Ready Prompt Templates for AI Coding Assistants

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![GitHub Release](https://img.shields.io/github/v/release/M3phist0s/promptkit?include_prereleases&label=release)](https://github.com/M3phist0s/promptkit/releases)
[![GitHub Stars](https://img.shields.io/github/stars/M3phist0s/promptkit?style=social)](https://github.com/M3phist0s/promptkit)
[![AI Ready](https://img.shields.io/badge/AI%20Ready-Score%20Your%20Repo-purple?style=flat)](https://github.com/M3phist0s/ai-ready)

**Stop writing the same CLAUDE.md from scratch every project.** PromptKit gives you battle-tested `CLAUDE.md` templates, `.cursorrules` files, code review prompts, and step-by-step workflows that make Claude Code, Cursor, and Copilot dramatically more useful -- out of the box.

[Browse the Free Sample](#free-sample) -- [Get the Full Kit](https://m3phist0s.github.io/promptkit/) -- [Join the Discussion](https://github.com/M3phist0s/promptkit/discussions)

---

## The Problem

You start a new project. You open Claude Code or Cursor. The AI has no idea about your stack, your conventions, or your architecture. So it generates generic code that you spend an hour fixing.

Or you copy-paste a CLAUDE.md from your last project, tweak it for 30 minutes, and it still misses half the patterns you care about.

**Without PromptKit:**
- AI generates vanilla React with class components and default exports
- No awareness of your API patterns, auth setup, or error handling conventions
- Code reviews catch nothing because the prompt is "review this code"
- You build the same SaaS boilerplate by hand every single time

**With PromptKit:**
- Drop a `CLAUDE.md` into your repo and Claude Code immediately follows your architecture
- `.cursorrules` files make Cursor produce code that matches your team's patterns from the first keystroke
- Code review prompts catch security flaws, performance issues, race conditions, and code smells -- systematically
- A step-by-step workflow takes you from `mkdir` to a deployed SaaS with auth and billing in one session

---

## What's Inside

### CLAUDE.md Templates
Project-level instruction files that tell Claude Code exactly how your codebase works. Each template covers file structure, naming conventions, code style, architecture decisions, and common pitfalls for a specific project type.

| Template | What It Covers |
|----------|---------------|
| SaaS Application | React + TypeScript + Node, multi-tenant auth, Stripe billing, API design |
| REST API | Express/Fastify patterns, validation with Zod, error handling, rate limiting |
| CLI Tool | Argument parsing, output formatting, error codes, testable command structure |
| Frontend App | Component patterns, state management, routing, accessibility, performance |

### .cursorrules Files
Cursor rules that enforce your team's coding standards automatically. Every file covers component patterns, TypeScript strictness, import ordering, anti-patterns to avoid, and framework-specific idioms.

| Rules File | Stack |
|-----------|-------|
| React + TypeScript | Function components, hooks patterns, discriminated unions, named exports |
| Node.js API | Route handlers, middleware chains, service layer, database query patterns |
| Python + FastAPI | Pydantic models, dependency injection, async patterns, SQLAlchemy |
| Go | Package structure, error handling, interfaces, concurrency patterns |

### Prompt Collections
Copy-paste prompts for specific development tasks. Each prompt is engineered to produce thorough, actionable output from any AI coding assistant.

| Collection | Prompts | Covers |
|-----------|---------|--------|
| Code Review | 16 | Security, performance, concurrency, code smells, architecture, accessibility, API design, test quality, dependency audit, and more |
| Architecture | 15+ | System design, database modeling, API design, migration planning |
| Debugging | 15+ | Root cause analysis, performance profiling, memory leaks, race conditions |

### Workflow Recipes
Multi-step prompt sequences that walk you through an entire development task from start to finish.

| Workflow | Steps | Result |
|---------|-------|--------|
| Zero to Deployed SaaS | 7 phases | Empty directory to production app with auth, billing, and landing page |
| Legacy Modernization | Multi-phase | Assess, plan, and incrementally modernize a legacy codebase |
| Test Suite Generation | Multi-phase | Generate comprehensive test coverage for an existing project |

---

## Free Sample

Try before you buy. The free sample includes one file from each category so you can see the quality and depth.

| File | Description |
|------|-------------|
| [`CLAUDE-saas-sample.md`](free-sample/CLAUDE-saas-sample.md) | CLAUDE.md template for SaaS apps -- project structure, code style, auth patterns, billing integration |
| [`cursorrules-react-sample.md`](free-sample/cursorrules-react-sample.md) | Cursor rules for React + TypeScript -- component patterns, hooks, TypeScript strictness, anti-patterns |
| [`code-review-sample.md`](free-sample/code-review-sample.md) | 5 code review prompts -- security, performance, general PR review, concurrency, code smell detection |
| [`zero-to-saas-sample.md`](free-sample/zero-to-saas-sample.md) | First 2 phases of the Zero to SaaS workflow -- project scaffold + authentication setup |

### Quick Start (30 seconds)

**For Claude Code users:**
```bash
# Download the CLAUDE.md sample and drop it in your project root
curl -sL https://raw.githubusercontent.com/M3phist0s/promptkit/main/free-sample/CLAUDE-saas-sample.md -o CLAUDE.md

# Claude Code will now follow these conventions automatically
claude
```

**For Cursor users:**
```bash
# Download the .cursorrules sample
curl -sL https://raw.githubusercontent.com/M3phist0s/promptkit/main/free-sample/cursorrules-react-sample.md -o .cursorrules

# Cursor reads this file automatically -- start coding
```

**For any AI coding assistant:**
```
# Open code-review-sample.md, copy a prompt, paste it into your tool
# Works with Claude Code, Cursor, GitHub Copilot Chat, ChatGPT, Windsurf, or any LLM
```

---

## Pricing

| | Free | Starter | Pro | Team |
|---|---|---|---|---|
| **Price** | $0 | $19 | $39 | $99 |
| CLAUDE.md templates | 1 sample | 4 | 12 | 12 |
| .cursorrules files | 1 sample | 4 | 10 | 10 |
| Prompt collections | 5 prompts | 2 collections | All collections | All collections |
| Workflow recipes | 2 phases | 2 workflows | All workflows | All workflows |
| Cheat sheets | -- | -- | All | All |
| Team license | -- | -- | -- | Up to 10 devs |
| Updates | -- | 6 months | 1 year | 1 year |
| | [Download](https://github.com/M3phist0s/promptkit/releases) | [Buy](https://m3phist0s.github.io/promptkit/#pricing) | [Buy](https://m3phist0s.github.io/promptkit/#pricing) | [Buy](https://m3phist0s.github.io/promptkit/#pricing) |

All paid tiers include instant download and future updates for the included period.

[View full pricing and details](https://m3phist0s.github.io/promptkit/)

---

## Who This Is For

- **Solo developers** building SaaS products who want Claude Code or Cursor to understand their stack from day one
- **Team leads** who want consistent AI-assisted code across the team without writing a 20-page style guide
- **Developers learning prompt engineering** for AI coding tools and want proven templates to study and adapt
- **Anyone tired of generic AI output** who wants their assistant to produce code that actually fits their project

## How It Works

AI coding assistants like Claude Code and Cursor read project-level configuration files to understand your codebase. Claude Code reads `CLAUDE.md`. Cursor reads `.cursorrules`. The better these files are, the better the AI output.

PromptKit provides professionally written versions of these files, covering architecture, conventions, edge cases, and anti-patterns that take hundreds of hours of trial and error to get right.

The prompt collections and workflow recipes work with any tool -- paste them into Claude Code, Cursor, GitHub Copilot Chat, ChatGPT, or any LLM that accepts text input.

---

## FAQ

**Will these work with my specific stack?**
The CLAUDE.md and .cursorrules files cover the most common stacks (React, Node, Python, Go). The prompt collections and workflows are stack-agnostic. You can also use any template as a starting point and customize it for your exact setup.

**What is a CLAUDE.md file?**
A `CLAUDE.md` file is a project-level instruction file that Claude Code reads automatically when you open a project. It tells Claude about your file structure, coding conventions, architecture decisions, and patterns to follow. Think of it as a style guide that your AI pair programmer actually reads.

**What is a .cursorrules file?**
A `.cursorrules` file is the Cursor editor's equivalent -- a configuration file that tells Cursor's AI about your project's conventions, patterns, and constraints. It shapes every code suggestion and generation to match your standards.

**Can I customize the templates?**
Yes. Every template is a Markdown file. Fork it, edit it, make it yours. The templates are starting points designed to save you hours of writing from scratch.

**Do I get updates?**
Starter includes 6 months of updates. Pro and Team include 1 year. Updates include new templates, new prompt collections, and improvements to existing files.

---

## Contributing

PromptKit is a commercial product, but the free sample and this repository are open for community input.

- **Found a bug in a sample file?** Open an issue.
- **Have a suggestion for a new template?** Start a [Discussion](https://github.com/M3phist0s/promptkit/discussions).
- **Want to share how you use PromptKit?** Post in [Discussions -- Show and Tell](https://github.com/M3phist0s/promptkit/discussions).

We read every discussion post and use community feedback to prioritize new templates and collections.

---

## License

The free sample files in `free-sample/` are released under the [MIT License](LICENSE). Use them however you like.

The paid product files are licensed per-seat. See [pricing](https://m3phist0s.github.io/promptkit/#pricing) for details.

---

Built for developers who use AI coding assistants daily and want them to actually understand the project.

[Get the Free Sample](https://github.com/M3phist0s/promptkit/releases) -- [Browse the Full Kit](https://m3phist0s.github.io/promptkit/) -- [Star this repo](https://github.com/M3phist0s/promptkit)
