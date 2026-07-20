# Module 1 – Secure Pipeline Basics

## Step 1: Creating the app project

We created `labs/app-1` and ran `npm init -y` inside it.

- `npm init -y` creates a **package.json** — the manifest of a Node.js
  project. It lists the project's name, its runnable scripts, and (most
  importantly for security later) its **dependencies**.
- The `"scripts"` section holds named commands you run with `npm run <name>`.
  The default `test` script just prints an error — we will replace it with a
  real test later, because our pipeline will run `npm test`.

## Step 2: Installing Express and the dependency tree

We ran `npm install express` to get Express, the most widely used Node web
framework.

Key observation: we asked for **1** package, but npm installed **67**.
That's because Express depends on other packages, which depend on others —
this is called a **dependency tree**. Every one of those 67 packages is
code we now run, written by strangers. This is exactly why dependency
scanning (SCA) exists.

npm also printed `found 0 vulnerabilities` — npm automatically checks
installed packages against a database of known-vulnerable versions. That
was our first (automatic) security scan.

The install created two new things:

### package-lock.json → COMMIT to Git
Records the *exact* version of all 67 installed packages. Without it,
another machine (a teammate, or the pipeline) installing tomorrow might get
slightly different versions — and "slightly different" can mean "a version
with a security hole." The lock file makes installs reproducible.

### node_modules/ → NEVER commit to Git
The actual downloaded code of all packages — thousands of files.
We never commit it because:
- It's huge and bloats the repo.
- It's fully reproducible: `package.json` + `package-lock.json` +
  `npm install` recreates it identically.
- Committing it would sneak third-party code into our repo without review.

## Step 3: .gitignore

We created a `.gitignore` at the **repo root** (so it covers all future
labs). It lists paths Git must pretend don't exist:

```
node_modules/
.env
```

`.env` is the standard file where Node projects keep secrets (passwords,
API keys). We ignore it from day one because accidentally committing
secrets is the single most common leak in Git history. More on this in the
secrets-scanning stage.

## Step 4: Verifying, not assuming

`git status` only showed `labs/` collapsed as one line, which doesn't prove
node_modules is ignored. So we ran:

```
git status -uall
```

The `-uall` flag expands untracked folders and lists every file. Result:
`package.json` and `package-lock.json` were visible, `node_modules` was
completely absent → the ignore rule works.

**Lesson: always verify a control is working, don't assume it is.** This
mindset is the core of security work.

## Q&A / issues from this session

**Q: Why did npm install 67 packages when I asked for 1?**
Dependencies have their own dependencies (a tree). All of them become code
you run in production, which is why they must be scanned.

**Q: Why commit package-lock.json but not node_modules?**
The lock file is a small text list of exact versions (reproducibility).
node_modules is the giant downloaded result, always recreatable from the
lock file.

## Step 5: Switching to TypeScript

We chose TypeScript over plain JavaScript: it catches type errors at
compile time, before the code runs — the same "catch problems early"
philosophy as DevSecOps itself.

Node doesn't understand TypeScript natively, so a compile step (TS → JS)
is needed. Bonus: this gives our pipeline a real "build" stage.

Installed as devDependencies (`-D`):
- `typescript` — the compiler (tsc), turns .ts into .js
- `tsx` — runs .ts directly during development, no compile step while coding
- `@types/express`, `@types/node` — type definitions so the compiler
  understands Express and Node APIs

## Mistake: installed npm packages in the wrong folder

We ran `npm install` at the repo root instead of `labs/app-1`. npm installs
into whatever folder you're in, so it created a second package.json,
package-lock.json, and node_modules at the root.

Fix: from the root, `rm -rf node_modules package.json package-lock.json`,
then cd into `labs/app-1` and re-run the install.

Lesson: always check the current directory before npm commands.

## Security note: install scripts (supply-chain risk)

npm warned: `esbuild@0.28.1 (postinstall: node install.js)`. Some packages
run a script automatically DURING installation. This is a real supply-chain
attack vector: a malicious package can execute code the moment it's
installed, before you run anything.

Our npm blocks un-approved install scripts and asks first — a security
control doing its job. We left it blocked (tsx works without it); revisit
only if something breaks.

## Concept: an app's two "moments" and dependency types

An app's code has two different moments in its life:

- **Moment 1 — building** (my laptop, or the pipeline machine): compiling
  TypeScript, running tests and linters. Needs the power tools.
- **Moment 2 — running** (the production server): executing the
  already-compiled JavaScript for real users. Needs no build tools.

**The pipeline is NOT the production server.** The pipeline is a temporary
machine GitHub spins up on every push — an automated version of my laptop.
It compiles, tests, and scans, then gets thrown away. It installs ALL
dependencies. The production server only runs the finished output and only
installs `dependencies`.

Flow:
laptop (write .ts) → pipeline (compile/test/scan, needs all deps)
→ production (runs .js, only real dependencies)

### dependencies — needed to RUN
Delete it from the production server and the app breaks.
Examples: express (handles every request), pg (queries the database at
runtime), jsonwebtoken (verifies login tokens per request), dotenv (loads
secrets when the app starts).

### devDependencies — needed only to BUILD
The running app has no idea they exist.
Examples: typescript (compiler), eslint (code style), jest/vitest (tests),
prettier (formatting), @types/* (compiler-only type hints — they vanish
after compilation).

Tricky one: `@types/express` is a devDependency even though express itself
is a dependency. Types exist only for the compiler.

### peerDependencies — "I assume you already have this"
Not "install for me" but a compatibility declaration: a package saying it
will use YOUR copy of something. Example: a React component library
declares React as a peer dependency because the app must have exactly one
React. Mostly seen when consuming libraries.

### Why the split matters for security
Production installs use `npm install --omit=dev` → devDependencies are
deliberately excluded → fewer packages on the server → smaller attack
surface. Every extra package is extra code that could carry a
vulnerability.

## Q&A from this session

**Q: Is being on the pipeline the same as being on a server?**
No. The pipeline is a temporary build machine (Moment 1) — my laptop's
routine, automated. The production server (Moment 2) only receives and
runs the compiled output. Once the pipeline passes, production doesn't
need any build tools.