# Mergeability Part 2 — Merging on the Server Side, Pull Requests, and a Brief Introduction to the Test Merge

---

## 1) Authenticate to GHES and Push the Repo

```sh
# Log in the GHES host (token or web flow)
gh auth login --hostname maxiscoding28-p0jira.ghe-test.net

# Create the remote repo on GHES and push the current directory
GH_HOST=maxiscoding28-p0jira.ghe-test.net \
  gh repo create ghe-admin/mergeability-demo --private --source . --push --remote origin

# Verify remotes
git remote -v
```

---

## 2) Inspect on the Server (Bare Repo)

```sh
# SSH into the repo host (spoke) and drop into the bare repo
ghe-spokesctl ssh ghe-admin/mergeability-demo

# Basic history graph (server-side)
git log --oneline --decorate --graph --all --boundary
```

```sh
# Show packed layout
ls -lah objects/ objects/pack/

# Enumerate reachable objects (type + short id) — works with packs
git rev-list --objects --all | cut -d' ' -f1 | \
while read id; do
  printf "%-6s %.8s\n" "$(git cat-file -t "$id")" "$id"
done
```

---

### 3) Same Merge Flow on the Client

```sh
# Create branch, change, commit
git checkout -b feature-branch
echo "Server-side demo change" >> README.md
git add README.md
git commit -m "Feature: server-side demo change"

# Back to main, different change, commit
git checkout main
echo "Main-side change" >> main.txt
git add main.txt
git commit -m "Main: add main.txt"

# Merge (creates a merge commit)
git merge feature-branch -m "Merge feature-branch into main"

# View graph locally
git log --oneline --decorate --graph --all --boundary
```

Push the result:

```sh
git push origin main
```

---

### 4) Inspect the New State on the Server

```sh
ghe-spokesctl ssh ghe-admin/mergeability-demo

git log --oneline --decorate --graph --all --boundary

# Quick inventory again
git rev-list --objects --all | cut -d' ' -f1 | \
while read id; do
  printf "%-6s %.8s\n" "$(git cat-file -t "$id")" "$id"
done
```

### Pull Requestsd

---

## 1) Create PR with a change

```sh
# New branch, change, commit, push
git checkout -b pr-demo
echo "Change for PR demo" >> PR_DEMO.md
git add PR_DEMO.md
git commit -m "PR demo: add PR_DEMO.md"
git push -u origin pr-demo

# Open PR (base=main, head=pr-demo)
gh pr create --base main --head pr-demo \
  --title "PR demo: pr-demo → main" \
  --body "Adds PR_DEMO.md; used to demonstrate PR flow."
```

---

## 2) Show mergeability via API (REST) and GraphQL

```sh
# Capture PR number
PR=$(gh pr view --json number -q .number)
OWNER=ghe-admin
REPO=mergeability-demo
export GH_HOST=maxiscoding28-p0jira.ghe-test.net
```

**REST (mergeability + computed state):**
```sh
gh api -X GET "repos/$OWNER/$REPO/pulls/$PR" -q \
  '{number: .number, state: .state, mergeable: .mergeable, mergeable_state: .mergeable_state, sha: .head.sha}'
```

Here’s the addition you can slot in right after your GraphQL snippet to demonstrate how mergeability flips when a conflict is introduced:  

---

**GraphQL (authoritative mergeability + state):**
```sh
gh api graphql -f query='
  query($owner:String!,$repo:String!,$pr:Int!) {
    repository(owner:$owner,name:$repo) {
      pullRequest(number:$pr) {
        number
        state
        mergeable
        mergeStateStatus
        headRefName
        baseRefName
      }
    }
  }' -f owner="$OWNER" -f repo="$REPO" -F pr=$PR
```

---

### 2.1) Create a conflict on the `main` side  

```sh
# Switch to main
git checkout main

# Edit the same file the PR branch touched
echo "Conflicting line from main" >> PR_DEMO.md
git add PR_DEMO.md
git commit -m "Main: introduce conflict with pr-demo branch"

# Push change to server
git push origin main
```

Now re-run the GraphQL or REST query to observe how mergeability changes:  

```sh
gh api graphql -f query='
  query($owner:String!,$repo:String!,$pr:Int!) {
    repository(owner:$owner,name:$repo) {
      pullRequest(number:$pr) {
        number
        state
        mergeable
        mergeStateStatus
      }
    }
  }' -f owner="$OWNER" -f repo="$REPO" -F pr=$PR
```

Typical result:
```
mergeable: CONFLICTING
mergeStateStatus: DIRTY
```

---

## 3) How GitHub decides “can this be merged?”

- **Content side:** Compute a **synthetic merge** of `base` + `head` (no refs change) and detect conflicts.  
- **Result surfaces as:**  
  - **GraphQL:** `mergeable`, `mergeStateStatus`  
  - **REST:** `mergeable`, `mergeable_state`  
  - **UI:** the merge box shows “merge conflict” or “can be merged cleanly.”  

---

## 4) Inspect the internals (test-merge refs)

```sh
# Fetch the PR "head" and the synthesized "merge" commit refs created by GitHub
git fetch origin \
  "pull/$PR/head:refs/pull/$PR/head" \
  "pull/$PR/merge:refs/pull/$PR/merge"

# Quick peek: the merge preview commit (should have two parents)
git log --oneline --decorate --graph --boundary -1 refs/pull/$PR/merge

# Inspect commit headers/parents/trees
git show --no-patch --pretty=raw refs/pull/$PR/merge
```