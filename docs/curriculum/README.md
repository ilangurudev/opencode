# Coding Agent Architecture Curriculum

A 2-week curriculum for learning how coding agents work by studying opencode's architecture and building your own.

## Viewing the Curriculum

### Option 1: GitHub Pages (Recommended)

1. Go to your repo's **Settings > Pages**
2. Under "Source", select **Deploy from a branch**
3. Select the branch (e.g., `main` or `claude/coding-agent-curriculum-O6xMm`)
4. Set the folder to `/docs/curriculum`
5. Click **Save**

Your curriculum will be available at:
```
https://<username>.github.io/<repo>/curriculum/
```

### Option 2: Local Development

Run a local server in this directory:

```bash
# Using Python
cd docs/curriculum
python -m http.server 3000

# Using Node.js
npx serve docs/curriculum

# Using docsify-cli
npm i -g docsify-cli
docsify serve docs/curriculum
```

Then open http://localhost:3000

### Option 3: Read on GitHub

The markdown files render directly on GitHub. Start with [index.md](index.md).

## Curriculum Structure

```
curriculum/
├── index.html          # Docsify app (for GitHub Pages)
├── index.md            # Curriculum overview
├── _sidebar.md         # Navigation sidebar
├── day-*.md            # Daily lessons
├── assignments/        # Hands-on Python assignments
└── deep-dives/         # Advanced topics
```

## Features

- **Search**: Press `/` to search the entire curriculum
- **Navigation**: Sidebar with nested structure
- **Syntax Highlighting**: Python, TypeScript, bash, JSON
- **Copy Code**: Click to copy code blocks
- **Dark Mode**: Respects system preference
- **Mobile Friendly**: Responsive design
