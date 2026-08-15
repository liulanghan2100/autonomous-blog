---
title: From Prompting to Production: What agent-skills Actually Teaches Your AI Coding Agent
topic: addyosmani/agent-skills: Production-grade engineering skills for AI coding agents.
source: https://github.com/addyosmani/agent-skills
generated_at: 2026-08-15T11:33:44.058365+08:00
ai_generated: true
---

# From Prompting to Production: What agent-skills Actually Teaches Your AI Coding Agent

I’ve spent the last two years watching teams paste “You are a senior engineer” into system prompts and then wonder why their AI still ships code that looks like a junior’s first PR. The problem isn’t the model—it’s the *context*. Most agents operate with zero knowledge of your repo’s conventions, your deployment pipeline, or even the difference between a hotfix and a feature branch. That’s where addyosmani/agent-skills comes in, and honestly, it’s the first thing I’ve seen that treats agent behavior like a codebase, not a vibe.

## The Pain Point: Your Agent is a Smart Intern with Amnesia

Picture this: You ask your agent to “add a retry logic to the payment service.” It writes 200 lines of beautiful TypeScript. Then you run `npm test` and watch it fail because your project uses `vitest`, not `jest`, and your CI requires 100% coverage on new code. The agent never knew because nobody told it. You *could* dump your entire `CONTRIBUTING.md` into every prompt, but that’s 3,000 tokens of noise that will dilute the actual task.

I’ve seen teams solve this by creating massive “project context” files. They work for a week, then rot because nobody updates them. The core issue: **skills are not documentation—they are executable procedures.** And most repos don’t have any.

## What agent-skills Actually Is

[addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) is a curated collection of **production-grade Markdown files** that teach AI coding agents (Claude Code, Cursor, Windsurf, etc.) how to behave in specific engineering contexts. Think of it as a “standard library” for agent behavior. It’s not a framework, not a plugin—it’s a set of **skill definitions** that you drop into your project’s `.agent/` or `.claude/` directory.

The repo covers things like:
- **Code review skills** (how to actually review a PR, not just check for syntax errors)
- **Testing skills** (how to write tests that match your project’s existing patterns)
- **Refactoring skills** (how to break a monolith without breaking the build)
- **Security audit skills** (how to look for OWASP Top 10 issues in code, not just scan with a tool)

Each skill is a Markdown file with a specific frontmatter structure (name, description, triggers) and a body that contains step-by-step instructions, checklists, and examples. The magic is in the **trigger conditions**—the skill only activates when the agent detects a matching context (e.g., “user mentions ‘refactor’ or ‘clean up’”).

## Why This Matters More Than Better Prompts

Let me be blunt: **prompt engineering is a dead end for production code.** You can’t prompt your way out of missing domain knowledge. Skills are different because they are **structured, versionable, and testable**. You can commit them to git, review them in a PR, and update them when your CI pipeline changes.

Here’s a concrete example. In my current project, we use a custom error handling pattern (`Result<T, Err>`). Without a skill, the agent would happily write `throw new Error()`. With a skill file that says:

```
## Error Handling
- ALWAYS use Result<T, Err> for fallible functions.
- NEVER throw exceptions in business logic.
- Log errors with the `logger.error` method, including the `request_id` from context.
```

…the agent produces code that actually passes review. This isn’t magic—it’s **constraint setting** at the right layer.

## Getting Started: My 30-Minute Setup

Here’s the exact workflow I used with Claude Code (works similarly for Cursor):

### 1. Clone and Copy the Relevant Skills

```bash
git clone https://github.com/addyosmani/agent-skills.git
cd agent-skills
# Copy the ones that match your stack
cp skills/code-review/ ~/my-project/.claude/skills/
cp skills/testing/ ~/my-project/.claude/skills/
cp skills/security/ ~/my-project/.claude/skills/
```

**Important:** Don’t copy everything. The repo has ~30 skills. If you load all of them, your agent will have decision paralysis. Pick 3-5 that match your biggest pain points.

### 2. Customize the Frontmatter

Each skill file starts with something like:

```yaml
---
name: code-review
description: Use when reviewing pull requests or changes for correctness, security, and maintainability.
triggers:
  - "review"
  - "pr"
  - "code review"
---
```

Change the `triggers` to match your team’s vocabulary. If your team says “check my work,” add that. If they say “inspect,” add that. The agent uses these to decide when to load the skill.

### 3. Test with a Minimal Example

Don’t test on your real codebase first. Create a tiny test repo with a known bug (e.g., a missing null check). Ask your agent to “review the code.” Watch if it picks up the skill. If not, check the trigger words.

## The Hidden Gem: Writing Your Own Skills

The real power isn’t the pre-built skills—it’s the **pattern**. After a week of using the repo, I started writing my own. Here’s the template I use:

```markdown
---
name: api-contract
description: Use when modifying or adding REST endpoints.
triggers:
  - "endpoint"
  - "API"
  - "route"
  - "REST"
---

## Steps
1. Check `openapi.yaml` for existing endpoint patterns.
2. If adding a new endpoint, follow the naming convention `/{resource}/{id}`.
3. Always return a structured error response: `{ "error": { "code": "...", "message": "..." } }`.
4. Update the OpenAPI spec in the same PR.
5. Add at least one integration test using the existing `test/integration/` harness.

## Examples
### Bad
```javascript
app.get('/users', (req, res) => {
  res.send(users); // no pagination, no error handling
});
```

### Good
```javascript
app.get('/users', (req, res) => {
  const { page = 1, limit = 20 } = req.query;
  // ... pagination logic
  res.json({ data: paginatedUsers, meta: { page, limit } });
});
```
```

The key insight: **write skills like you’re teaching a junior engineer, not like you’re writing documentation.** Include the “why” and the anti-patterns. Your agent doesn’t need to know *what* the code does—it needs to know *how* your team expects it to look.

## What I’ve Learned (The Hard Way)

**Lesson 1: Skills are not a replacement for CI.** I initially thought skills would catch all the “forgot to run the linter” issues. They don’t. Skills influence *how* the agent writes code, but they don’t enforce it. Your CI pipeline is still the final gate. Treat skills as a way to reduce CI failures, not eliminate them.

**Lesson 2: Trigger words are fragile.** If you write a skill for “refactoring” but your team says “restructure,” the agent won’t load it. I’ve started adding multiple trigger phrases, plus a catch-all in the system prompt: “If you’re not sure, check the skills directory.”

**Lesson 3: Version control your skills.** I’ve seen teams lose a month of skill tuning because they didn’t commit the `.agent/` directory. Add it to git. Review changes to skills just like you review code changes. A bad skill update can silently degrade your agent’s output for weeks before you notice.

**Lesson 4: The pre-built skills are opinionated.** Addy’s repo reflects his preferences (e.g., he prefers `vitest` over `jest`, values type safety highly). Don’t blindly trust them—read each skill and adjust to your stack. I had to rewrite the testing skill entirely because we use `mocha` and a custom assertion library.

## A Concrete Workflow That Works

Here’s what I’ve settled on after two months of use:

1. **Onboarding:** When a new dev joins, they clone the repo and run a setup script that copies the base skills into their local `.agent/` folder.
2. **Daily:** Before starting a task, the agent loads the relevant skill (e.g., “testing” if you’re writing tests). This happens automatically based on trigger words.
3. **PR Time:** The agent uses the `code-review` skill to self-review its own changes before submitting. This catches about 60% of the issues my human reviewers used to flag.
4. **Monthly Review:** We spend 30 minutes reviewing skill updates. If a skill caused a bug, we fix the skill, not the code.

## The Bottom Line

If you’ve been frustrated