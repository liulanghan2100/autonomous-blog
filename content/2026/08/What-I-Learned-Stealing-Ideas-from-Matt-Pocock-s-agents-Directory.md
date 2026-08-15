---
title: What I Learned Stealing Ideas from Matt Pocock’s `.agents` Directory
topic: mattpocock/skills: Skills for Real Engineers. Straight from my .agents directory.
source: https://github.com/mattpocock/skills
generated_at: 2026-08-15T11:35:04.188843+08:00
ai_generated: true
---

# What I Learned Stealing Ideas from Matt Pocock’s `.agents` Directory

If you’ve spent more than ten minutes on TypeScript Twitter, you know Matt Pocock. He’s the guy who made `zod` and TS generics feel approachable. But a few weeks ago, I stumbled onto something more interesting than his type gymnastics: a repo called `mattpocock/skills`, which is literally a dump of his `.agents` directory.

At first I thought it was a joke. Then I realized it’s a goldmine for anyone building AI-assisted coding workflows. This isn’t a “prompt engineering” fluff piece. This is about how a working engineer structures the instructions, context, and guardrails that an AI agent needs to actually ship code without wrecking your codebase.

Here’s what I learned, what I copied, and what I’d change.

## The Problem: Your AI Agent Is Only as Good as Your Defaults

Let me set the scene. You’ve got Cursor, or Claude Code, or some other agentic tool. You ask it to “refactor this function.” It does. Then you realize it:

- Renamed a public API that three other files depend on.
- Used a pattern your team explicitly banned six months ago.
- Wrote tests that mock everything so they pass but assert nothing.

Sound familiar? The root cause isn’t the model. It’s that you gave the agent zero context about *your* project’s conventions. Most people write a two-line system prompt and expect magic. Matt’s approach is different: he treats the agent like a junior engineer who needs a detailed onboarding doc, not a mind reader.

His `skills` repo is essentially a set of Markdown files that define, in explicit terms, how the agent should behave in specific situations. Think of it as a `CONTRIBUTING.md` for your AI pair programmer.

## What’s Actually in the Repo (Don’t Just Clone It)

I’m not going to paste the whole thing here—go read it yourself (link: `github.com/mattpocock/skills`). But structurally, it breaks down into a few key categories that matter.

### 1. Role and Tone Definitions

The first thing you’ll notice is that Matt doesn’t just say “you are a helpful assistant.” He defines the *specific persona* for a task. For example, a skill for writing tests might start with:

```markdown
You are a senior test engineer. You write tests that verify behavior, not implementation details. You prefer integration tests over unit tests when the tradeoff is reasonable. You never mock what you don't own.
```

That last line is gold. “Never mock what you don’t own” is a rule that prevents a whole class of brittle test bugs. Generic prompts don’t do that.

### 2. Explicit “Do Not” Lists

This is where most people fail. We tell the agent what to do, but we rarely tell it what to *stop* doing. Matt’s skills have explicit “Avoid” sections. For instance:

```markdown
- Do not use `any` in TypeScript unless absolutely necessary and commented.
- Do not introduce new dependencies without asking.
- Do not refactor code unrelated to the task at hand.
```

The third one is critical. Agents love to “clean up” things they see. That’s how you end up with a 400-line diff when you asked for a 10-line change.

### 3. Context Injection Patterns

The most practical takeaway isn’t the content of the files—it’s *how* they’re structured for injection. Matt’s skills are designed to be loaded into the agent’s context window at specific moments. He uses a pattern where each skill is a self-contained Markdown file with a clear filename like `write-typescript.md` or `review-pr.md`.

The trick is that each file starts with a “When to use this” section. This isn’t for the human; it’s for the agent’s routing logic. If you’re using a tool like Claude Code or Cursor’s rules, you can set up triggers that load the right skill when the conversation matches a certain pattern.

## How I Actually Implemented This (Code Included)

