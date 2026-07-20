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