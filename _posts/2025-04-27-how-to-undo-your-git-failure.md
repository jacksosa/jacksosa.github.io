---                                                                                                                                                                                                                                                   
title: Git | How to Undo Your Git Failure?
tags: [Git, Version Control, Development]
style: fill
color: warning
description: Using `git reflog` and `git reset` to Save Your Code
---

---

## How to Undo Your Git Failure?

Using `git reflog` and `git reset` to Save Your Code

Git is a powerful version control system, but it’s not immune to user errors. Whether you’ve accidentally reset a
branch, deleted a `commit`, or `force-pushed` over your work, Git’s safety net `git reflog` and `git reset` can often
save the day. In this post, we’ll explore how to use these tools to recover lost code, with practical examples and tips
to avoid common pitfalls.

### What is `git reflog`?

`git reflog` (reference log) is like Git’s memory of every action that updates the **HEAD** pointer, including
`commits`, `resets`, `checkouts`, and `merges`. Unlike the regular Git log, which shows the commit history of the
current branch, reflog tracks all **HEAD** movements, even those that are no longer reachable in your branch’s history.
Think of it as a time machine for your repository. If you’ve lost a commit, reflog can help you find it. Viewing the
Reflog Run this command to see the reflog:

```bash
git reflog
```

Output might look like:

```plaintext
abc1234 HEAD@{0}: reset: moving to HEAD^ # Recent reset
def5678 HEAD@{1}: commit: Added feature X # Your lost commit
ghi9012 HEAD@{2}: checkout: moving to main # Branch switch
````

Each entry shows:

* A commit SHA (e.g., abc1234).
* The action (e.g., reset, commit, checkout).
* A description of what happened.

### Common Git Failures and How to Fix Them

Let’s walk through typical scenarios where things go wrong and how `git reflog` and `git reset` can rescue your code.

#### Scenario 1: Accidentally Reset a Commit

You’ve just run `git reset --hard HEAD^` to undo the last commit, but you realize you needed that work.
**Step 1**: Check the Reflog Run `git reflog` to find the commit you want to recover.

```bash
git reflog
```

Identify the commit you want to recover:

```plaintext
abc1234 HEAD@{0}: reset: moving to HEAD^
def5678 HEAD@{1}: commit: Added feature X
```

def5678 is the commit you want to recover.
**Step 2**: Restore the Commit using `git reset` to move **HEAD** back to that commit:

```bash
git reset --hard def5678
```

This restores your branch to include the commit at def5678. Your code is back!

#### Scenario 2: Lost a Commit After a Force Push

You `force-pushed` (`git push --force`) and overwrote commits on the remote. Those commits seem gone, but they’re still
in your local `reflog`.
**Step 1**: Check the Reflog Run `git reflog` to find the commit before the force push.

```bash
git reflog
```

Identify the commit before the force push:

```plaintext
xyz7890 HEAD@{2}: push: forced update
def5678 HEAD@{3}: commit: Fixed bug Y
```

def5678 is your lost commit.
**Step 2**: Recover the Commit using `git reset` to move **HEAD** back to that commit:

```bash
git reset --hard def5678
```

Then, push it back to the remote (carefully, to avoid overwriting others’ work):

```bash
git push origin main
```

If others have pushed since, you may need to merge or rebase first.

#### Scenario 3: Dropped a Stash

You stashed some changes (`git stash`), then accidentally dropped them (`git stash drop`). Stashes are stored in the
reflog.
**Step 1**: Check the Stash Reflog Run `git reflog stash` to find the dropped stash.

```bash
git reflog stash
```

Look for the stash you dropped:

```plaintext
abc1234 stash@{0}: drop: dropped stash
def5678 stash@{1}: stash: Stashed changes
```

def5678 is your dropped stash.
**Step 2**: Recover the Stash using `git stash apply` or create a branch from it:

```bash
git stash apply def5678
```

Or create a branch from it:

```bash
git checkout def5678
git branch recovered-stash
```

## Using `git reset` Wisely

`git reset` moves the branch pointer to a specific commit, but it comes in three flavors:

* `--soft`: Keeps changes in the working directory and index (staged).
* `--mixed `(default): Keeps changes in the working directory but unstages them.
* `--hard`: Discards all changes, resetting everything to the target commit.

For recovery, `--hard` is common to restore the exact state, but use `--soft` or `--mixed` if you want to preserve
uncommitted changes:

```bash
git reset --soft def5678 # Recover commit, keep changes staged
```

### Best Practices to Avoid Git Disasters

1. **Check Reflog First**: Before panicking, always run `git reflog` to see what happened.
2. **Backup Branches**: Before risky operations (e.g., `reset`, `force push`), create a backup branch, for example:
   `git branch backup-main` and use `--force-with-lease`: Instead of `git push --force` to prevent overwriting others’
   commits (e.g., `git push --force-with-lease origin main`).
3. **Commit Often**: Frequent commits make it easier to recover specific points.
4. **Clean Reflog Periodically**: Reflog entries expire (default: 90 days). To keep important ones, tag them for
   example: `git tag recovery-point def5678`

### When `git reflog` Can’t Help

* **No Local Reflog**: If you only have the remote repo (e.g., after cloning), reflog won’t exist. Check remote logs or
  CI/CD artifacts.
* **Garbage Collection**: If `git gc` ran and expired reflog entries (unlikely within 90 days), recovery is harder.
* **Uncommitted Changes**: If you didn’t commit or stash changes before resetting, they’re likely gone unless in your
  editor’s undo history.

### Conclusion

Git failures can feel catastrophic, but `git reflog` and `git reset` are your lifelines. By understanding the reflog’s
history and using reset strategically, you can recover lost commits, stashes, and branches with confidence. Next time
you make a Git mistake, take a deep breath, check the reflog, and roll back time.

Have you ever lost code in Git? Share your recovery stories with me [here]({{ site.baseurl }}/contact/), or let me know
if you need help with a specific Git mishap!


<p class="text-center">
{% include elements/button.html link="https://git-scm.com/docs/git-reflog" text="Git Reflog Docs" %}
{% include elements/button.html link="https://git-scm.com/docs/git-reset" text="Git Reset Docs" %}
</p>
