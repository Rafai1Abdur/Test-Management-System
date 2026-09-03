# Update Git Remote and Push

## Context
- Local branch: `main`
- Local repo has 1 commit: `02aac07 docs: establish architecture and assessment model baseline`
- Working tree is clean
- Current remote: `https://github.com/Rafai1Abdur/Test-Management-System.git`
- Target remote: `https://github.com/Coderatory-Codebase/AI-Test-Maker.git`

## Assumption
`git ls-remote https://github.com/Coderatory-Codebase/AI-Test-Maker.git` returned no output, indicating the target repository is likely empty. If it is not empty, a regular push will fail.

## Steps
1. Change the remote URL:
   ```bash
   git remote set-url origin https://github.com/Coderatory-Codebase/AI-Test-Maker.git
   ```
2. Verify the remote was updated:
   ```bash
   git remote -v
   ```
3. Push the current `main` branch to the new remote:
   ```bash
   git push -u origin main
   ```
4. If step 3 fails because the remote already has commits, confirm with the user before running:
   ```bash
   git push --force origin main
   ```

## Validation
- Confirm `origin` points to `https://github.com/Coderatory-Codebase/AI-Test-Maker.git`
- Confirm `git log --oneline` matches the local commit history after push
- Confirm the branch is up to date with `origin/main`

## Risk
- Force push may overwrite existing remote history if the target repo is not empty. Only execute after user confirmation.
