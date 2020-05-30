---
title: "Learning to Use Git"
date: 2020-05-29T19:13:50-05:00
draft: false
tags: [
	"Git",
]
---

[![](/images/git.png)](https://git-scm.com)

# Learning to use Git

[Git](https://git-scm.com) is a version control program that is very useful for developing any type of software of even documentation with the ability to branch and merge at any point. It has many advanced features such as cherry picking and the ability to pick certain changes from other branches and merge into the main branch usually called the master branch.

This guide should be cross-platform compatible with Windows and Unix-like systems. For Windows, [PowerShell](https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-7) or [Git Bash](https://gitforwindows.org/) are recommended as commands will be the same as written below.

---

## Main concepts:

- **Repositories:** the name of where code is stored and can be hosted at sites such as [Github](https://github.com).

- **Branches:** are sub-repositories that allow different versions of code to be developed simultaneously.

- **Master branch:** the main branch name which usually stores release code and or the upstream of other branches.

- **Commits:** is a stated change that has been added to repository histroy with a unique [SHA1](https://en.wikipedia.org/wiki/SHA-1) hash value.

- **Pull requests:** is a proposed commit that is pointed towards a repository branch, usually made by 3rd party developers or used for branch merging.

- **Releases:** are tagged points in repository history usually for production ready versions.

---

## Requirements:

- [Git](https://git-scm.com)

---

## Quick start guide:

### Step 1. Make a directory on your computer and enter it.

```
mkdir example
cd example
```

### Step 2. Initialize a new Git repository.

```
git init
```

### Step 3. With new Git repository create README.md and commit it.

Create README.md file

```
touch README.md
```

Make sure to add file to git tracking as these files will be commited next.

```
git add README.md
```

Shows changes to be committed.

```
git status
```

Commits and staged changes.

```
git commit -m "Initial Commit"
```

Displays any commits on this repository in a clean one line format with a partial unique hash. 

```
git log --oneline
```

What was demonstrated in Step 3 were the fundamental basics of adding a file to a Git repository and commiting changes to the repository log.

---

## Git Advanced:

### [Git Branches:](https://git-scm.com/docs/git-checkout)

This section will discuss branches as well as being able to merge commits and pull requests across branches.

- **Creating Branches**

First line shows which branch current project directory is following.

```
git status
```

Creating a new branch.

```
git checkout -b test
```

After creating a new branch you can check "git status" to make sure that branch has been changed.

One thing about branches new branches is that there needs to be a new upstream set if "git push" to a remote repository Git will tell you the commands to run.

- **Switching Branches**

```
git checkout branch
```

- **Merging Branches**

For the scenarios where a development branch needs to be merged into usually the master branch.

This will automatically use latest commit on branch and merge it to master

```
git request-pull master ./
```

For specified commit to push to master.

```
git request-pull <commit> master ./
```

Switch to master branch and merge change from other branch.

```
git checkout master
git merge branch 
```

### [Stashing Changes](https://git-scm.com/docs/git-stash)

Git stash is useful for saving any local changes when performing any changes with repository.
```
git stash
```

### [Reverting Commits:](https://git-scm.com/docs/git-revert)

How to revert Git repository to a previous commit.

This command reverts repository files back to lastest commit.

```
git log --oneline
git reset --hard <commit> 
```

This command reverts repository commit history without affecting files.

```
git log --oneline
git reset --soft <commit>
```

Viewing any reverts or resets in a Git repository.

```
git reflog --oneline
```

Any value after ~ will be the amount of commits to rollback.
```
git reset HEAD~1
```

---

## Resources:

- [Differences between Git reset and Git revert](https://www.pixelstech.net/article/1549115148-git-reset-vs-git-revert)

- [Git Advanced Tips](https://www.atlassian.com/git/tutorials/advanced-overview)

- [Git Commands](https://git-scm.com/docs/git)

- [Git Documentation](https://git-scm.com/docs)

- [How to reset, revert, and return to previous states in Git](https://opensource.com/article/18/6/git-reset-revert-rebase-commands)


