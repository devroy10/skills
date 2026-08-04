---
name: capture-learning
description: "Capture a notable discovery, lesson, or answered question from the current session into a structured learning document. Use when the user says 'capture this', 'save this learning', 'write this up', 'add this to learnings', 'create a learning note', 'document what we just learned', 'log this insight', or otherwise signals that something learned in this session is worth preserving for interviews or future reference. Fires on statements like 'I want to remember this', 'save this for later', 'note this down', or when a substantive Q&A produced new understanding."
user-invocable: true
---

# Capture Learning

Save a discovery from the current session as a structured learning document.

## When to Fire

Fires when the user signals that something learned in this session is worth
preserving. Triggers include:

- "capture this", "save this learning", "write this up"
- "add this to learnings", "create a learning note"
- "document what we just learned", "log this insight"
- "I want to remember this", "save this for later", "note this down"
- any substantive Q&A in the session that produced new understanding

## What to Capture

Extract the essence of the learning from the session context. Identify:

1. **Topic** - the core subject, in a few words.
2. **Question** - the user's question or the problem the discovery answers, verbatim when available.
3. **Answer** - the full explanation from the session, rewritten as a standalone document. Ground it in the actual code or concepts involved. Do not reference the assistant or the session. Do not say "in this session we". Write as if it is reference material.
4. **Practice questions** - 3 to 10 review questions that test understanding of the answer.
5. **Resources** - recommended books, blogs, GitHub repositories, and YouTube videos relevant to the topic. Only include real, well-known resources. Do not invent URLs or book titles. When uncertain about a resource, search the web to verify it exists before including it.

## Steps

### Step 1: Determine the topic slug

- Derive a slug from the topic. Use lowercase, hyphen-separated words.
- Example: `throughput-load-capacity`.
- Completion criterion: a slug that identifies the learning uniquely.

### Step 2: Determine the repository name

- Read the git remote of the current project, or fall back to the project directory name.
- Example: `framexport`.
- Completion criterion: a repo name resolved from remote or directory.

### Step 3: Write the learning document

Create one markdown file with this structure:

```markdown
# Learning: <Topic Title>

Date: YYYY-MM-DD
Source: <repo name>

## Question

> <the question asked>

## Answer

<full standalone explanation, grounded in code/concepts>

## Practice Questions

1. <question>
2. <question>
...

## Recommended Resources

### Books

- **Title** by Author. One-line reason it is relevant.

### Blogs

- **Name** (URL). One-line description.

### GitHub Repositories

- **owner/repo** (URL). One-line description.

### YouTube Videos

- **Channel / creator** (URL). One-line description.
```

### Step 4: Write to both destinations

Write the identical content to:

1. `<project>/docs/learnings/<slug>.md` - the project-local copy.
2. `~/Documents/learnings/<repo-name>/<slug>.md` - the personal archive copy.

Ensure both directories exist before writing.

Completion criterion: the file exists at both paths with identical content.

### Step 5: Confirm

Report to the user:

- the topic captured,
- both file paths written,
- a one-line summary of the learning.

## Rules

- Do not modify any other files.
- Do not overwrite an existing learning file with the same slug. If one exists, append a numeric suffix (for example `-2`) or ask the user how to proceed.
- Keep the answer self-contained. It must make sense to someone who was not in the session.
- Never fabricate resources. Verify books, URLs, repos, and channels before including them.
- Use the date of the session (today) as the `Date` field.
