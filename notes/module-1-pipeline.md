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

## Step 6: tsconfig.json — the compiler's rulebook

The TypeScript compiler needs instructions before it can work:
1. Which files to compile? → `include: ["src"]` + `rootDir: "src"`
2. Where to put output? → `outDir: "dist"` (generated JS, kept separate
   from source; added `dist/` to .gitignore — like node_modules, it's
   reproducible and never committed)
3. What JS version to produce? → `target: "ES2022"` (modern Node
   understands it)
4. How strict? → `strict: true`

Why strict mode is non-negotiable for us: the compiler refuses code that
might use an undefined variable or an unverifiable type. Loose mode lets
these slide into runtime crashes/bugs — and some vulnerabilities start as
exactly these bugs. Strict mode is our first quality gate, before we even
have a pipeline.

tsconfig.json is saved in the repo so every machine (laptop, pipeline)
compiles identically — same reproducibility idea as package-lock.json,
but for the build.

Plumbing options: `esModuleInterop` (imports work smoothly with packages
like Express), `skipLibCheck` (skip re-checking library type files; faster
builds, still fully checks OUR code).

## Step 7: The server in TypeScript (src/index.ts)

- `import express, { Request, Response }` — TS import syntax; also pulls
  the type definitions from @types/express.
- `(req: Request, res: Response)` — typed handler params. Typos like
  `res.jsn()` now fail at compile time instead of crashing at runtime.
- `/` route — replies with JSON. req = incoming request, res = our reply.
- `/health` route — an "am I alive?" endpoint. Kubernetes will probe
  endpoints like this in Module 4.
- `app.listen(PORT)` — bind port 3000 and wait, like a process binding a
  socket.

## Step 8: npx and npm scripts

- `npx <tool>` runs a binary from local node_modules (node_modules/.bin)
  without global install.
- The `"scripts"` block in package.json gives named shortcuts that humans
  AND pipelines use. Inside scripts, npx isn't needed — npm checks
  node_modules/.bin automatically.

Our scripts:
- `npm run dev` → tsx runs the .ts directly (fast, dev only)
- `npm run build` → tsc compiles src/ to dist/ (silence = success)
- `npm start` → node runs the compiled dist/index.js (what production does)

The flow:
src/index.ts → npm run build (tsc) → dist/index.js → npm start (node)
dev shortcut: npm run dev (tsx, skips compiling for speed)

We verified all three: dev server answered on /, /health returned
{"status":"ok"}, build produced dist/index.js, and npm start ran the
compiled output.

## Mistake: committed dist/ to Git

The commit output showed `create mode ... labs/app-1/dist/index.js` —
compiled output got committed because `dist/` was missing from .gitignore.

Fix:
1. Added `dist/` to the root .gitignore.
2. `.gitignore` only affects UNTRACKED files, so for the already-tracked
   file we ran `git rm -r --cached labs/app-1/dist` — this removes it from
   Git's tracking but keeps it on disk.
3. Committed the fix.

Lesson: read every commit's output. The file list is a mini security
review — it's how we caught this.

## Concept: GitHub Actions from zero

GitHub Actions is GitHub's built-in pipeline machine. You describe the
pipeline in a file at `.github/workflows/<name>.yml` — GitHub watches that
folder, and any .yml inside becomes a **workflow**. The pipeline is code,
versioned in the same repo it protects.

Three building blocks:
1. **Trigger (`on:`)** — WHEN to run (ours: on every push)
2. **Job** — a set of steps that runs on one fresh machine
3. **Steps** — the commands, in order (mirrors what I did by hand:
   checkout code, install Node, npm install, npm run build)

Mental model: a workflow file = "when X happens, spin up a machine and run
these commands." All security tools later (SAST, SCA, secrets) are just
more steps/jobs in this file.

Security seed: the runner executes whatever the workflow file says — so
anyone who can change that file can make GitHub's machines run arbitrary
commands. Pipeline files are a prime attack target.

"Actions" also means reusable pre-built steps from a marketplace (e.g.
actions/checkout). Third-party steps are a supply-chain risk, same idea as
npm packages — we'll scrutinize them later.

## Deep dive: what a runner actually is

