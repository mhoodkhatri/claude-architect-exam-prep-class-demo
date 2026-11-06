# What is CI?

**CI (Continuous Integration)** is the practice of frequently merging code changes into a shared branch, with every change automatically built and tested.

## Key ideas

- Developers integrate small changes often (at least daily) instead of in large, risky batches.
- Each push or pull request triggers an automated pipeline: check out the code, build, run tests and linters.
- The result (pass or fail) is reported back on the commit or PR, so problems are found within minutes.
- A failing build is fixed quickly, keeping the main branch in a working state.

## Why it matters

- Catches bugs and integration conflicts early, when they are cheap to fix.
- Gives reviewers and teammates confidence that a change did not break anything.
- Enforces team standards (formatting, tests, file rules) automatically.
- Is the foundation for CD (Continuous Delivery/Deployment), which automatically releases changes that pass CI.

## Example in this repo

This repo uses GitHub Actions as its CI system. The `Markdown only` workflow runs on pushes to `main` and on PRs targeting `main`, and fails if any tracked file is not Markdown. See [ci-notes.md](ci-notes.md) for details on every workflow.

## Self-check

- [ ] Can I explain the difference between CI and CD?
- [ ] Can I describe what triggers a CI run and what it reports?
