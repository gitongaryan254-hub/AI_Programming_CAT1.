# AI_Programming_CAT1

**Project:** AI Programming CAT1  
**Author:** Ryan  
**Repository:** AI_Programming_CAT1

## Project overview
This repository contains the submission for **CAT1: AI Programming**.  
It demonstrates the tasks required for the assignment: creating a GitHub repository, initializing a local Git repository in Visual Studio Code, committing project files, and pushing them to GitHub.

## What this tests
- Basic Git workflow: `git init`, `git add`, `git commit`, `git branch`, `git remote`, `git push`.  
- Using Visual Studio Code integrated terminal for Git operations.  
- Creating and documenting a public GitHub repository.  
- Verifying successful push with a terminal screenshot.

## Tools and environment used
- **Visual Studio Code** — editor and integrated terminal.  
- **Git** — version control (Git for Windows / macOS / Linux).  
- **Git Bash** (recommended on Windows) or system shell (bash/zsh).  
- **GitHub** — remote repository hosting.  
- **Optional**: GitHub CLI (`gh`) or a Personal Access Token (PAT) for HTTPS authentication.

## Files included
- `README.md` — this file describing the project and steps taken.  
- (Other project files) — add your assignment code, scripts, or notebooks here.

## Steps performed (what I did)
1. **Installed Git** on my machine and verified with:
   ```bash
   git --version
   #!/usr/bin/env bash
set -euo pipefail

REPO_NAME="AI_Programming_CAT1"
REPO_DESC="CAT1 submission: tests AI programming tasks; includes README and project files."
README_TEXT="# AI_Programming_CAT1

**Project:** AI Programming CAT1  
**What it tests:** (briefly describe what your assignment tests; e.g., basic AI algorithms, Python scripting, Git usage)  
**Tools used:** Git, GitHub, Visual Studio Code, (add languages/tools used e.g., Python 3, NumPy)

## Notes
This repository was created and pushed using an automated script.
"

info(){ printf "\n[INFO] %s\n" "$1"; }
err(){ printf "\n[ERROR] %s\n" "$1"; exit 1; }

# 1. Basic checks
if ! command -v git >/dev/null 2>&1; then
  err "git is not installed. Install Git and re-run."
fi
info "git: $(git --version)"

# 2. Create README.md if missing (do not overwrite existing README)
if [ -f README.md ]; then
  info "README.md already exists; leaving it unchanged."
else
  printf "%s\n" "$README_TEXT" > README.md
  info "Created README.md"
fi

# 3. If gh (GitHub CLI) is available and authenticated, use it
if command -v gh >/dev/null 2>&1; then
  info "GitHub CLI (gh) found: $(gh --version | head -n1)"
  # Check auth
  if gh auth status >/dev/null 2>&1; then
    info "gh is authenticated. Creating repository via gh..."
    # If repo already exists under your account, gh will error; handle that
    set +e
    gh repo create "$REPO_NAME" --public --description "$REPO_DESC" --confirm
    GH_EXIT=$?
    set -e
    if [ $GH_EXIT -ne 0 ]; then
      info "gh create returned non-zero (repo may already exist). Continuing to set remote and push."
    else
      info "Repository $REPO_NAME created on GitHub."
    fi
    # Continue to local git steps below
  else
    info "gh is installed but not authenticated. Will fall back to API method."
  fi
fi

# 4. If gh not used to create repo, use REST API with PAT
if ! gh auth status >/dev/null 2>&1 2>/dev/null || [ -z "$(gh auth status 2>/dev/null || true)" ]; then
  # If repo already exists, API will return 422; script will handle that.
  read -r -p "Enter your GitHub username (for API repo creation) or press Enter to skip (if repo already exists): " GITHUB_USER
  if [ -z "${GITHUB_USER:-}" ]; then
    info "No username provided. Assuming repository already exists or you will add remote manually."
  else
    # Get PAT
    read -r -s -p "Enter a GitHub Personal Access Token (PAT) with repo scope (input hidden): " GITHUB_PAT
    printf "\n"
    if [ -z "${GITHUB_PAT:-}" ]; then
      err "No PAT provided. Cannot create repo via API. Either install & authenticate gh or provide PAT."
    fi
    info "Creating repository via GitHub API..."
    set +e
    CREATE_RESP=$(curl -s -o /dev/stderr -w "%{http_code}" -H "Authorization: token $GITHUB_PAT" \
      -d "{\"name\":\"$REPO_NAME\",\"description\":\"$REPO_DESC\",\"private\":false}" \
      https://api.github.com/user/repos)
    HTTP_CODE=$?
    set -e
    # curl exit code check: if curl failed, abort
    if [ $HTTP_CODE -ne 0 ]; then
      err "Failed to call GitHub API (curl error)."
    fi
    info "Repository creation attempted (if it already existed, that's OK)."
  fi
fi

# 5. Local git init / add / commit
if [ ! -d .git ]; then
  info "Initializing local git repository..."
  git init
else
  info "Local git repository already initialized."
fi

# Stage files
git add .

# Commit: if no commits exist, create initial commit; otherwise commit changes if any
if git rev-parse --verify HEAD >/dev/null 2>&1; then
  if git diff --quiet && git diff --cached --quiet; then
    info "No changes to commit."
  else
    git commit -m "Add project files and README for $REPO_NAME"
    info "Committed changes."
  fi
else
  info "Creating initial commit..."
  git commit -m "Initial commit for $REPO_NAME" --allow-empty || true
fi

# 6. Ensure branch main
git branch -M main || true

# 7. Add remote origin (use HTTPS). If origin exists, replace it.
REMOTE_URL="https://github.com/${GITHUB_USER:-gitongaryan254-hub}/$REPO_NAME.git"
# If user didn't provide username earlier, default to the repo URL the user gave earlier (their account)
# But to be safe, if GITHUB_USER empty, use the repo URL the user provided earlier in conversation:
if [ -z "${GITHUB_USER:-}" ]; then
  REMOTE_URL="https://github.com/gitongaryan254-hub/$REPO_NAME.git"
fi

if git remote | grep -q '^origin$'; then
  info "Removing existing origin remote..."
  git remote remove origin
fi
git remote add origin "$REMOTE_URL"
info "Added remote origin -> $REMOTE_URL"
git remote -v

# 8. Push to GitHub
info "Pushing to origin main..."
set +e
git push -u origin main
PUSH_EXIT=$?
set -e

if [ $PUSH_EXIT -eq 0 ]; then
  info "Push succeeded. Repository is ready at: $REMOTE_URL"
  info "Take a screenshot of this terminal showing the successful 'git push' output for submission."
  exit 0
fi

# 9. If push failed, give helpful next steps and attempt credential helper
warn_push() { printf "\n[WARN] %s\n" "$1"; }
warn_push "Push failed (exit code $PUSH_EXIT). Common causes: authentication or repo permissions."

# Try to configure credential helper and retry (best-effort)
unameOut="$(uname -s)"
case "${unameOut}" in
    MINGW*|MSYS*|CYGWIN*)
        git config --global credential.helper manager-core || true
        ;;
    Darwin*)
        git config --global credential.helper osxkeychain || true
        ;;
    *)
        git config --global credential.helper cache || true
        ;;
esac
info "Configured credential helper. Try pushing again; you may be prompted for credentials (use GitHub username and PAT for HTTPS)."
git push -u origin main || err "Push failed again. Check that the remote URL is correct and that you have push permission."




# AI_Programming_CAT1.
