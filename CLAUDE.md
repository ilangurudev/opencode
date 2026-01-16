# CLAUDE.md - Context for Future Claude Instances

> **Internal guide for Claude Code: Understanding the coding agent curriculum in the `docs/` directory**

## Purpose of This File

This file provides context for future Claude instances working on this repository. It explains:
1. **What** the curriculum is and why it exists
2. **Where** the curriculum files are located
3. **How** the documentation is organized and deployed
4. **What** to be careful about when making changes

## What Is This Curriculum?

This repository contains a **2-week coding agent architecture curriculum** in the `docs/` directory. It was created to teach developers how production coding agents (like opencode itself) work.

### Origin Story
The user requested:
- "I use Claude Code regularly but have never looked under the hood"
- "I understand basic agent harness (LLM in a loop with tools) but don't know what the orchestration layer means"
- "Create a 1-2 week curriculum with bite-sized Python assignments that teaches me how coding agents work **in the context of opencode specifically**"
- "Not generic - I want it customized to opencode so I can look into the code itself"

The result: A comprehensive curriculum that teaches coding agent architecture by studying opencode's TypeScript implementation while students build their own agent in Python.

## Directory Structure

```
docs/
├── index.html                    # Main Docsify app (GitHub Pages entry point)
├── .nojekyll                     # Tells GitHub Pages not to use Jekyll
└── curriculum/                   # All curriculum content
    ├── index.md                  # Curriculum overview (homepage content)
    ├── _sidebar.md               # Navigation sidebar (IMPORTANT: uses absolute paths)
    ├── README.md                 # Setup instructions for viewing locally
    │
    ├── day-01-02-overview.md     # Daily lessons (core concepts)
    ├── day-03-04-main-loop.md
    ├── day-05-06-tools.md
    ├── day-07-08-agents.md
    ├── day-09-10-memory.md
    ├── day-11-12-extensibility.md
    ├── day-13-14-capstone.md
    │
    ├── assignments/              # Hands-on Python coding assignments
    │   ├── 01-minimal-agent.md
    │   ├── 02-streaming-agent.md
    │   ├── 03-file-tools.md
    │   ├── 04-permissions.md
    │   ├── 05-memory.md
    │   ├── 06-extensibility.md
    │   └── 07-capstone.md
    │
    └── deep-dives/               # Advanced optional topics
        ├── architecture.md
        └── ...
```

## How It's Deployed

### GitHub Pages Configuration
- **Deployment**: GitHub Pages serves from the `docs/` folder
- **Entry Point**: `docs/index.html` (contains the full Docsify configuration)
- **Live URL**: https://ilangurudev.github.io/opencode/
- **Technology**: Docsify (a documentation site generator that runs in the browser)

### Important Docsify Configuration Details

In `docs/index.html`, the key configuration is:

```javascript
window.$docsify = {
  basePath: '/opencode/curriculum/',  // Load markdown from curriculum subdirectory
  loadSidebar: true,                  // Load _sidebar.md for navigation
  // ... other config
}
```

This means:
- The HTML lives in `docs/index.html`
- The markdown content lives in `docs/curriculum/`
- Docsify loads content from `curriculum/` subfolder via the `basePath` setting
- URLs are clean: `https://ilangurudev.github.io/opencode/#/day-01-02-overview` (no `/curriculum/` in URL)

## Critical Information for Making Changes

### 1. Sidebar Links MUST Use Absolute Paths

In `docs/curriculum/_sidebar.md`, all links use absolute paths starting with `/`:

```markdown
✅ CORRECT:
* [Day 1-2: Overview](/day-01-02-overview.md)
* [Assignment 1](/assignments/01-minimal-agent.md)

❌ WRONG:
* [Day 1-2: Overview](day-01-02-overview.md)
* [Assignment 1](assignments/01-minimal-agent.md)
```

**Why?** Docsify resolves relative paths from the current page location. Absolute paths ensure links work from any page.

**Recent Fix:** We fixed a routing bug where sidebar links only worked from the homepage but broke when navigating to other pages. The fix was changing all relative paths to absolute paths.

### 2. The basePath Configuration

The `basePath: '/opencode/curriculum/'` in `docs/index.html` tells Docsify:
- Repository name: `opencode` (matches GitHub Pages URL structure)
- Content location: `curriculum/` subdirectory

**If the repository is renamed or forked**, update the basePath to match the new repo name:
```javascript
basePath: '/new-repo-name/curriculum/',
```

### 3. Don't Break the Build

When editing curriculum content:
- ✅ Edit markdown files in `docs/curriculum/`
- ✅ Keep the Docsify config in `docs/index.html`
- ✅ Use absolute paths in `_sidebar.md`
- ❌ Don't move `docs/index.html` to a subdirectory
- ❌ Don't change relative to absolute path references without understanding implications
- ❌ Don't remove `.nojekyll` file (GitHub Pages needs it)

### 4. Testing Changes

To test curriculum changes locally:

```bash
# Option 1: Simple HTTP server
cd docs
python -m http.server 3000
# Visit http://localhost:3000

# Option 2: Docsify CLI (recommended)
npm i -g docsify-cli
docsify serve docs
# Visit http://localhost:3000
```

## What the Curriculum Teaches

The curriculum is organized as a **2-week progressive learning path**:

### Week 1: Understanding the Architecture
- How coding agents differ from chatbots
- The main loop (user → LLM → tool → repeat)
- Tool system design and execution
- Students build basic agent with file and bash tools

### Week 2: Making It Real
- Permission systems (ask/allow/deny)
- Context and memory management
- Extensibility (skills, MCP, hooks)
- Capstone: Complete functional agent

### Teaching Approach
- **Core concepts** (~20-30 min reading): Explain architecture with diagrams
- **Assignments** (~30-60 min coding): Bite-sized Python implementation tasks
- **Deep dives** (optional): Advanced topics for those wanting more detail
- **Learn from opencode**: Study TypeScript implementation, build in Python

## Common Tasks You Might Be Asked

### Adding a new lesson
1. Create `docs/curriculum/day-XX-YY-topic.md`
2. Add entry to `docs/curriculum/_sidebar.md` with absolute path: `/day-XX-YY-topic.md`
3. Follow existing lesson structure (Goal blockquote, core concepts, code examples)

### Adding a new assignment
1. Create `docs/curriculum/assignments/0X-name.md`
2. Add link in `_sidebar.md` nested under the relevant day: `/assignments/0X-name.md`
3. Include clear objectives, starter code, and success criteria

### Fixing navigation issues
1. Check `docs/curriculum/_sidebar.md` - ensure all paths start with `/`
2. Verify `basePath` in `docs/index.html` matches repo name
3. Test locally before pushing

### Updating styling or Docsify config
- Edit `docs/index.html` (contains both HTML and Docsify config)
- The `<style>` block contains custom CSS
- The `window.$docsify` object contains all Docsify settings

## Git Workflow for Curriculum Changes

When making changes to the curriculum:

```bash
# Make your changes to files in docs/curriculum/
git add docs/

# Commit with clear message
git commit -m "docs: [describe what you changed]"

# Push to appropriate branch
git push -u origin <branch-name>
```

GitHub Pages will automatically rebuild when changes are merged to the deployment branch.

---

**Remember:** This curriculum is the user's learning resource for understanding how opencode (and coding agents in general) work. Maintain quality, clarity, and accuracy when making changes.
