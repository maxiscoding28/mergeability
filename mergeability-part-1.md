# The First Commit

We start by initializing a repo, staging a file, and inspecting Git’s internal object storage.

```sh
# Initialize repo and create README
mkdir mergeability-demo
cd mergeability-demo
git init
echo "# Mergeability Demo" > README.md
```

--- 

### Object IDs

Git stores objects under `.git/objects/` by splitting the SHA-1:  
- first 2 chars → directory  
- remaining 38 chars → file  

Example:  
`f8e123...` → `.git/objects/f8/e123...`  

Recombine dir + file = full 40-char object ID.  

```sh
git cat-file -t <object-id>   # show type
git cat-file -p <object-id>   # show contents
```

---

### Explore All Objects  

The `find` command:  
- walks `.git/objects` (ignoring pack/info)  
- reconstructs object IDs from dir+file  
- prints type + contents for each  

```sh
find .git/objects -type f ! -path "*/pack/*" ! -path "*/info/*" \
  -exec sh -c '
    id="$(basename "$(dirname "$1")")$(basename "$1")"
    printf "== %s (%s) ==\n" "$id" "$(git cat-file -t "$id")"
    git cat-file -p "$id"
    printf "\n"
  ' sh {} \;
```

---

### After Commit  

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
find .git/objects -type f ! -path "*/pack/*" ! -path "*/info/*" \
  -exec sh -c '
    id="$(basename "$(dirname "$1")")$(basename "$1")"
    printf "== %s (%s) ==\n" "$id" "$(git cat-file -t "$id")"
    git cat-file -p "$id"
    printf "\n"
  ' sh {} \;
```

---

## Merging a Change

Here we create divergent history by committing on a feature branch and on main, then merging.

```sh
# Branch off main and edit README
git checkout -b feature-branch
echo "This line is from the feature branch." >> README.md
git add README.md
git commit -m "Update README on feature branch"
```

### Divergence on Main

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

### Merge with Commit

```sh
git merge feature-branch -m "Merge branch 'feature-branch' into main"
git log --oneline --decorate --graph --all --boundary
git cat-file -p HEAD
```

### Inspect and Clean Up

```sh
find .git/objects -type f ! -path "*/pack/*" ! -path "*/info/*" \
  -exec sh -c '
    f="$1"
    id="$(basename "$(dirname "$f")")$(basename "$f")"
    printf "== %s (%s) ==\n" "$id" "$(git cat-file -t "$id")"
    git cat-file -p "$id"
    printf "\n"
  ' sh {} \;

git branch -d feature-branch
```

---

## Merge Conflict

Now let’s force an actual conflict and resolve it on the CLI.

```sh
# Create a new branch and edit README
git checkout -b conflict-branch
echo "This line is from conflict-branch." >> README.md
git add README.md
git commit -m "Edit README on conflict-branch"
```

### Conflicting Change on Main

```sh
git checkout main
echo "This line is from main branch (conflict)." >> README.md
git add README.md
git commit -m "Edit README on main branch"
```

### Attempt Merge

```sh
git merge conflict-branch
git status
cat README.md   # shows conflict markers
```

### Resolve Conflict and Commit

```sh
git checkout --ours README.md
git add README.md
git commit -m "Resolve merge conflict between main and conflict-branch"
```

### Confirm History

```sh
git log --oneline --decorate --graph --all --boundary
```

---

## What Did We Learn?

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