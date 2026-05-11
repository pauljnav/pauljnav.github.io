---
layout: post
title: "Contributing to PSAISuite: A Practical, Real‑World Guide"
date: 2026-05-11
categories: [opensource, psaisuite, powershell]
tags: [git, github, pull-requests, contributions, workflow]
author: Paul
description: "A clear, practical guide to contributing to the PSAISuite PowerShell module using Git, GitHub, branches, PRs, and best practices."
---

# **Contributing to PSAISuite: A Practical Guide for New Contributors**

Open‑source thrives when people contribute to the community effort — not only with code, but with ideas, documentation, bug reports, and improvements. When I first joined **[PSAISuite](https://github.com/dfinke/psaisuite)**, I found the hardest part wasn't the code; it was the mechanics of the workflow. Handling forks, remotes, branches, and PR etiquette can be daunting, but once you have that down, contributing becomes fun.

**Two key tips for new contributors:**

* **Start the conversation early:** You don't need a finished change to open a Pull Request. Use a **Draft PR** to signal you’ve claimed a task. This invites feedback before you spend a lot of time on something.
* **Respect the foundation:** You are a guest in the codebase. Avoid refactoring project structure or changing style. Stick to existing patterns to ensure your contribution is welcome and easy to merge.

This guide walks through the steps I follow.
It's simple, repeatable, and avoids the common pitfalls that can frustrate new contributors.

---

## **1. Fork the Upstream Repository**

You never work directly in the owner's repository.
Instead, you:

1. Go to the PSAISuite GitHub page
2. Click **Fork**
3. GitHub creates a copy under your account

**Terminology:**

- **upstream** → the owner's repo
- **origin** → your fork

Your fork is your sandbox. You can experiment freely without affecting the upstream project.

---

## **2. Clone *Your Fork* (origin)**

```bash
git clone https://github.com/<you>/PSAISuite.git
cd PSAISuite
```

At this point:

```bash
git remote -v
```

shows only your fork:

```
origin  https://github.com/<you>/PSAISuite.git (fetch)
origin  https://github.com/<you>/PSAISuite.git (push)
```

---

## **3. Add the Upstream Remote (and why this matters)**

Add the owner's repo as a second remote:

```bash
git remote add upstream https://github.com/dfinke/psaisuite.git

```

Now:

```bash
git remote -v
```

shows both:

```
origin    <your fork>
upstream  <owner's repo>
```

### **Why add upstream?**

Because your fork will fall behind the real project.
Without an upstream remote, you have no clean way to pull in the latest changes.

This leads to:

- merge conflicts
- stale branches
- PRs that can't be merged
- frustration for maintainers

Adding upstream is how you keep your fork healthy.

---

## **4. Never Develop on `main` (a simple story)**

Here's the story I use when teaching this:

> Imagine your `main` branch is your “clean room.”
> It should always match upstream exactly — no experiments, no edits, no half‑finished work.
>
> If you start hacking on `main`, and upstream changes while you're working, your `main` becomes a tangled branch.
>
> Now you can't cleanly sync with upstream, and every future PR becomes a conflict nightmare.

This is why experienced contributors **never** develop on `main`.

### **But is it OK for tiny edits?**

Yes — *if*:

- it's a trivial change (typo, formatting, documentation), and
- you immediately open a PR, and
- you don't plan to keep working on `main`

But even then, the best practice is:

> **Always create a branch.**
> Even for a one‑line typo fix.

It keeps your workflow consistent and prevents accidental messes.

---

## **5. Create a Feature/Bugfix/Docs Branch**

Branch names should describe the change:

```bash
git checkout -b feature/add-model-metadata
```

This isolates your work and keeps PRs clean.

---

## **6. Make Your Edits and Commit Them (Using Conventional Commits)**

Edit files, then:

```bash
git add .
git commit -m "feat: add model metadata extraction"
```

### **Conventional Commits Format**

```
<label>: <short description>

<optional longer explanation>
```

Examples:

```
feat: add model metadata extraction
```

```
fix: correct null reference in Invoke-ChatCompletion

The previous implementation failed when no system message was provided.
```

Common labels:

- `feat:` new feature
- `fix:` bug fix
- `docs:` documentation
- `refactor:` code cleanup
- `test:` tests
- `chore:` non‑code tasks

This makes commit history readable and searchable.

---

## **7. Sync Your Branch With Upstream Before Pushing**

Keep your branch up to date to avoid conflicts later.

```bash
git fetch upstream
git merge upstream/main
```

Or, for a cleaner history (locally and in the upstream):

```bash
git fetch upstream
git rebase upstream/main
```

This updates **your local branch**, not your fork.

**NB — `git merge` vs `git rebase`:**  
Use **git merge** if you've already pushed your branch — it preserves shared history and avoids rewriting commits that others may have pulled.  
Use **git rebase** to clean up your commits *before your first push*, producing a cleaner history that also results in a tidier upstream merge.

**Never rebase after a push** because it rewrites commit hashes. Anyone who has already pulled your branch will suddenly have a different history, causing conflicts, confusion, and extra manual effort to reconcile diverging branches.

Also see note **#10 (Draft PRs)**:  
Rebasing aims for one polished commit in the upstream history, while a Draft PR encourages incremental commits during collaboration — both are useful, but they are like opposites and draft PRs encourage many commits during collaboration.

---

## **8. Push Your Branch to Your Fork**

If this is the first push of this branch:

```bash
git push --set-upstream origin feature/add-model-metadata
```

### **Why `--set-upstream`?**

Because Git doesn't yet know:

- which remote branch your local branch should track
- where future `git push` and `git pull` should go

After the first push, you can simply run:

```bash
git push
```

Git now knows the relationship.

---

## **9. Open a Pull Request (PR)**

GitHub on detecting the change will show:

> Compare & pull request

Choose:

- **base:** upstream/main  (target upstream/main, not your fork's main)
- **compare:** origin/feature/add-model-metadata

Write a clear PR description.

---

## **10. Draft PRs (A Great Tool for Early Feedback)**

A **Draft PR** is perfect when:

- you want early visibility  
- you want CI to run  
- you want to discuss the approach  
- you're not ready for a full review  

Rebase before your first push; that first push is what opens the Draft PR if you choose to draft.

I used this myself:

1. Opened a Draft PR early  
2. Continued pushing commits  
3. Once tests passed and code was ready  
4. Clicked **“Ready for review”**  

This signals to maintainers:

> “I'm done — please review.”

Draft PRs can improve collaboration when suitable.

---

## **11. Respond to Review Feedback**

When maintainers request changes:

```bash
git add .
git commit -m "fix: address review feedback"
git push
```

GitHub updates the PR automatically.

---

## **12. Getting Merged**

Once approved, maintainers merge your PR into upstream.

Your contribution becomes part of the project.

---

## **13. Cleaning Up**

After merge:

```bash
git checkout main
git pull upstream main
git branch -d feature/add-model-metadata
git push origin --delete feature/add-model-metadata
```

This keeps your fork tidy as it updates your local main to match upstream.

---

# **Contributor Checklist**

A quick checklist you can follow every time:

- [ ] Fork upstream
- [ ] Clone origin
- [ ] Add upstream remote
- [ ] Never develop on `main`
- [ ] Create a feature branch
- [ ] Make changes
- [ ] Commit (use Conventional Commits)
- [ ] Sync with upstream
- [ ] Push branch (`--set-upstream` on first push)
- [ ] Open PR (as draft if needed)
- [ ] Respond to reviews
- [ ] Merge by owner
- [ ] Clean up local + remote branches
