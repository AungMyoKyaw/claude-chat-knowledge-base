# AGENT.md

## Core Directive: Proactive Context7 Usage

**CRITICAL**: Always use the **context7 MCP tool** when working with third-party libraries, frameworks, or APIs. This ensures you get the most up-to-date documentation and best practices.

### When to Use Context7 (MANDATORY)

You MUST use context7 when:

- Querying about specific technology, library, framework, or API
- Need to verify function signatures, APIs, or configuration options
- Generating code using third-party dependencies
- Current documentation would enhance solution quality
- Looking up best practices for any technology

### Context7 Workflow

```
1. Identify technical subject
2. Call context7 resolve-library-id to find the library
3. Call context7 get-library-docs to retrieve documentation
4. Integrate findings into your solution
```

---

## Chat Save Structure

This repository stores useful chat responses in a **topic-based folder structure**.

### Structure Pattern

```
my-claude-chat/
├── {topic-name}/
│   └── YYYY-MM-DD-{description}.md
```

### Rules for Saving Chats

1. **Folder name**: Use kebab-case topic (e.g., `github-private-repos`, `sparkboard-chat`)
2. **File name**: Use `YYYY-MM-DD-{brief-description}.md`
3. **Consolidate docs**: Prefer one comprehensive doc over multiple small files
4. **Use frontmatter**: Add title and date at the top of each file

### Example

```markdown
# {Title}

**Date:** YYYY-MM-DD
**Topic:** {Topic name}

{Content goes here...}
```

---

## Working on This Project

1. **Read existing code first** - Never modify code you haven't read
2. **Use context7** for any technology-specific questions
3. **Follow patterns** already established in the codebase
4. **Keep solutions simple** - Avoid over-engineering
5. **Save useful chats** - When user asks to save, create topic folder following structure above

---

## Temporary Workspace: `playground/`

The `playground/` folder is **git-ignored** and exists specifically for temporary/experimental work.

### When to Use Playground

Use `playground/` for:

- **Running scripts** - Test scripts, one-off commands, data processing
- **Experiments** - Try new approaches without committing
- **Temporary files** - Cache, downloads, intermediate outputs
- **Prototypes** - Quick drafts before moving to proper location

### Rules

1. **Never commit** anything from `playground/` - it's git-ignored
2. **Clean up periodically** - Delete old files when done
3. **No important work** - Only temporary/scratch work goes here
4. **Feel free to be messy** - This is the dirty work zone

### Other Git-Ignored Folders

- `tmp/` - General temporary files
- `sandbox/` - Experimental workspace
- See `.gitignore` for complete list
