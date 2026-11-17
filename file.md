//save-hello-test/SKILL.md
---
name: save-hello-test
description: Save the text "Hello Test" into a file in the current directory or a directory the user names. Use when the user asks to save Hello Test, create the hello test file, or run the save-hello-test skill.
---

# Save "Hello Test" to a file

## Steps
1. **Pick the target directory.**
   - If the user names a directory, use it.
   - If not, use the current working directory (`pwd`).
2. **Pick the file name.**
   - Use the name the user gives.
   - If none is given, use `hello-test.txt`.
3. **Write the file** by running the bundled script:
   `bash scripts/save_hello.sh <directory> <filename>`
   The script creates the directory if needed and writes exactly `Hello Test` into the file.
4. **Don't overwrite silently.** If the file already exists, ask the user whether to overwrite it before running the script with `--force`.
5. **Confirm.** Show the full path of the file and its content (`cat` it).









//save-hello-test/scripts/save_hello.sh
#!/usr/bin/env bash
# Usage: save_hello.sh [directory] [filename] [--force]
set -euo pipefail

DIR="${1:-.}"
FILE="${2:-hello-test.txt}"
FORCE="${3:-}"
TARGET="$DIR/$FILE"

mkdir -p "$DIR"

if [[ -e "$TARGET" && "$FORCE" != "--force" ]]; then
  echo "File already exists: $TARGET (rerun with --force to overwrite)" >&2
  exit 1
fi

printf 'Hello Test\n' > "$TARGET"
echo "Saved: $(cd "$DIR" && pwd)/$FILE"
