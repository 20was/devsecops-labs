# Module 0 – Baseline

## Idea 1: The old way of shipping software

Before DevOps, a company had two separate teams.

- **Developers** wrote the code on their laptops.
- **Operations (Ops)** owned the servers where the code actually runs.

When developers finished a feature, they handed the code to Ops and said
"please put this on the servers."

This handoff caused constant problems:

- The code worked on the developer's laptop but broke on the server, because
  the two machines were set up differently.
- The handoff was manual — someone copied files, edited configs by hand,
  restarted things. Humans make mistakes.
- Because releasing was painful, companies did it rarely — maybe once every
  few months. Each release was huge, so when something broke, nobody knew
  which of the 500 changes caused it.

In short: **shipping code to servers was slow, manual, and scary.**

## Idea 2: DevOps = automate that path

DevOps is one main idea: **replace the manual handoff with an automated
machine.**

You push code → a machine automatically tests it → if the tests pass, the
machine automatically puts it on the servers.

No wall between teams. No handoff. No human copying files.

Because it's automatic and reliable, you can release small changes many
times a day instead of one giant release every 3 months. Small changes are
easy to test, and easy to undo if something breaks.

That automated machine is called a **pipeline**. This is the most important
word in this whole module.

## Idea 3: Security had the same "wall" problem

Security used to work exactly like Ops did: as a separate team at the end
of the process.

A security team would check the app for weaknesses **right before release** —
like a building inspection after the building is fully constructed.

Problems with this:

- If they found a serious flaw, it was buried deep in the finished product.
  Fixing it meant tearing things apart. Expensive and slow.
- The check happened once. Code written the day after the check went
  completely unchecked.
- Developers and security barely talked, so developers kept making the same
  mistakes over and over.

## Idea 4: DevSecOps = put security checks inside the pipeline

We already have a machine that runs automatically on every code push — the
pipeline. DevSecOps says: **add security checks to that machine.**

So on every push, the pipeline doesn't just ask "does the code work?" —
it also checks things like:

- Did someone accidentally include a password in the code?
- Is the code using a library with a known security hole?
- Does the code contain a pattern that hackers commonly exploit?

If any security check fails, the pipeline stops the release — exactly the
same way a failing test stops it.

The payoff: a flaw gets caught **minutes after it's written**, when it's a
one-line fix — not months later during an inspection, when it's an
expensive teardown.

Every security tool I'll learn in the next 6 months is just a different
kind of automated check bolted onto this pipeline.

## What is this lab for?

Over 6 months I will build a real app, a pipeline, cloud infrastructure,
and a Kubernetes cluster. The pattern for every module:

1. Build the simple (possibly insecure) version first.
2. Then harden it step by step using real security tools and best practices.

This repo is my portfolio: `notes/` for explanations like this one,
`labs/` for the projects, `diagrams/` for architecture drawings.

## Q&A from learning session

**Q: Why were manual handoffs painful?**
Because the developer's machine and the server were set up differently
("works on my laptop"), manual steps caused human error, and rare, huge
releases made failures hard to trace back to a specific change.

**Q: What triggers the pipeline?**
A code push. Every single push runs the tests automatically — and later in
this lab, security checks too.