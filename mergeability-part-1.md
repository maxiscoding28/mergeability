# Mergeability Part 1 — Merging on the Client Side

---

## Introduction

This is Max Winslow.  

I’ve been working as a Customer Reliability Engineer since August 2024 for some of our Premium Plus support customers

- Many of the customers I support run GitHub Enterprise Server (GHES) and share two common traits:  
  1. They typically work with very large repositories that have lots of open pull requests.  
  2. They experience a rapid and high volume of changes landing on their `main` branch throughout the day.

- That combination of factors creates significant challenges around mergeability checks.  

- There’s a whole constellation of terms tied to mergeability checks. You’ll often hear about “test merges,” and you may recognize the acronym **CPRMC** (Create Pull Request Merge Commit job). While that job is being phased out, it’s still a term you’ll see used heavily in Slack or Issues or old tickets.  

- My goal with this talk is to provide newer CREs and support engineers an introduction to the mergeability checking system of GitHub Enterprise Server.

- I’ll cover the core functionality, highlight some of the most common problems in this system, and share ways to think about mitigating those issues.  


## Git is Just Objects: The First Commit

To start, let's work in local Git to refresh our memories on the internals of Git and the mechanics of a merge.  
We’ll initialize a repo, create a file, stage it, commit it, and then inspect Git’s internal object storage.

```sh
# Initialize repo and create README
mkdir mergeability-demo && cd mergeability-demo && git init

# Look into the .git directory (objects is currently empty)
ls -lahR .git/objects

# Create our first file
echo "# Mergeability Demo" > README.md

# Stage it so Git starts tracking it
git add README.md

# Now if we peek into the objects directory, a new directory exists
ls -lah .git/objects

# That directory contains a compressed version of the README file
ls -lah .git/objects/70
```

At this point Git has created its first **blob object**, representing the file contents.  
We can verify this using `git cat-file`:

```sh
# Construct the full SHA by combining directory and filename
git cat-file -t <object-id>   # show type (blob)
git cat-file -p <object-id>   # show contents
```

Git stores objects under `.git/objects/` by splitting the SHA-1:
- First 2 characters → directory  
- Remaining 38 characters → file  

Example:  
`f8e123...` → `.git/objects/f8/e123...`  
Recombine dir + file = full 40-char object ID.

---

### Committing

```sh
# Commit the change
git commit -m "Initial commit"

# After committing, we see several new objects:
# - the blob (file contents) we already saw
# - a tree object (snapshot of directory structure pointing to the blob)
# - a commit object (pointing to the tree, and possibly to a parent)
```

When you run `git commit`:
- Git already has the **blob** (file).  
- It creates a **tree** mapping filenames to blob IDs.  
- It creates a **commit** referencing that tree.  

The chain looks like:

```
commit → tree → blob
```

---

### 1.3) After Commit  

When you run `git commit`:  
- Git already has a **blob** (the file contents).  
- It creates a **tree** that references that blob (mapping filename → blob ID).  
- It creates a **commit** that references the tree (and possibly a parent commit).  

So the chain looks like:  

```
commit → tree → blob
```  

```sh
git commit -m "Adding Readme"
ls -R .git/objects/
git rev-list --objects --all | cut -d' ' -f1 | \
while read id; do
  printf "== %s (%s) ==\n" "$id" "$(git cat-file -t "$id")"
  git cat-file -p "$id" 2>/dev/null || true
  printf "\n"
done
```

---

## 2) Merging a Change

Here we create divergent history by committing on a feature branch and on main, then merging.

```sh
# Branch off main and edit README
git checkout -b feature-branch
echo "This line is from the feature branch." >> README.md
git add README.md
git commit -m "Update README on feature branch"
```

### 2.1) Divergence on Main

```sh
git checkout main
echo "This line is from main branch." >> main.txt
git add main.txt
git commit -m "Add main.txt on main branch"
```

At this point both branches have unique commits:

```sh
git log --oneline --decorate --graph --all --boundary
```

### 2.2) Merge with Commit

```sh
git merge feature-branch -m "Merge branch 'feature-branch' into main"
git log --oneline --decorate --graph --all --boundary
git cat-file -p HEAD
```

### 2.3) Inspect and Clean Up  

Again, enumerate all objects via `git rev-list` and show their type and contents.  

```sh
git rev-list --objects --all | cut -d' ' -f1 | \
while read id; do
  printf "== %s (%s) ==\n" "$id" "$(git cat-file -t "$id")"
  git cat-file -p "$id" 2>/dev/null || true
  printf "\n"
done

git branch -d feature-branch
```

---

## 3) Merge Conflict

Now let’s force an actual conflict and resolve it on the CLI.

```sh
# Create a new branch and edit README
git checkout -b conflict-branch
echo "This line is from conflict-branch." >> README.md
git add README.md
git commit -m "Edit README on conflict-branch"
```

### 3.1) Conflicting Change on Main

```sh
git checkout main
echo "This line is from main branch (conflict)." >> README.md
git add README.md
git commit -m "Edit README on main branch"
```

### 3.2) Attempt Merge

```sh
git merge conflict-branch
git status
cat README.md   # shows conflict markers
```

### 3.3) Resolve Conflict and Commit

```sh
git checkout --ours README.md
git add README.md
git commit -m "Resolve merge conflict between main and conflict-branch"
```

### 3.4) Confirm History

```sh
git log --oneline --decorate --graph --all --boundary
```

---

## 4) What Did We Learn?

- **Git objects**:  
  - **Blob**: file contents  
  - **Tree**: directory structure linking blobs  
  - **Commit**: points to a tree and parents  

- **Merging**:  
  - If both branches diverged, Git makes a merge commit with multiple parents.  

- **Merge conflicts**:  
  - Occur when the same part of a file is modified differently.  
  - Git marks conflicts and prevents the merge.  
  - You need to resolve the conflict before completing the merge (edit file, stage, then commit).  