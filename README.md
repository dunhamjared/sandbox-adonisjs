# Sandbox for AdonisJS

This repo has many branches testing different versions, issues, bugs, and features related to AdonisJS.

This branch (`assembler-filesystem-race-condition`) documents and reproduces a filesystem race condition in `@adonisjs/assembler` (`copyFiles` during `node ace build`).

### Environment
- Branch: `assembler-filesystem-race-condition`
- `@adonisjs/assembler`: `>= 8.5.2` (tested on `8.6.0`)
- Node.js: `24.x`
- Filesystem: Host <--> VM virtualized mounts (Docker Desktop / DevContainers on macOS or Windows with VirtioFS, gRPC-FUSE, or WSL2. Will not reproduce on native host filesystems or native Linux Docker mounts)

### Steps to Reproduce

Run from the repository root:

```bash
git checkout assembler-filesystem-race-condition

docker run --rm -v "$(pwd)/repro-enotdir":/app -v /app/node_modules -w /app node:24 sh -c "
npm install &&
for i in \$(seq 1 30); do
  echo \"Run \$i\"
  node ace build || exit 1
done
"
```

*(Or inside a DevContainer / WSL: `cd repro-enotdir && npm i && for i in $(seq 1 30); do node ace build || break; done`)*

#### Expected Failure (usually within ~10 runs)
```text
ℹ Error: ENOTDIR: not a directory, mkdir '/app/build/resources'
 ⁃ at async mkdir (node:internal/fs/promises:1571:10)
 ⁃ (async file://node_modules/@adonisjs/assembler/build/utils-QbfbfSOx.js:991:3)
 ⁃ at async copyFiles (node_modules/@adonisjs/assembler/build/utils-QbfbfSOx.js:994:9)
```

### Problem & Possible Fix
In `@adonisjs/assembler`, `copyFiles` uses `filePaths.map` + `Promise.all` calling `mkdir(dirname(dest), { recursive: true })` concurrently for every file sharing the destination directory tree. On virtualized filesystem drivers, concurrent lookups/creations trigger `ENOTDIR` before the directory inode finishes linking.

Pre-creating unique target directories sequentially before `Promise.all(copyPromises)` eliminates the collision.

---

### AI Disclosure

**Human Note (Jared):**  
I found this issue while building within a monorepo in a vscode devcontainer. I did some debugging myself, but then used AI to help run tests and find the root cause.  

**AI-Generated Statement:**  
AI was used to help create this reproduction repository, isolate the timing-dependent race condition, and test it across native and virtualized (Docker/VirtioFS) filesystems.
