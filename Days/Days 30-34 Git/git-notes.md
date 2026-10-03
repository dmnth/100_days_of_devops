
#### Hard reset - If one wishes to reset changed made to a specific branch

```bash
git reflog
6095e10 HEAD@{1}: commit: Test Commit10
c9f4a94 HEAD@{2}: commit: Test Commit9
c0d172e HEAD@{3}: commit: Test Commit8
7ba41d7 HEAD@{4}: commit: Test Commit7
41d5bcc HEAD@{5}: commit: Test Commit6
9b0515a HEAD@{6}: commit: Test Commit5
24cb728 HEAD@{7}: commit: Test Commit4
e8c1e0c HEAD@{8}: commit: Test Commit3
13672c2 HEAD@{9}: commit: Test Commit2
be94fd2 HEAD@{10}: commit: Test Commit1
d9c27f6 (HEAD -> master, origin/master, origin/HEAD) HEAD@{11}: commit: add data.txt file
e8f2283 HEAD@{12}: commit (initial): initial commit
# Here we point to a hash of a commit we want to revert to
git reset --hard d9c27f6
# Push changes to branch
git push --force
# Note that HEAD now points back to d9c27f6
d9c27f6 (HEAD -> master, origin/master, origin/HEAD) HEAD@{0}: reset: moving to d9c27f6
6095e10 HEAD@{1}: commit: Test Commit10
c9f4a94 HEAD@{2}: commit: Test Commit9
c0d172e HEAD@{3}: commit: Test Commit8
7ba41d7 HEAD@{4}: commit: Test Commit7
41d5bcc HEAD@{5}: commit: Test Commit6
9b0515a HEAD@{6}: commit: Test Commit5
24cb728 HEAD@{7}: commit: Test Commit4
e8c1e0c HEAD@{8}: commit: Test Commit3
13672c2 HEAD@{9}: commit: Test Commit2
be94fd2 HEAD@{10}: commit: Test Commit1
d9c27f6 (HEAD -> master, origin/master, origin/HEAD) HEAD@{11}: commit: add data.txt file
e8f2283 HEAD@{12}: commit (initial): initial commit
```

#### Day 31 - Stash

When you want to record the current state of the working directory and the index, but want to go back to a clean working directory. The command saves your local modifications away and reverts the working directory to match the `HEAD` commit.

```bash
natasha@ststor01 official]$ sudo git stash list  
stash@{0}: WIP on master: 4bbdc32 initial commit  
stash@{1}: WIP on master: 4bbdc32 initial commit  
[natasha@ststor01 official]$ sudo git stash apply stash@{1}  
On branch master  
Your branch is up to date with 'origin/master'.

Changes to be committed:  
(use "git restore --staged <file>..." to unstage)  
new file: welcome.txt

[natasha@ststor01 official]$ sudo git commit -m 'restore stash'  
[natasha@ststor01 official]$ sudo git reflog
[natasha@ststor01 official]$ sudo git log --oneline -2 # new commit on top of 4bbdc32
```

#### Day 32 - Rebase

We are asked to rebase `feature` branch with `master` branch without loosing any data from `feature` branch withou `merging`.

`git rebase` can be used to transplant a series of commits onto a different starting point.

```bash
sudo git reflog
fc4d66c (HEAD -> feature, origin/feature) HEAD@{0}: checkout: moving from master to feature
9e95572 (origin/master, master) HEAD@{1}: commit: Update info.txt
7fd2baf HEAD@{2}: checkout: moving from feature to master
fc4d66c (HEAD -> feature, origin/feature) HEAD@{3}: commit: Add new feature
7fd2baf HEAD@{4}: checkout: moving from master to feature
7fd2baf HEAD@{5}: commit (initial): initial commit
sudo git branch --list
* feature
  master
sudo git rebase master feature
Successfully rebased and updated refs/heads/feature.
sudo git log --oneline --graph --all --decorate
* 32c1515 (HEAD -> feature) Add new feature
* 9e95572 (origin/master, master) Update info.txt
| * fc4d66c (origin/feature, list) Add new feature
|/  
* 7fd2baf initial commit
[natasha@ststor01 news]$ sudo git branch -D list
Deleted branch list (was fc4d66c).
sudo git push --force origin feature
sudo git log --oneline --graph --all --decorate
# history is a single straight line with no merge commit
* 32c1515 (HEAD -> feature, origin/feature) Add new feature
* 9e95572 (origin/master, master) Update info.txt
* 7fd2baf initial commit
```

