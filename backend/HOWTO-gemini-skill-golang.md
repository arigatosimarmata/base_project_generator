# HOWTO: Gemini AI Skill — Go Clean Architecture

This document explains how the `go-clean-architecture` skill is structured,
registered, and invoked in Gemini CLI and the Antigravity IDE for Golang projects.

---

## 1. What Is a Skill?

A **Skill** in Gemini/Antigravity is a folder of instructions that extends the AI's
capabilities for a specific technical domain. When invoked, the AI reads the
`SKILL.md` file inside the folder and follows its instructions precisely.

Skills are **global** — they are registered once and available across all projects
without needing to be copied into each project repository.

```
Skill = a named folder + a SKILL.md instruction file
```

---

## 2. File Location

```
~/.gemini/config/skills/
└── go-clean-architecture/          ← skill folder name = invocation name
    └── SKILL.md                    ← instruction file read by the AI agent
```

> The folder name defines the slash command: `/go-clean-architecture`

---

## 3. How to Register This Skill in Antigravity IDE

### Step 1 — Locate the skills config directory
```bash
ls ~/.gemini/config/skills/
```

### Step 2 — Create the skill folder (if it does not exist)
```bash
mkdir -p ~/.gemini/config/skills/go-clean-architecture
```

### Step 3 — Place the SKILL.md file
Copy or create the `SKILL.md` file inside the folder. The file must include
a YAML frontmatter block at the top:

```yaml
---
name: go-clean-architecture
description: <one-line description used by the IDE to show the skill>
---
```

The `description` field is what appears in the Antigravity skill list and
determines when the AI considers auto-applying the skill based on task context.

### Step 4 — Restart the Antigravity IDE session
Skills are loaded at session start. A restart is required for changes to take effect.

---

## 4. How to Invoke the Skill

### Method A — Explicit invocation via slash command
Type the following in the Antigravity chat input:
```
/go-clean-architecture
```
The IDE will notify the AI to read `SKILL.md` and apply its instructions to the
current task before generating any response.

### Method B — Automatic detection (AI-side)
When the skill is registered, its `name` and `description` are injected into the
AI agent's system context at the start of every session. The AI may proactively
read `SKILL.md` if it determines the skill is relevant to the current task.

> **Note**: Explicit invocation (Method A) is more reliable. Use it whenever
> you want guaranteed compliance with the Go Clean Architecture standards.

---

## 5. What Happens When the Skill Is Invoked

The AI reads `SKILL.md` and follows the **Document Reading Protocol** defined
inside it:

```
/go-clean-architecture
     │
     ▼
Read SKILL.md
     │
     ▼
Step 1: Read 📌 Quick Reference Index in clean_architecture_go.md
     │
     ▼
Step 2: Read Section 1–6 (Core Rules) — only if task requires it
     │
     ▼
Step 3: Read relevant Section 7.x based on "When to Read" column
     │
     ▼
Apply standards and write code
```

This three-step protocol ensures the AI loads only what is needed,
minimizing token consumption per task.

---

## 6. Source of Truth File

The skill points to:
```
/Users/a2275/Artech/base-project/backend/clean_architecture_go.md
```

This file is a **living document**. The AI agent reads it directly every time
the skill is invoked — it never relies on cached or assumed knowledge of its
contents. Any update to the source file is automatically reflected the next
time the skill is used.

---

## 7. How to Update the Skill

### To change the reading protocol or enforcement rules:
Edit `~/.gemini/config/skills/go-clean-architecture/SKILL.md` directly.

### To add new architectural knowledge:
Update the source of truth file:
```
backend/clean_architecture_go.md
```
Follow the Changelog format defined in **Section 0** of that document, then
execute the AI Agent Git Automation Protocol (stage → commit → push).

### To add a new section that requires a new entry in the Quick Reference Index:
1. Add the new section (7.x) to `clean_architecture_go.md`
2. Add a corresponding row to the `📌 Quick Reference Index` table
3. Add a Changelog entry in Section 0
4. Commit: `git add backend/clean_architecture_go.md && git commit -m "docs(arch): ..." && git push`

---

## 8. Adding This Skill to a New Project

This skill is **global** — no per-project setup is required for the skill itself.

However, for projects that use this architecture standard, it is recommended to
add a project-level `AGENTS.md` (or equivalent) that references the skill and
defines the reading protocol for AI agents that do not have the global skill registered:

```markdown
# Project Rules

**Source of Truth:** `backend/clean_architecture_go.md`

Follow the reading protocol defined in:
`~/.gemini/config/skills/go-clean-architecture/SKILL.md`

Do not call `/go-clean-architecture` manually if this AGENTS.md is already loaded —
doing so causes the source file to be read twice, wasting tokens.
```

---

## 9. Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `/go-clean-architecture` not recognized | Skill folder not registered or IDE not restarted | Verify folder exists at `~/.gemini/config/skills/go-clean-architecture/SKILL.md`, then restart IDE |
| AI reads the entire `clean_architecture_go.md` on every call | Old SKILL.md without the reading protocol | Update SKILL.md to the latest version with the 3-step protocol |
| AI ignores architecture standards | Skill not invoked | Explicitly type `/go-clean-architecture` before the task |
| Source file changes not reflected | AI using cached context | Skill always reads the file directly — restart session if context seems stale |