I’m not going to pretend I copied his repo verbatim. I took the *philosophy* and built a leaner version for my own project, which is a Node.js monorepo with a mix of TypeScript and some legacy JavaScript.

Here’s the structure I landed on:

```
.agents/
  skills/
    typescript.md
    testing.md
    git-workflow.md
    security-review.md
  rules/
    global.md
```

### Step 1: Write the Global Rules

This is your baseline. It loads every time. Mine looks like this (shortened for brevity):

```markdown
# Global Rules
- You are working in a Node.js monorepo using pnpm workspaces.
- TypeScript is the default. Do not write plain JS unless the file is in /legacy.
- Follow the existing code style. If you see 2-space indentation, keep it.
- Never run `git push`. Propose the command, let the human run it.
- If a task takes more than 5 steps, break it into sub-tasks and ask for confirmation.
```

The last rule is a lifesaver. It prevents the agent from going off on a 30-minute refactoring spree without checkpoints.

### Step 2: Write a Specific Skill (Testing Example)

Here’s the skill file I use for writing tests. This is the one that’s saved me the most pain:

```markdown
# Skill: Write Tests

## When to use
- User asks to add tests for a new feature.
- User asks to fix a failing test.
- User asks to increase coverage on a specific module.

## Rules
- Use Vitest. Do not use Jest.
- Tests must be colocated: `src/foo.ts` -> `src/foo.test.ts`.
- Name tests in the format: `describe('foo', () => { it('should do X', ...) })`.
- Never mock a module you don't own (e.g., `fs`, `http`). Use real filesystem in a temp dir.
- Assert on behavior, not implementation. Do not assert that a specific function was called unless it's a side-effect boundary.

## Example
Given a function `add(a, b)`, a good test:
```typescript
import { describe, it, expect } from 'vitest';
import { add } from './add';

describe('add', () => {
  it('adds two numbers', () => {
    expect(add(2, 3)).toBe(5);
  });
});
```

A bad test:
```typescript
// BAD: asserts on implementation detail
expect(add).toHaveBeenCalledTimes(1);
```
```

### Step 3: Wire It Up

If you’re using Cursor, you can put these in the `.cursor/rules` directory. If you’re using Claude Code, you can use the `CLAUDE.md` file and reference the skills. For a more manual approach, I use a small shell script that prepends the relevant skill to my prompt:

```bash
#!/bin/bash
# agent.sh - Load skill context
SKILL=$1
shift
cat ".agents/skills/${SKILL}.md" | xargs -0 -I{} claude -p "{} $*"
```

Not elegant, but it works. The point is: the skill file is a *unit of context*. You load it when needed, not always.

## The Hard Lessons (What Didn’t Work)

I’ve been running this for three weeks. Here are the real-world gotchas.

### Lesson 1: Skills Go Stale Fast

I wrote a skill for our API style guide. Two weeks later, we switched from REST to tRPC. The skill was now actively harmful because it kept telling the agent to use REST patterns. **You have to treat skills like code—they need version control and review.** I now have a rule: any skill that hasn’t been touched in 30 days gets flagged for review.

### Lesson 2: Too Much Context Is Worse Than Too Little

My first iteration had a 3000-word global rule file. The agent started ignoring it. It’s like onboarding a dev with a 50-page manual—they’ll skim it and miss the critical bits. Keep the global rules under 500 words. Put the details in specific skills.

### Lesson 3: The “Do Not” List Is Non-Negotiable

I forgot to add “Do not modify package.json” to my global rules. An agent decided to add a dependency to fix a linting issue. That dependency had a security vulnerability. The agent didn’t know. The skill didn’t tell it not to. **Your guardrails are your security boundary.**

## A Practical Template for Your Own `.agents` Directory

If you’re starting from scratch, don’t copy Matt’s repo wholesale. Here’s the minimal viable version:

1. **`global.md`** (under 500 words): Project structure, language rules, forbidden actions, and a rule to ask before touching `package.json` or CI configs.
2. **`code-review.md`**: Focus on security, not style. Tell the agent