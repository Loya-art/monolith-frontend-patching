# Node Script Replacement Pattern

> Alternative to the `patch` tool when modifying large blocks (50+ chars) in monolith HTML/JS files. Used when `patch` fails with escape-drift, literal `\n` serialization, or cannot find the exact match due to whitespace variation.

## When to use

- The `patch` tool returns "Could not find a match" for a block you know exists
- The replacement block is >20 lines
- The patch introduces literal `\n` characters at code boundaries (serialization artifact)
- You need to target a specific function or block by marker string, not by line numbers

## Pattern: `write_file` → `node -e` script → run → clean

```javascript
// 1. Write the replacement script to temp dir
const fs = require('fs');
const path = 'target/index.html';
let html = fs.readFileSync(path, 'utf8');

// 2. Find exact insertion point by marker string
const markerStart = 'function targetFunction() {';
let startIdx = html.indexOf(markerStart);
if (startIdx < 0) { console.error('marker not found'); process.exit(1); }

// 3. Find end by brace matching (for functions)
let depth = 0, endIdx = startIdx;
for (let i = startIdx; i < html.length; i++) {
  if (html[i] === '{') depth++;
  if (html[i] === '}') { depth--; if (depth === 0) { endIdx = i + 1; break; } }
}

// 4. Build new content
const newFunc = `function targetFunction() {
  // ... new implementation ...
}`;

// 5. Replace in place
const before = html.slice(0, startIdx);
const after = html.slice(endIdx);
fs.writeFileSync(path, before + newFunc + after);
console.log('Replaced successfully');
```

## Pattern: precise string replacement (for smaller blocks)

```javascript
const fs = require('fs');
let html = fs.readFileSync('index.html', 'utf8');

const oldText = 'exact old string to replace (must be unique)';
const newText = 'new replacement text';

if (html.includes(oldText)) {
  html = html.replace(oldText, newText);
  fs.writeFileSync('index.html', html);
  console.log('Replaced OK');
} else {
  console.error('String not found');
  process.exit(1);
}
```

## Why this works when patch doesn't

| Issue | `patch` tool | Node script |
|-------|-------------|-------------|
| Escape-drift (literal `\n` in output) | Common with multi-line replacements | Never — strings are real newlines |
| Whitespace sensitivity | 9 fuzzy matching strategies, can match wrong location | Exact string match, controlled |
| Block too large for tooling constraints | May be truncated | No limit |
| Replacing complex template literals | Backticks + quotes cause escape cascades | Direct string content |

## Safety checklist after replacement

1. `node --check file.js` (or the JS extracted from HTML)
2. Brace balance check (see main skill)
3. Verify the new content is actually present: `grep "newFunction" index.html`
4. Line count check: compare total lines before/after to confirm the block was replaced (not duplicated)

## Chinese path workaround

On Windows with Chinese usernames, `node /c/Users/<user>/path/to/script.js` fails with:
```
Error: Cannot find module 'E:\\c\\Users\\...'
```
The linter prefix `E:\c\` is a drive-letter corruption from cross-drive path translation.

**Fix**: Use `cd` to the directory first, then run with relative path:
```bash
cd /c/Users/<user>/AppData/Local/Temp
node script.js
```
OR use absolute path from the same drive:
```bash
cd /c/Users/<user>/AppData/Local/Temp
node C:/Users/<user>/AppData/Local/Temp/script.js
```