**Physical or not?** GitHub (owned by Microsoft) runs datacenters of
physical servers. On each, a **hypervisor** slices the hardware into many
**virtual machines (VMs)** — complete fake computers with their own OS,
CPU share, RAM, and disk, fully isolated from each other. Like processes
isolated by a kernel, but one level deeper: whole OSes isolated by a
hypervisor.

**Resource allocation — never unlimited.** Same as running VMware on my
laptop: the VM gets a fixed slice of the host (some vCPUs, some GB RAM,
some disk) and sees only that slice. A **vCPU** is a share of physical
cores the hypervisor schedules for the VM. If a job exceeds its slice, it
fails. GitHub publishes current runner sizes (link below).

**A "runner"** = the VM + the runner agent (GitHub's open-source program
pre-installed in it) that talks to GitHub and executes jobs.

**The full flow of a run:**
1. git push
2. GitHub checks .github/workflows/ for matching triggers
3. Match → job created and queued
4. Control plane picks a pre-booted VM from a warm pool (fast starts)
5. Runner agent polls GitHub outbound over HTTPS: "any job for me?"
   (outbound-only = nothing needs to reach into the VM)
6. Agent receives the job (workflow steps as a to-do list)
7. Executes steps in order, streams logs live to the Actions tab
8. Job ends → the ENTIRE VM is destroyed. Not cleaned — destroyed.

**VM images:** each VM starts from a snapshot (e.g. `ubuntu-latest`:
Ubuntu preloaded with Git, Node, Python, Docker...). Every job gets an
identical fresh copy.

**Why destroy every time (security):**
- No leftovers: code, caches, secrets vanish with the VM
- No cross-contamination between jobs/users on shared hardware
- Reproducibility: every run starts from the identical image
This is why every run must npm install again — the VM has never seen our
project before.

**Who controls what:** GitHub/Microsoft control hardware, hypervisors,
images, queue, orchestration. I control only the workflow file — which is
exactly why it's the attack target. Alternative: **self-hosted runners**
(run the agent on your own machine; more control, but you own its
security).

## Q&A: re-running failed jobs when VMs are destroyed

The VM was never the thing being re-run — the **job definition** is.
GitHub's control plane keeps the job definition (workflow content at a
specific commit). "Re-run failed jobs" = same definition, back in the
queue, executed by a brand-new fresh VM from the start.

- Fixes flaky failures (network hiccup during npm install) because the
  retry is a clean attempt.
- Can never fix a real code bug — same code, same steps, same failure.

What survives the VM: logs (streamed out), results, and **artifacts** —
files a job explicitly uploads to GitHub before its VM dies (used to pass
things like scan reports between jobs).

Analogy: VM = process, job definition = program on disk. Killing the
process doesn't delete the program.

## Q&A: 4 stages = 4 VMs or 1?

Rule: **steps in one job share the same VM; each job gets its own fresh VM.**

- 1 job with 4 steps = 1 VM. Steps share disk/files (install creates
  node_modules, build uses it). Serial execution.
- 4 jobs = 4 VMs. They share nothing (each must checkout + install again;
  data passes only via artifacts) and run in PARALLEL by default. Order is
  declared with `needs:` (e.g. scan needs build).

