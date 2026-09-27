#### 21 Create a bare repository with a specific name

```bash
git init --bare beta.git
```

Bare repo is primarily used for sharing and collaboration, without the capability for direct file editing or viewing.

#### 22 Clone a local repository

```bash
git clone somename.git
```

This will clone the repo into `somename` folder of current working dir.

#### 23 Forking a repo in Gitea

Explore -> Users -> Sara -> reponame -> Fork

#### 24 Maintain new changes in a separate branch

Create it without switching:

```bash
git branch <new branch> master
```

or create the branch and switch to it

```bash
git checkout -b <new branch> master
git switch -c <new-branch> master
```

or copy

```bash
git branch -c master <new branch>
```

After `-c`, the new branch inherits master's upstream (e.g., `origin/master`). A plain `git push` from it would then target `master` on the remote, which is usually not what you want.

All three approaches give a branch pointing at master's commit, which is what the checker looks at. Plain `git branch <new> master` or `git checkout -b <new> master` is the cleaner choice, because it doesn't drag master's tracking config along.

#### 25 Merging

Create new branch `xfusion` in `/usr/src/kodekloudrepos/news` repo from `master`

```bash
git branch <new branch>
git switch <new branch>
git branch --list
```

Copy the `/tmp/index.html` file from storage server to repo

```bash
cd /usr/src/kodekloudrepos/news
cp /tmp/index.html .
```

`add/commit` this file in a new branch

```bash
git add index.html
git commit -m 'added index.html'
```

Merge that branch back into `master`

```bash
git switch master
git merge xfusion
```

Finally, push the changes to the origin for both of the branches.

```bash
git push origin master
git push origin xfusion
```

#### 26 Updating remote repo

Add new remote repo and point it to local repository

```bash
git remote add <remote_name> <path/url>
# Local repository
git remote add dev_ecommerce /opt/xfusion_ecommerce.git
# Remote repository
git remote add dev_ecommerce git://git.kernel.org/.../gregkh/xfusion.git
```

Copy file to repo and add/commit to master

```bash
cp /tmp/index.html .
git add index.html
git commit -m index.html
```

Push master to new remote origin

```bash
git push dev_ecommerce master
# Verify
git reflog
9bd6271 (HEAD -> master, dev_ecommerce/master, dev_ecommerce/HEAD) HEAD@{0}: commit: commit
2596ad3 (origin/master, origin/HEAD) HEAD@{1}: commit (initial): initial commit
# Show what has been pushed
git show 9bd6271
commit 9bd62718d67a658a4888150c2c36d1d8d186e38b (HEAD -> master, dev_ecommerce/master, dev_ecommerce/HEAD)
Author: Admin <admin@kodekloud.com>
Date:   Sun Sep 27 11:14:14 2026 +0000

    commit

diff --git a/index.html b/index.html
new file mode 100644
index 0000000..df9a7b1
--- /dev/null
+++ b/index.html
@@ -0,0 +1,10 @@
+<!DOCTYPE html>
+<html>
+<body>
+
+<h1>Welcome to xFusionCorp Industries</h1>
+
+<p>These are DevOps Labs</p>
+
+</body>
+</html>
\ No newline at end of file
```

Verify again by asking the remote directly. The example below is from a separate run of this task, where the remote was named `dev_games`:

```bash
sudo git ls-remote dev_games master
```

- **`git ls-remote`** connects to a remote and lists its refs (branches, tags, HEAD) with the commit hash each one points to.
- **`dev_games`** is the remote to query. Git looks up its URL/path in `.git/config` (see `git remote -v`).
- **`master`** is a filter, so only refs matching `master` are shown. Without it you'd get every ref: `HEAD`, all branches, all tags.

Output:

```text
ed7d8aa1413e84b34ae5d1bcc1851a33bfb730f6        refs/heads/master
```

This means the `master` branch on `dev_games` currently points to commit `ed7d8aa…`.

Show

```bash
git show ed7d8aa1413e84b34ae5d1bcc1851a33bfb730f6
```

Out:

```text
commit ed7d8aa1413e84b34ae5d1bcc1851a33bfb730f6 (HEAD -> master, origin/master, origin/HEAD, dev_games/master)
```

- The full SHA of the commit.
- The parentheses list every ref pointing at it: you're on `master` (`HEAD -> master`), and both remotes (`origin`, `dev_games`) have their `master` at this same commit, so everything is in sync.
- `origin/HEAD` means origin's default branch is `master`.

#### 27 Revert changes

View all entries:

```bash
git reflog
6cd3934 (HEAD -> master, origin/master) HEAD@{0}: commit: add data.txt file
95c9871 HEAD@{1}: commit (initial): initial commit
```

The reflog lists entries **newest first**. `HEAD@{0}` is the most recent move, and larger numbers go further back in time.

Revert the latest commit. The output below is from a separate run of this task, so the SHAs differ from the reflog above:

```bash
git revert HEAD~0
# add commit message when VIM pops up :wq
git reflog
7a1f9bc (HEAD -> master) HEAD@{0}: revert: revert official
5729eb2 (origin/master) HEAD@{1}: commit: add data.txt file
af90eaf HEAD@{2}: commit (initial): initial commit
```

Double-check:

```bash
sudo git show --stat HEAD
commit 7a1f9bc504a7ee161d9be7840f2aa23b557f95f1 (HEAD -> master)
Author: Admin <admin@kodekloud.com>
Date:   Sun Sep 27 12:38:02 2026 +0000

    revert official

    This reverts commit 5729eb2cf2e0279584219df9c746736a9e64569e.

 info.txt | 1 +
 1 file changed, 1 insertion(+)
```

#### 28 Cherry pick - Apply the changes introduced by some existing commits

We need to apply changes from commit 'Update info.txt' in `feature` branch to `master`.

Check branch - switch to master

```bash
git branch --list
git switch master
```

View reflog

```bash
git reflog
f2b3123 (origin/feature, feature) HEAD@{2}: commit: Update welcome.txt
31af010 HEAD@{3}: commit: Update info.txt
6feaa43 HEAD@{4}: checkout: moving from master to feature
6feaa43 HEAD@{5}: commit: Add welcome.txt
327bfea HEAD@{6}: commit (initial): initial commit
```

Cherry-pick

```bash
git cherry-pick 31af010
```

Confirm

```bash
git reflog
7ac2708 (HEAD -> master, origin/master) HEAD@{0}: cherry-pick: Update info.txt
6feaa43 HEAD@{1}: checkout: moving from feature to master
f2b3123 (origin/feature, feature) HEAD@{2}: commit: Update welcome.txt
31af010 HEAD@{3}: commit: Update info.txt
6feaa43 HEAD@{4}: checkout: moving from master to feature

# git log

sudo git log --oneline --graph --all -6
* 7ac2708 (HEAD -> master, origin/master) Update info.txt
| * f2b3123 (origin/feature, feature) Update welcome.txt
| * 31af010 Update info.txt
|/
* 6feaa43 Add welcome.txt
* 327bfea initial commit
```

- `327bfea` → `6feaa43` is the shared history: the initial commit, then "Add welcome.txt".
- `|/` is where the branches split at `6feaa43`.
- **`feature`** (right column) went on to `31af010` "Update info.txt", then `f2b3123` "Update welcome.txt".
- **`master`** (left column) has `7ac2708` "Update info.txt", your cherry-picked copy of `31af010`.

#### 29 Pull request using Gitea

Procedure of creating a pull request and assigning a reviewer before merging with master branch. Took part in UI mostly which was intuitive.
