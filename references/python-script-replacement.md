# Python Script Replacement (Windows/Git-Bash)

When the `patch` tool's `old_string`/`new_string` escaping fails (e.g. multi-line template literals, backtick-quoted JS strings with embedded single/double quotes, Chinese characters, or `\u` unicode escapes), use a Python script as an alternative.

## When to Use

- `patch` returns `"Escape-drift detected"` — the file content has quotes/backslashes that don't survive serialization
- The target block contains template literals (`` `${...}` ``), nested quotes, Chinese punctuation, or `\u` Unicode escapes
- The replacement is a block of 5-30 lines (single-string find-and-replace, not complex transformation)

## How It Works

Write a `.py` file with `skill_manage(action='write_file', ...)` or `write_file`, then execute with `python script.py` from the terminal.

## Template

```python
#!/usr/bin/env python
import os

filepath = 'D:/path/to/index.html'

with open(filepath, 'r', encoding='utf-8') as f:
    content = f.read()

old = """EXACT_TEXT_TO_REPLACE
LINE2
LINE3"""

new = """NEW_TEXT_REPLACEMENT
LINE2
LINE3"""

if old in content:
    content = content.replace(old, new)
    print("✓ Replaced successfully")
else:
    print("✗ Could not find old block!")
    idx = content.find('UNIQUE_ANCHOR')
    if idx >= 0:
        print(f"  Anchor found at idx {idx}")
        print(repr(content[idx:idx+300]))

with open(filepath, 'w', encoding='utf-8') as f:
    f.write(content)
print("Done")
```

## Key Differences from Node.js Approach

| Aspect | Node.js | Python |
|--------|---------|--------|
| Pre-installed on Windows | ❌ (requires install) | ✅ (via `python` alias) |
| Unicode handling | Good | Excellent (native UTF-8) |
| Multi-line strings | Template literals (`` ` ``) | Triple-quotes (`"""`) |
| Escape sensitivity | Lower | Lower (standard Python) |
| File write | `fs.writeFileSync` | Built-in `open()` |

## Verification After Patch

```bash
# Check no duplicate IDs
grep -c 'id="myElement"' /d/path/to/index.html
# Should be 1

# Check JS brace balance (rough)
python -c "
with open('index.html','r',encoding='utf-8') as f:
    js = f.read()
opens = js.count('{')
closes = js.count('}')
print('Braces:', 'OK' if opens == closes else f'FAIL ({opens} open, {closes} close)')
"

# Check key features present
grep -c 'myFunctionName\|myNewElement\|myDataKey' /d/path/to/index.html
```

## Limitations

- Python `replace()` is a **single pass** — if the old text appears multiple times, it replaces ALL occurrences. Use `replace(old, new, 1)` for single-replacement.
- Multi-pass patching with `replace()` can corrupt if later passes match against newly inserted text. Prefer writing one complete script per batch of related changes.
- The file is fully rewritten on every run — if the script crashes mid-write, the file content is lost. Keep a `.bak` copy.

## Pitfall: Exit Code 49 on Windows

If `python3 script.py` gives exit code 49 ("Python was not found"), use `python script.py` instead — Windows aliases `python3` inconsistently.
