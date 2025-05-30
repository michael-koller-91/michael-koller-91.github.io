---
layout: default
title: "Solving submodule pointer git rebase conflicts"
date: 2025-05-27
---

# The problem

We have two git repositories.
One repository called `super` and one repository called `sub` which is a submodule of the repository `super`.
In each repository, we have both a main and a dev branch.

The graph of `sub` looks as follows:
```
* 96ccceb (HEAD -> dev) s5
* feb588c s4
| * 2865633 (main) s3
|/
* a3f0de6 s2
* a5dfe38 s1
```
The main branch consists of the three commits with commit messages "s1", "s2", and "s3".
Branch dev was created off commit "s2" and has the two commits "s4" and "s5".

The graph of `super` looks like this:
```
* 2e0bb3b (HEAD -> dev) m6: s5
* 9112274 m5
* ca7130a m4: s4
| * 2230920 (main) m3: s3
|/
* fb9be25 m2
* 651d7db m1: s1
```
The main branch consists of the three commits with commit messages "m1: s1", "m2", and "m3: s3".
Branch dev was created off commit "m2" and has the three commits "m4: s4", "m5", and "m6: s5".
The commit messages tell us the corresponding submodule state.
For example, "m1: s1" means that with this commit, the submodule `sub` was checked out at commit "s1".
And "m6: s5" means that with this commit, the submodule `sub` was checked out at commit "s5".

If we `git rebase main` in `sub`, we get the following graph:
```
* ca00e6c (HEAD -> dev) s5
* f1bb5d6 s4
* 2865633 (main) s3
* a3f0de6 s2
* a5dfe38 s1
```
Note how the commit hashes of "s4" and "s5" have changed.
The commit hash of "s4" used to be `feb588c` but now it's `f1bb5d6`.
Similarly, the commit hash of "s5" used to be `96ccceb` but now it's `ca00e6c`.

The fact that the commit "s4", which `super` refers to in its commit "m4: s4", now de-facto no longer exist, leads to a problem when we run `git rebase main` in `super`:
```
Failed to merge submodule sub
CONFLICT (submodule): Merge conflict in sub
error: could not apply ca7130a... m4: s4
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
Could not apply ca7130a... m4: s4
```
The commit "m4: s4" can no longer be applied.
The conflict has two reasons.
First, the old commit hash of "s4" to which "m4: s4" points, no longer exists.
Second, on branch main, onto which we are currently rebasing, there was also a submodule commit: "m3: s3".

# The theoretical solution

Git now asks us to solve the conflict manually.
In principle, the solution is simple.
We know that "m4: s4" should point to "s4", so we look up the new commit hash of "s4", namely `f1bb5d6`, and add the corresponding submodule state:
```bash
cd sub
git checkout f1bb5d6
cd ..
git add sub
git commit
git rebase --continue
```
Now "m4: s4" points to "s4" again, the conflict is solved, we can continue rebasing.
Rebasing "m5" will not lead to a problem but the last commit "m6: s5" will give as a merge conflict again:
```
Failed to merge submodule sub (commits don't follow merge-base)
CONFLICT (submodule): Merge conflict in sub
error: could not apply 2e0bb3b... m6: s5
# ...
```
We'll solve it in a similar manner.
We look up the new commit hash of "s5", namely `ca00e6c`, and add the corresponding submodule state:
```bash
cd sub
git checkout ca00e6c
cd ..
git add sub
git commit
git rebase --continue
```
This finishes the rebase successfully and we end up with the following graph in `super`:
```
* d24c8c2 (HEAD -> dev) m6: s5
* c5cecb2 m5
* 91666f7 m4: s4
* 2230920 (main) m3: s3
* fb9be25 m2
* 651d7db m1: s1
```

# The challenge

The challenge with the theoretical approach is that for every submodule commit in `super`, we need to know what the corresponding new commit hash in `sub` is.
In our toy example, this is simple - in particular because the commit messages already tell us the right commit in `sub`.
In general, solving the merge conflict is not so simple.
Our only chance is running `git reflog` in `sub`.
In the reflog, we can find the old commit hash and the corresponding commit message.
If the commit message is unique, we can look for it in the reflog and that will give us the new commit hash.
If the commit message is not unique, we're in trouble.

In what follows, we'll write two useful git aliases that help us solve the described kind of merge conflicts.

# The practical solution

Here's what we'll do:
1. Before we rebase in `sub`, we find all commits on `sub`'s dev branch to which a commit in `super`'s dev branch points.
    * In our example, we would find the commits "s4" and "s5".

2. For every such commit, we make a new branch pointing to it. The branch name will contain the (pre-rebase) commit hash.
    * In our example, we would create two branches: pre-rebase-feb588c and pre-rebase-96ccceb

3. We rebase in `sub` with the command `git rebase main --update-refs`. The argument `--update-refs` force-updates all branches that point to commits that are being rebased.
    * In our example, this would lead to the following `sub` graph:

        ```
        * ca00e6c (HEAD -> dev, pre-rebase-96ccceb) s5
        * f1bb5d6 (pre-rebase-feb588c) s4
        * 2865633 (main) s3
        * a3f0de6 s2
        * a5dfe38 s1
        ```

