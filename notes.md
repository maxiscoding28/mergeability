- What are Git Objects
    - Blob
    - Tree
    - Ref
    - Tag


### The First Commit
```sh
# Create a new Git repo and add a file
mkdir mergeability-demo
cd mergeability-demo
git init
echo "# Mergeability Demo" > README.md

# Look inside .git/objects — should only have the default structure at first
ls -R .git/objects/

# Stage the file: this writes a blob object into the object database
git add README.md

# Confirm the file is staged
git status

# Inspect the object database

## A new object file appears — its name is split into <2-char dir>/<38-char file>
ls -R .git/objects/

## That file is a zlib-compressed representation of the object
cat .git/objects/<dir>/<id-part>

# Ask Git what type of object it is (blob, tree, commit, or tag)
git cat-file -t <object-id>

# Pretty-print / inflate the object to see its contents
git cat-file -p <object-id>

# Commit the staged blob into history — this creates a tree and a commit object
git commit -m "Adding Readme"

# Inspect the object database again — you’ll now see the new tree and commit objects
ls -R .git/objects/

# Dynamically reconstruct full object IDs from their directory and file names
find .git/objects -type f ! -path "*/pack/*" ! -path "*/info/*" \
  -exec sh -c 'echo "$(basename $(dirname {}))$(basename {})"' \;

# Show the type of each object (blob, tree, commit, etc.)
find .git/objects -type f ! -path "*/pack/*" ! -path "*/info/*" \
  -exec sh -c 'git cat-file -t "$(basename "$(dirname "$1")")$(basename "$1")"' sh {} \;

# Show the pretty-printed contents of each object
find .git/objects -type f ! -path "*/pack/*" ! -path "*/info/*" \
  -exec sh -c 'git cat-file -p "$(basename "$(dirname "$1")")$(basename "$1")"' sh {} \;

# Show type and contents together for easier reading
find .git/objects -type f ! -path "*/pack/*" ! -path "*/info/*" \
  -exec sh -c '
    f="$1"
    id="$(basename "$(dirname "$f")")$(basename "$f")"
    printf "== %s (%s) ==\n" "$id" "$(git cat-file -t "$id")"
    git cat-file -p "$id"
    printf "\n"
  ' sh {} \;
```

### Merging a Change
```sh
# Create a new branch from the current commit
git branch feature-branch
git checkout feature-branch

# Make a change on this branch (add a line to the README)
echo "This line is from the feature branch." >> README.md

# Stage and commit the change — this writes a new blob (the updated README),
# a new tree object, and a new commit object for feature-branch
git add README.md
git commit -m "Update README on feature branch"

# Show which objects are unique to feature-branch compared to main
git rev-list --objects main..feature-branch

# Print type + contents of only those unique objects
git rev-list --objects main..feature-branch | cut -d' ' -f1 | \
  while read id; do
    printf "== %s (%s) ==\n" "$id" "$(git cat-file -t "$id")"
    git cat-file -p "$id"
    echo
  done

# Switch back to main and add a *different* commit so histories diverge
git checkout main
echo "This line is from main branch." >> main.txt
git add main.txt
git commit -m "Add main.txt on main branch"

# At this point histories have diverged (each branch has a unique commit)
git log --oneline --decorate --graph --all --boundary

# (Optional) show the merge-base (common ancestor)
git merge-base main feature-branch

# Merge feature-branch into main — Git must create a *new merge commit*
git merge feature-branch -m "Merge branch 'feature-branch' into main"

# Verify: new merge commit exists, with two parents
git log --oneline --decorate --graph --all --boundary
git cat-file -p HEAD

# Inspect the object database again — you’ll see a new commit object (the merge commit)
find .git/objects -type f ! -path "*/pack/*" ! -path "*/info/*" \
  -exec sh -c '
    f="$1"
    id="$(basename "$(dirname "$f")")$(basename "$f")"
    printf "== %s (%s) ==\n" "$id" "$(git cat-file -t "$id")"
    git cat-file -p "$id"
    printf "\n"
  ' sh {} \;

# Clean up by deleting the now-merged topic branch
git branch -d feature-branch
```

### Merge Conflict
```sh
# Start fresh from main
git checkout main

# Create a new branch
git branch conflict-branch
git checkout conflict-branch

# Modify README in conflict-branch
echo "This line is from conflict-branch." >> README.md
git add README.md
git commit -m "Edit README on conflict-branch"

# Switch back to main and make a conflicting edit
git checkout main
echo "This line is from main branch (conflict)." >> README.md
git add README.md
git commit -m "Edit README on main branch"

# Histories have diverged
git log --oneline --decorate --graph --all --boundary

# Attempt to merge conflict-branch into main
git merge conflict-branch

# Git will report a merge conflict in README.md
git status

# Inspect the conflict markers inside README.md
cat README.md

# Choose ours
git checkout --ours README.md
git add README.md

# Complete the merge with a commit
git commit -m "Resolve merge conflict between main and conflict-branch"

# Show final history with merge commit
git log --oneline --decorate --graph --all --boundary
```

# What did we learn?
- Git's internal data is composed of 
  - blob (which represent file contents)
  - Trees which point to blobs
  - And commits which point to trees

- When you attempt to merge one branch into another
- Git needs to reconcile the changes from the two branches into one set of changes. Sometimes this works and can be completed cleanly. Sometimes the two sets of changes conflict and the changes cant be merged without further user intervention. Merge conflict.

### How Does This Work on the Server Side
- Push repo from main
- Make change locally
- Merge into Local Main
- Push to Main
- Inspect commands on server side to show objects

### The Pull Request
- Set Require Pull Request Review Before Merging
- Show the Error
- Pull Request, helpful UI to propose and review merges you would like to make onto Main
- Create change on new branch
- Push to remote branch
- Create Pull Request
- Show Pull Requst Merge Button, API, GraphQL queries
- Show a conflict (cannot merge), show success can merge)
- Show how this gets created on server side with Test Merge
- CPRMC

### The Problems with Test Merges
- Performing constant attempts to merge and then discarding objects
- Unreachable vs Reachable Objects
- Spamming requests, kick off these jobs
- Unreachable object growth for monorepos spamming checks and fast moving head
- Git nw-gc --pristine

### What To Do
- reduce API requests
- Graphql remove merge
- Fewer open PRs
- Feature requests

### Addendum
- CPRMC no more, Same problems exist

### Takeaways
- Deeper understanding of how git merges work
- Deeper understanding of concept of a test merge, and how it works in context of Github.com and GHES.
- Problems created by test merge and potential mitigations




Demo: 
- Create a repo
- Create each of these objects and inspect as I make them (on local repo)
- How does merge work
- Create a conflict


- How does GHES keep track of this? Mergeability Checks
 gheboot-maxiscoding28-3_15_6-single-node-20250828151816
    - API
    - GraphQL
    - Pull Request
    - Conditions for Kicking off a Mergeability Check


- How does merge check work
    - CPRMC, Ref,
    - What happens to that ref? Unreachable


- Spam unreachable
    - Network Mainteance doesn't clean it up
    - Fill up disk
    - git nw-gc --pristine


- Addendum, CPRMC Reloaded