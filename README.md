# Sandbox for AdonisJS

This repo has many branches testing different versions, issues, bugs, and features related to AdonisJS.

## Branch: `assembler-filesystem-race-condition`

This branch documents and reproduces a filesystem race condition in `@adonisjs/assembler` (`copyFiles` during `node ace build`).

### Reproduction Environment
- `@adonisjs/assembler`: `>= 8.5.2` / tested on `8.6.0`
- Node.js: `24.x`
- Filesystem: Bind-mounted / virtualized filesystems (Docker Desktop on macOS with VirtioFS, DevContainers, WSL2, 9p, gRPC-FUSE).

### Quick Repro with Docker
```bash
docker run --rm -v "$(pwd)/repro-enotdir":/app -v /app/node_modules -w /app node:24 sh -c "
npm install &&
for i in \$(seq 1 30); do
  echo \"Run \$i\"
  node ace build || exit 1
done
"
```

### Problem & Fix
In `@adonisjs/assembler`, `copyFiles` uses `filePaths.map` + `Promise.all` calling `mkdir(dirname(dest), { recursive: true })` concurrently for every file sharing the destination directory tree. On virtualized filesystem drivers, concurrent directory lookups and creations trigger `ENOTDIR` before the directory inode finishes linking.

Pre-creating unique target directories sequentially before running `Promise.all(copyPromises)` eliminates the collision.

---

### AI Disclosure

**Why AI was used:**
AI was used to accelerate reproducing, testing, and isolating this filesystem race condition across host and virtualized/bind-mounted environments. Because filesystem race conditions are timing- and environment-dependent, automated tooling helped rapidly validate whether the issue reproduces on native filesystems versus virtualized bind mounts (Docker/VirtioFS) and stress-test candidate solutions across repeated build loops.

**How AI was used:**
- Bootstrapped and configured the minimal AdonisJS reproduction project (`repro-enotdir`) with the exact `metaFiles` configuration.
- Set up and executed isolated container build loops to reproduce the `ENOTDIR` / `ENOENT` collisions under Docker Desktop's VirtioFS.
- Inspected `@adonisjs/assembler`'s `copyFiles` implementation, tested a deduplicated sequential directory creation patch across 30 build cycles, and reverted it back to reproduce the bug.
- Generated documentation and reproduction instructions for this repository.