Tradeoff: same VM = fast/simple/shared state but serial. Separate jobs =
parallel + isolated (a compromised scan step can't touch the build VM) but
each pays setup cost again.

Our first workflow: one job, few steps. Later, scanners split into
parallel jobs.

## Official docs
- GitHub Actions overview: https://docs.github.com/en/actions/get-started/understand-github-actions
- Workflow syntax reference: https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions
- GitHub-hosted runners (incl. current VM sizes): https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners
- Runner images (what's preinstalled): https://github.com/actions/runner-images
- Runner agent source code: https://github.com/actions/runner
- Self-hosted runners: https://docs.github.com/en/actions/hosting-your-own-runners

## Q&A: how can GitHub give away VMs for free?

It costs Microsoft real money — paid deliberately because:

1. **Free is metered, not unlimited.** Public repos get standard runners
   free; private repos get a monthly minutes quota before billing.
   Concurrency and job-time limits are capped, and abuse (e.g.
   crypto-mining on runners) is detected and banned.
2. **Funnel:** individuals learn Actions free → they join companies →
   companies pay for Enterprise plans, bigger runners, extra minutes.
3. **Lock-in:** CI is sticky infrastructure — migrating pipelines is
   painful, so free CI keeps users (and later their employers) on GitHub.
4. **Open-source strategy:** free public-repo CI makes GitHub the home of
   open source — worth more in positioning than the compute costs.
5. **Scale economics:** Microsoft owns Azure datacenters; a short-lived VM
   on already-running hardware costs cents. Warm pools + high utilization
   keep it cheap.

Takeaway for this lab: our public repo runs free standard runners with
generous limits — plenty for 6 months.

## Step 9: The first workflow file (.github/workflows/ci.yml)

Our first pipeline, naive version:

- `name: CI` — display name in the Actions tab
- `on: push` — trigger on every push, any branch
- one job `build` → one VM (`runs-on: ubuntu-latest`)
- steps: checkout (clone repo onto the VM) → setup-node (install Node 22)
  → `npm install` → `npm run build`, each with
  `working-directory: labs/app-1` since the app isn't at repo root

First run: green in 7s. The steps list also showed "Set up job" (agent
received the job), "Post Run" cleanup steps (in reverse order), and
"Complete job" (logs finalized, VM about to be destroyed).

## Found & fixed #1: deprecation warning on marketplace Actions

The run had 1 warning: checkout@v4 and setup-node@v4 internally run on
Node 20, which is deprecated on runners. Marketplace Actions are
THEMSELVES software with their own runtimes — they age like npm packages.
Fix: bump both to @v5. Lesson: never ignore pipeline warnings; third-party
Actions are dependencies too.

## Found & fixed #2: npm install → npm ci

I spotted this one myself: `npm install` in CI doesn't guarantee the
lockfile versions. Details below.

## Concept: version ranges, pinning, and semver

In package.json: `"express": "^5.2.1"`. The caret means "5.2.1 or newer,
as long as the first number stays 5".

Semver = MAJOR.MINOR.PATCH:
- PATCH (5.1.0 → 5.1.1): bug fixes, safe
- MINOR (5.1 → 5.2): new features, backwards compatible
- MAJOR (5 → 6): breaking changes

**Pinned** = exact version, no symbol ("5.2.1"). **Not pinned** = a range
(^) that several versions can satisfy.

## Concept: dependency resolution

`npm install` must turn every range into one concrete version — that
process is **resolution**: check the registry NOW, pick the newest
matching version, repeat for the whole tree. The result is frozen into
package-lock.json (the lockfile = a snapshot of one resolution).

Danger: resolution depends on what exists in the registry at that moment.
Same package.json on Monday vs Friday can produce different installs —
resolution is NOT reproducible over time. A new minor of a deep dependency
can carry a bug — or malware.

## Concept: npm install vs npm ci (silent fix vs loud failure)

If package.json and the lockfile disagree (e.g. someone hand-edited
package.json):
- `npm install` SILENTLY re-resolves, installs the new thing, and
  rewrites the lockfile. Pipeline stays green. The approved dependency
  list changed invisibly.
- `npm ci` compares the files, sees the mismatch, and FAILS LOUDLY with a
  red X. It never resolves anything — it only installs exactly what the
  lockfile says (and deletes node_modules first for a clean start).

Why loud failure is a feature in CI: the pipeline exists to surface
problems. A tool that papers over a mismatch makes the pipeline lie.
Red X → human looks → deliberate decision. General security principle:
**fail closed, not open.**

## CASE STUDY: the axios supply-chain attack (March 31, 2026)

What happened: an attacker compromised the npm account of the axios lead
maintainer and published two backdoored versions — axios@1.14.1 (tagged
latest) and axios@0.30.4 (legacy). Axios = the most popular JS HTTP
client, ~100M weekly downloads.

The mechanism: the malicious versions added a phantom dependency,
plain-crypto-js@4.2.1 — a package that hadn't existed before that day and
is never imported by axios code. Its only purpose: a **postinstall
script** that drops a cross-platform RAT (remote access trojan) on macOS,
Windows, and Linux. Attribution: a North Korea-nexus state threat actor.

Who got infected: projects whose install resolved `^1.14.0` / `^0.30.0`
ranges fresh during the ~3h window — the ^ range grabbed the newest
(poisoned) version. At least 135 endpoints were seen contacting the
attacker's command-and-control during those 3 hours.

Who was safe (maintainer's own post-mortem): anyone pinned to a clean
version via lockfile who didn't fresh-install in the window.

Scoreboard vs our controls:
| Our control | Effect in this attack |
|---|---|
| lockfile + npm ci | never resolves → never installs the poisoned version |
| blocked install scripts | payload ran via postinstall → blocking stops execution |
| ^ ranges + fresh resolution | the infection path itself |
| "assume compromise" | if it executed: rotate ALL secrets (npm tokens, cloud keys, SSH, CI/CD) |

Sources:
- Maintainer post-mortem: https://github.com/axios/axios/issues/10636
- Google Threat Intelligence: https://cloud.google.com/blog/topics/threat-intelligence/north-korea-threat-actor-targets-axios-npm-package
- Microsoft Security Blog: https://www.microsoft.com/en-us/security/blog/2026/04/01/mitigating-the-axios-npm-supply-chain-compromise/
- Elastic Security Labs (RAT analysis): https://www.elastic.co/security-labs/axios-one-rat-to-rule-all
- Unit 42 threat brief: https://unit42.paloaltonetworks.com/axios-supply-chain-attack/

## Q&A: if ranges are the infection path, why is ^ the npm default?

1. Ranges were meant to HELP security: patches flow in automatically;
   pinned versions go stale until a human remembers to bump them.
2. Dependency trees need ranges to resolve: many packages share
   sub-dependencies; exact-only demands would conflict or duplicate.
3. The default was chosen (2014) before the modern supply-chain threat
   era — optimized for that decade's problems.

Modern resolution of the tension = the two-file system:
- package.json + ^ = INTENT ("I accept compatible updates when I
  consciously update")
- package-lock.json = REALITY (exact frozen versions)
- npm ci = ENFORCEMENT (reality always wins in CI/production)

With this trio, ^ only acts when you deliberately update — a reviewed,
committed, CI-tested act. Axios burned projects with undisciplined
process (no lockfile, npm install in CI, auto-merge bots). Some
high-security teams remove ^ entirely and pin package.json too — valid,
but then every update needs active management (update bot + human review).

## Q&A: how does a pinned, clean version "get" a vulnerability later?

The code never changes — our KNOWLEDGE about it changes. The flaw was
there all along; someone found it.

Lifecycle:
1. Flaw is born — an innocent bug ships in e.g. express@5.2.1.
2. Discovery — by a researcher, an incident investigation, or an attacker
   (if attackers know first: a **zero-day**).
3. Responsible disclosure — private report to maintainers, fix developed
   before publication (typical deadline ~90 days).
4. Fix + CVE — patched version released; the flaw gets a public **CVE**
   ID in the worldwide vulnerability catalog: "express ≤ 5.2.1 vulnerable
   to X, fixed in 5.2.2".
5. The race — the moment the CVE is public, attackers have a map of every
   unpatched app. Automated scans sweep the internet within hours.
   Publication protects the fast and exposes the slow.

"Your version got a vulnerability" = its CVE entry appeared. Yesterday:
unknowingly vulnerable. Today: KNOWINGLY vulnerable — and visible.

This is exactly what SCA scanning in the pipeline does: on every push,
compare all locked versions against the latest CVE catalog and fail the
build on known-vulnerable versions. Results can change overnight with
zero code changes — the catalog moved, not the code.

The two dependency threats pull in OPPOSITE directions:
- Supply-chain attack (axios): malicious code ADDED to a NEW version →
  don't grab new versions blindly (lockfile, npm ci, blocked scripts)
- CVE (discovered flaw): innocent bug FOUND in an OLD version → don't sit
  on old versions (SCA scanning, timely deliberate upgrades)
Security is the balance between the two; the pipeline automates both.

## Official docs
- npm ci: https://docs.npmjs.com/cli/commands/npm-ci
- semver ranges: https://docs.npmjs.com/cli/using-npm/dependency-selectors
- package-lock.json: https://docs.npmjs.com/cli/configuring-npm/package-lock-json
- CVE program: https://www.cve.org/About/Overview