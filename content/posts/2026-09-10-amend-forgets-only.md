---
title: "amend forgets --only"
date: 2026-09-10
draft: false
categories: ["tools"]
tags: ["git", "debugging"]
summary: "git commit --only scopes a commit to specific paths, but --amend on top of it commits the whole index anyway."
---

`git commit --only path/a path/b` is the move when the working tree has a pile of unrelated staged changes from other work you don't want to touch yet. It commits just the paths you name, leaving everything else staged exactly as it was.

Then I needed to fix one thing in that same commit, so I ran `git commit --amend`. Obvious move, or so I thought.

`--amend` doesn't remember the `--only` scoping from the commit it's amending. It just commits the current index, in full. Everything that was sitting staged from the unrelated work rode along into the amended commit, silently.

I caught it by running `git show --stat HEAD` instead of trusting the commit summary in the terminal output, which only echoes the message, not the actual file list. The diff had files in it I hadn't touched all session.

Fix was `git reset --soft HEAD~1`, which puts the index back to how it was before the amend, then a fresh `git commit --only path/a path/b` to redo the scoped commit correctly.

Now if I need to amend a `--only` commit, I reset and re-run with `--only` again rather than trusting `--amend` to remember what I meant the first time. And I check `git show --stat HEAD` after any commit that matters, not just the printed summary.
