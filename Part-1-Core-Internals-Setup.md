# 🔧 Git Senior Level Guide — Part 1: Core Internals, Architecture & Setup

> **Audience:** Developer jo production-level Git use karna chahta hai  
> **Language:** Hinglish  
> **Goal:** Zero confusion, har concept crystal clear

---

## Table of Contents

1. [Git Kya Hai — Actually (Internals)](#1-git-kya-hai--actually-internals)
2. [Git ka Architecture](#2-git-ka-architecture)
3. [Git Objects — The Foundation](#3-git-objects--the-foundation)
4. [Git References (Refs)](#4-git-references-refs)
5. [Index / Staging Area — Deep Dive](#5-index--staging-area--deep-dive)
6. [Git Config — Master Setup](#6-git-config--master-setup)
7. [.gitignore — Advanced Patterns](#7-gitignore--advanced-patterns)
8. [Git Aliases — Productivity Boost](#8-git-aliases--productivity-boost)

---

## 1. Git Kya Hai — Actually (Internals)

Git ek **content-addressable filesystem** hai jiske upar ek VCS (Version Control System) build kiya gaya hai.

### ❌ Common Misconception
> "Git file changes track karta hai" — WRONG

### ✅ Reality
> Git **snapshots** store karta hai — har commit ek complete snapshot hota hai, diff nahi.

```
# Yeh sochna band karo:
v1 → v2 → v3  (changes)

# Yeh sochna shuru karo:
v1 (full snapshot) → v2 (full snapshot) → v3 (full snapshot)
# Unchanged files ke liye Git sirf ek pointer store karta hai — space waste nahi hota
```

### Git vs Other VCS

| Feature | SVN/CVS (old) | Git |
|---------|---------------|-----|
| Storage | Delta (diffs) | Snapshots (with deduplication) |
| History | Server pe | Local copy complete |
| Branching | Slow, expensive | Instant (just a pointer) |
| Offline work | Not possible | Fully possible |
| Integrity | Basic | SHA-1/SHA-256 cryptographic hash |

---

## 2. Git ka Architecture

```
Working Directory  →  Staging Area (Index)  →  Local Repo  →  Remote Repo
      │                      │                      │                │
   (edit files)          (git add)             (git commit)    (git push)
```

### Teeno "Trees" of Git

```
┌─────────────────────────────────────────────────┐
│  HEAD → (points to current branch/commit)        │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Working  │  │  Index   │  │  Repository  │   │
│  │Directory │  │(Staging) │  │   (Objects)  │   │
│  └──────────┘  └──────────┘  └──────────────┘   │
└─────────────────────────────────────────────────┘
```

### `.git` Directory Structure

```bash
.git/
├── HEAD              # Current branch ka pointer
├── config            # Repo-level config
├── index             # Staging area (binary file)
├── COMMIT_EDITMSG    # Last commit message
├── MERGE_HEAD        # Merge in progress hoga to yahan
├── MERGE_MSG         # Merge commit message
├── objects/          # ALL data yahan store hota hai
│   ├── pack/         # Compressed objects
│   └── info/
├── refs/             # All references (branches, tags)
│   ├── heads/        # Local branches
│   ├── remotes/      # Remote tracking branches
│   └── tags/         # Tags
├── hooks/            # Scripts that run on events
└── logs/             # Reflog data
```

---

## 3. Git Objects — The Foundation

Git ke 4 types ke objects hote hain, sab `.git/objects/` mein store hote hain.

### Object Types

```
1. blob    → File content store karta hai
2. tree    → Directory structure store karta hai (filenames + permissions + blob refs)
3. commit  → Snapshot + metadata (author, message, parent commits, tree ref)
4. tag     → Annotated tag data
```

### Kaise kaam karta hai internally?

```bash
# Ek file ka blob object dekho
echo "hello git" | git hash-object --stdin
# Output: 8d0e41234f24b6da002d962a26c2495ea16a425f

# Manually object create karo
echo "hello git" | git hash-object -w --stdin
# -w flag = actually write to .git/objects/

# Object ka content dekho
git cat-file -p 8d0e41234f24b6da002d962a26c2495ea16a425f
# Output: hello git

# Object ka type dekho
git cat-file -t 8d0e41234f24b6da002d962a26c2495ea16a425f
# Output: blob
```

### Tree Object samjho

```bash
# Current commit ki tree dekho
git cat-file -p HEAD^{tree}
# Output:
# 100644 blob a1b2c3... README.md
# 100644 blob d4e5f6... index.js
# 040000 tree 789abc... src/

# Permissions ka matlab:
# 100644 = regular file
# 100755 = executable file
# 120000 = symbolic link
# 040000 = directory (tree)
```

### Commit Object dekho

```bash
git cat-file -p HEAD
# Output:
# tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904
# parent a1b2c3d4e5f6...
# author John Doe <john@example.com> 1703001234 +0530
# committer John Doe <john@example.com> 1703001234 +0530
#
# feat: add login functionality
```

### SHA-1 Hash — Content Addressable

```
SHA-1 Hash = hash(object_type + content_size + content)

Same content → ALWAYS same hash (deterministic)
Different content → Different hash (collision extremely rare)
```

> 💡 **Fact:** Git SHA-1 hash 40 characters ka hota hai. Git usually sirf 7 characters use karta hai (short hash) kyunki woh enough unique hote hain.

### Object Storage Path

```
Hash: 4b825dc642cb6eb9a060e54bf8d69288fbee4904
Path: .git/objects/4b/825dc642cb6eb9a060e54bf8d69288fbee4904
              ↑↑   ↑↑
         First 2    Remaining 38 chars
```

---

## 4. Git References (Refs)

References = human-readable names for SHA-1 hashes

```bash
# Branch ek simple text file hota hai:
cat .git/refs/heads/main
# Output: a1b2c3d4e5f6789...  (just a SHA-1 hash!)

# HEAD bhi ek file hai:
cat .git/HEAD
# Output: ref: refs/heads/main  (symbolic ref)
# OR: a1b2c3d4...  (detached HEAD state mein)
```

### Reference Types

```bash
# Local branches
.git/refs/heads/main
.git/refs/heads/feature/login

# Remote tracking branches
.git/refs/remotes/origin/main
.git/refs/remotes/origin/develop

# Tags
.git/refs/tags/v1.0.0

# Special refs
HEAD      → current position
ORIG_HEAD → before merge/rebase
FETCH_HEAD → last fetched
MERGE_HEAD → branch being merged
```

### Packed Refs

```bash
# Bahut saare refs hone par Git ek single file mein pack karta hai:
cat .git/packed-refs
# Output:
# # pack-refs with: peeled fully-peeled sorted
# a1b2c3d4 refs/heads/main
# d4e5f678 refs/tags/v1.0.0
# ^e5f6789a   ← peeled tag (actual commit)
```

---

## 5. Index / Staging Area — Deep Dive

Index ek **binary file** hai (`.git/index`) jo working directory aur repository ke beech ka bridge hai.

```bash
# Index ka contents dekho (human readable)
git ls-files --stage
# Output:
# 100644 a1b2c3d4... 0  README.md
# 100644 d4e5f678... 0  src/index.js
#         ↑ hash    ↑ stage number (0=normal, 1=base, 2=ours, 3=theirs during conflict)
```

### Staging Area ke Powers

```bash
# Partial file stage karo (hunk by hunk)
git add -p filename.js
# y = stage this hunk
# n = skip
# s = split into smaller hunks
# e = manually edit hunk
# q = quit

# Interactive staging
git add -i

# Sirf tracked files stage karo (new files exclude)
git add -u

# Everything stage karo
git add .
git add -A  # .git/index update + new files + deleted files

# Kya difference hai?
# git add .  → current directory se sab (new + modified + deleted)
# git add -A → entire repo se sab
# git add -u → only tracked files (modified + deleted, NO new files)
```

### Unstage karna

```bash
# Modern way (Git 2.23+)
git restore --staged filename.js

# Old way
git reset HEAD filename.js

# Sab unstage karo
git restore --staged .
```

---

## 6. Git Config — Master Setup

Config 3 levels pe hota hai (priority: local > global > system):

```bash
# System level (sabke liye, /etc/gitconfig)
git config --system user.name "Company Bot"

# Global level (~/.gitconfig)
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Local level (current repo, .git/config) — highest priority
git config --local user.email "work@company.com"
```

### Production-Ready Global Config

```bash
# ~/.gitconfig ya git config --global ile karo

# Identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global user.signingkey "GPG_KEY_ID"  # commit signing

# Core settings
git config --global core.editor "code --wait"  # VS Code
git config --global core.autocrlf input          # Linux/Mac
git config --global core.autocrlf true           # Windows
git config --global core.whitespace fix          # trailing whitespace fix
git config --global core.fileMode false          # file permission changes ignore

# Default branch
git config --global init.defaultBranch main

# Pull behavior (IMPORTANT!)
git config --global pull.rebase true   # pull = fetch + rebase (recommended)
# OR
git config --global pull.ff only       # only fast-forward, never merge commit

# Push behavior
git config --global push.default current         # push to same-name remote branch
git config --global push.autoSetupRemote true    # auto set upstream (Git 2.37+)

# Diff/Merge tools
git config --global merge.tool vscode
git config --global diff.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# Rebase settings
git config --global rebase.autoStash true        # stash before rebase automatically
git config --global rebase.autoSquash true       # fixup! commits auto-squash

# Credential storage
git config --global credential.helper store      # Linux (plain text, not secure)
git config --global credential.helper osxkeychain  # Mac
git config --global credential.helper manager   # Windows (Git Credential Manager)

# Log formatting
git config --global log.date relative           # "2 days ago" format
git config --global format.pretty "%C(yellow)%h%Creset %s %C(green)(%cr)%Creset %C(blue)<%an>%Creset"

# Color
git config --global color.ui auto

# Safety
git config --global fetch.prune true             # dead remote branches auto-delete
```

### Config File Directly Edit Karo

```bash
git config --global --edit
# Opens ~/.gitconfig in your editor
```

### Conditional Config (Work vs Personal)

```ini
# ~/.gitconfig
[user]
    name = Your Name
    email = personal@gmail.com

[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work

# ~/.gitconfig-work
[user]
    email = you@company.com
    signingkey = WORK_GPG_KEY
```

---

## 7. .gitignore — Advanced Patterns

```bash
# Basic patterns
*.log           # sab .log files
!important.log  # lekin important.log nahi
/build          # sirf root level build directory
build/          # koi bhi build directory
**/logs         # koi bhi directory mein logs
doc/**/*.txt    # doc directory mein kisi bhi depth pe .txt files

# Specific examples
node_modules/
.env
.env.local
.env.*.local
dist/
coverage/
*.DS_Store      # Mac
Thumbs.db       # Windows
*.orig          # merge conflict originals
```

### .gitignore Priority Order

```
1. .git/info/exclude       → sirf local machine ke liye, repo mein nahi jaata
2. ~/.config/git/ignore    → global ignore (sab repos ke liye)
3. .gitignore files        → repo mein committed
```

### Global .gitignore Setup

```bash
git config --global core.excludesFile ~/.gitignore_global

# ~/.gitignore_global
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db
*.swp       # Vim temp files
*.swo
.idea/      # IntelliJ
.vscode/    # VS Code (optional, team prefer kare to repo mein)
```

### Already Tracked File ko Ignore Karna

```bash
# Problem: file already committed hai, ab ignore karna chahte ho
# Solution:
git rm --cached filename.txt          # single file
git rm --cached -r node_modules/      # directory

# Ab .gitignore mein add karo
echo "node_modules/" >> .gitignore
git add .gitignore
git commit -m "chore: ignore node_modules"
```

### .gitignore Debug Karna

```bash
# Pata karo koi file ignore kyun ho rahi hai
git check-ignore -v filename.txt
# Output: .gitignore:3:*.txt  filename.txt
#                  ↑ line number

# Kaunsi files ignored hain dekho
git status --ignored

# Force add an ignored file (when you really need to)
git add -f ignored-file.txt
```

---

## 8. Git Aliases — Productivity Boost

```bash
# ~/.gitconfig [alias] section mein ya git config --global alias.<name> "<command>"

git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "restore --staged"
git config --global alias.undo "reset HEAD~1 --mixed"
git config --global alias.alias "config --get-regexp alias"  # list all aliases

# Powerful log alias
git config --global alias.alog "log --graph --abbrev-commit --decorate --format=format:'%C(bold blue)%h%C(reset) - %C(bold green)(%ar)%C(reset) %C(white)%s%C(reset) %C(dim white)- %an%C(reset)%C(bold yellow)%d%C(reset)' --all"

# Cleanup stale branches
git config --global alias.cleanup "!git branch --merged | grep -v '\\*\\|main\\|develop' | xargs -n 1 git branch -d"

# Show files changed in last commit
git config --global alias.changed "show --stat HEAD"

# Whoops alias
git config --global alias.whoops "reset HEAD~1 --soft"
```

### Shell Functions (Bash/Zsh mein add karo)

```bash
# ~/.bashrc ya ~/.zshrc

# Quick commit with message
function gc() { git commit -m "$1"; }

# Create and switch to new branch
function gnb() { git checkout -b "$1"; }

# Push current branch
function gpush() { git push origin "$(git branch --show-current)"; }

# Pull with rebase
function gpr() { git pull --rebase origin "$(git branch --show-current)"; }

# Delete branch locally and remotely
function gbdel() {
  git branch -d "$1"
  git push origin --delete "$1"
}
```

---

## 🎯 Key Takeaways — Part 1

| Concept | Remember |
|---------|----------|
| Git storage | Content-addressable, snapshots not diffs |
| 4 Object types | blob, tree, commit, tag |
| SHA-1 | 40-char hash, unique identifier for everything |
| HEAD | Pointer to current branch/commit |
| Index | Staging area binary file |
| Config priority | local > global > system |
| .gitignore | Already tracked files ko `git rm --cached` se ignore karo |

---

## ⚠️ Common Pitfalls — Part 1

```
1. git add . vs git add -A — scope difference bhool jaate hain
2. core.autocrlf — Windows/Mac mismatch cause karta hai CR/LF issues
3. .gitignore already tracked files pe kaam nahi karta
4. pull.rebase=true set na karna → ugly merge commits history mein
5. No user.email set → commits anonymous jaate hain
```

---

> ➡️ **Next:** Part 2 — Branching, Merging, Rebasing (The Heavy Stuff)