4. Whenever we get a submodule pointer merge conflict while rebasing in `super`, we get the old `sub`-commit to which the conflicting `super`-commit points, we find the corresponding `pre-rebase`-branch, and then the commit to which the branch is pointing is the commit to which `super` should be pointing to solve the conflict.
    * In our example, when we do `git rebase main` in `super`, we get a merge conflict while rebasing "m4: s4". Git tells us that we have a submodule merge problem. So we get the commit hash `feb588c` to which "m4: s4" is currently pointing (we'll see how in a minute), then we find the corresponding branch `pre-rebase-feb588c` in the graph and read off `f1bb5d6` as the new commit hash for `feb588c`.

5. Now that we have the new commit hash, we solve the merge conflict and continue rebasing.
    * In our example, we would

        ```bash
        cd sub
        git checkout f1bb5d6
        cd ..
        git add sub
        git commit
        git rebase --continue
        ```

In what follows, we'll write two convenient functions to implement the practical solution.

## Git plumbing commands

We'll write two git aliases to achieve our goal.
To this end, we'll make use of some what [git-scm](https://git-scm.com/) calls [plumbing commands](https://git-scm.com/docs).

#### 1. Find `super`-commits containing submodule pointers and get the corresponding `sub`-commit

We need to iterate through all commits on `super`'s dev branch and check if they contain pointers to `sub`.
To iterate through a range of commits, we can use `git rev-list --abbrev-commit` where `--abbrev-commit` gives us only a prefix for every commit to make the output more readable.
A branch consists of all commits from repository root to tip.
Since we are only interested in the part of branch dev which is different from main, we use `git merge-base main dev` to find the best common ancestor.
Overall, this leaves us with:
```bash
# find all commits on branch dev from where dev branched off main to the tip
git rev-list --abbrev-commit ($git merge-base main dev)..dev
# prints (without the comments which I added for clarity):
2e0bb3b # "m6: s5"
9112274 # "m5"
ca7130a # "m4: s4"
```

To get the contents of a commit, we use `git show`.
However, we're only interested in a plain list of all files in the commit, we don't need the diff.
In our example, we get:
```bash
git show --name-only --format="" 2e0bb3b # "m6: s5"
# prints:
sub

git show --name-only --format="" 9112274 # "m5"
# prints:
m5.txt

git show --name-only --format="" ca7130a # "m4: s4"
# prints:
sub
```
All commits which contain submodule pointers contain the submodule name as a "filename".
Hence, as we iterate through all commits (rev-parse) and and then iterate through all files in them  (show), we look for filenames matching "sub".
For every match, we want to know to which `sub`-commit the commit points.
This can be achieved with ls-tree:
```bash
git ls-tree ca7130a sub # "m4: s4"
# prints:
160000 commit feb588c254bfda6252bf499b0b7c1420313cf774  sub
```
Here, we identify `sub`-commit "s4" from its prefix "feb588c".

So far, we have
```bash
# iterate through all commits
for commit in $(git rev-list --abbrev-commit $(git merge-base main dev)..dev); do
    # iterate through all filenames in the commit
    for filename in $(git show --name-only --format="" $commit); do
        # if the filename is sub, we found a submodule pointer
        if [ $filename == "sub" ]; then
            # get the corresponding commit in sub to which the commit points
            t=$(git ls-tree $commit sub)
            arrt=(${t// / }) # split what ls-tree returns at spaces ...
            subcommit=${arrt[2]} # ... and get the third element
            # 
            # step 2.
            #
        fi
    done
done
```

### 2. Make a branch for every `sub`-commit found in step 1.

Once we identify for example that `feb588c254bfda6252bf499b0b7c1420313cf774` (commit "s4" in `sub`) is a commit in `sub` to which `super`'s dev points, we want to create a branch called `pre-rebase-feb588c254bfda6252bf499b0b7c1420313cf774` that points to it.
This can be achieved with the plumbing command `update-ref`.
Its first argument is `refs/heads/branchname` where we replace "branchname" with our desired branch name, and its second argument is the commit hash to which the branch should point.
Thus, we can finish implementing what we started in step 1.:
```bash
#
# step 2.
#
# choose the branch name
branchname="pre-rebase-$subcommit"
# make a branch in sub
cd sub
git update-ref refs/heads/$branchname $subcommit
cd ..
```

### Steps 3.-5.

In step 3., we simply do the rebase in `sub` but with `--update-refs`.
To make the steps 4. and 5. simpler, we write a small function that lists all `pre-rebase-*`-branches and the commit to which they now point.
The commit to which a branch points is given by `rev-parse`.
```bash
# in sub
cd sub
# iterate through all branches starting with pre-rebase-
for branch in $(git branch --list "pre-rebase-*"); do
    # and print the branch name as well as the commit to which it points
    echo "old commit hash: $branch  --  new commit hash: $(git rev-parse $branch)"
done
cd ..
```
Now, whenever we encounter a submodule pointer merge conflict, we take a look at our list to find out to which commit the submodule should be pointing, then we solve the conflict and continue rebasing.

## Git aliases

We can wrap the two helpers that we derived into git aliases.
A git alias needs to be added to the alias section in the .gitconfig file.
I call my two aliases `prepare-rebase-branches` and `list-rebase-branches`.
They can be found [here](https://github.com/michael-koller-91/.dotfiles/blob/main/.gitconfig).
In our example, they would be used as follows:
```bash
git prepare-rebase-branches dev sub main
git list-rebase-branches sub
```
Note that `sub` needs to be the relative path to the submodule.

# Bonus: Add a submodule pointer without doing a checkout in the submodule

When we want to solve one of our conflicts, instead of doing
```bash
cd sub
git checkout f1bb5d61791fe96927ad78e64b0aad7bded08bdc
cd ..
git add sub
# git commit
# git rebase --continue
```
we can use a plumbing command
```bash
git update-index --cacheinfo 160000,f1bb5d61791fe96927ad78e64b0aad7bded08bdc,sub
# git commit
# git rebase --continue
```
where 160000 is the mode of a submodule which does not depend on the commit hash or the submodule name.

[back](./../)