#### Day 33 - Merge conflicts

How to resolve:

- Decide not to merge. The only clean-ups you need are to reset the index file to the `HEAD` commit to reverse 2. and to clean up working tree changes made by 2. and 3.; `git` `merge` `--abort` can be used for this.

- Resolve the conflicts. Git will mark the conflicts in the working tree. Edit the files into shape and `git` `add` them to the index. Use `git` `commit` or `git` `merge` `--continue` to seal the deal. The latter command checks whether there is a (interrupted) merge in progress before calling `git` `commit`.

```bash
git status
On branch master
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)

nothing to commit, working tree clean
git push
To http://gitea:3000/sarah/story-blog.git
 ! [rejected]        master -> master (fetch first)
error: failed to push some refs to 'http://gitea:3000/sarah/story-blog.git'
hint: Updates were rejected because the remote contains work that you do not
hint: have locally. This is usually caused by another repository pushing to
hint: the same ref. If you want to integrate the remote changes, use
hint: 'git pull' before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
git pull
remote: Enumerating objects: 4, done.
remote: Counting objects: 100% (4/4), done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
Unpacking objects: 100% (3/3), 354 bytes | 354.00 KiB/s, done.
From http://gitea:3000/sarah/story-blog
   3487b34..77f1c12  master     -> origin/master
Auto-merging story-index.txt
CONFLICT (add/add): Merge conflict in story-index.txt
Automatic merge failed; fix conflicts and then commit the result.
# Edit the file 
vi story-index.txt
# Add amd commit
max@ststor01 story-blog]$ git add story-index.txt 
[max@ststor01 story-blog]$ git commit -m 'merge conflict'
[master 44244c2] merge conflict
[max@ststor01 story-blog]$ git pull
Already up to date.
# Push changes
[max@ststor01 story-blog]$ git push
Username for 'http://gitea:3000': max
Password for 'http://max@gitea:3000': 
Enumerating objects: 10, done.
Counting objects: 100% (10/10), done.
Delta compression using up to 16 threads
Compressing objects: 100% (7/7), done.
Writing objects: 100% (7/7), 1.08 KiB | 1.08 MiB/s, done.
Total 7 (delta 2), reused 0 (delta 0), pack-reused 0 (from 0)
remote: . Processing 1 references
remote: Processed 1 references in total
To http://gitea:3000/sarah/story-blog.git
   77f1c12..44244c2  master -> master
# Origin master moved from 77f1c12 to 44244c2 which is a merge conflict commit
git log --oneline --graph --all --decorate -5
*   44244c2 (HEAD -> master, origin/master, origin/HEAD) merge conflict
|\  
| * 77f1c12 Added Index
* | 32f19eb Added the fox and grapes story
|/  
*   3487b34 Merge branch 'story/frogs-and-ox'
|\  
| * 38564c5 Completed frogs-and-ox story

```

#### Day 34 - Hook

Hooks are programs you can place in a hooks directory to trigger actions at certain points in git’s execution. Hooks that don’t have the executable bit set are ignored.

Task is to create a hook that is triggered after the `master` branch receives an update. Hook should run `git tag` anc create a tag with current date.

```bash
# below will produce date in YYYY-MM-DD
export GIT_COMMITER_DATE=$(date +%F)
cd /opt/news.git/hooks/
mv post-update.sample post-update
vi post-update
# Added `exec git tag release-$GIT_COMMITER_DATE`
cd /usr/src/kodekloudrepos/news/
[natasha@ststor01 news]$ git status
On branch feature
nothing to commit, working tree clean
[natasha@ststor01 news]$ git branch --list
* feature
  master
git switch master
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
git merge feature master
Updating 7ac0129..597d006
Fast-forward
 feature.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 feature.txt
[natasha@ststor01 news]$ git push
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/news.git
   7ac0129..597d006  master -> master
ls /opt/news.git/refs/tags/
release-2026-03-10
```

More appropriate post-update hook code sample:

```bash
#!/bin/bash
# Runs in the bare repo after a push; $@ = list of updated refs

for ref in "$@"; do
    if [ "$ref" = "refs/heads/master" ]; then
        TAG="release-$(date +%F)"
        if git rev-parse -q --verify "refs/tags/$TAG" >/dev/null; then
            echo "Tag $TAG already exists, skipping"
        else
            git tag "$TAG" refs/heads/master
            echo "Created tag $TAG"
        fi
    fi
done
```
