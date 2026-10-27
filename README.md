# Identity Platform as Code: Entra ID + Okta with Terraform

Automated identity lifecycle for a fictional company, **NovaLab**. Users and groups are defined once in a YAML file, then created in **Microsoft Entra ID** and **Okta** with **Terraform**. Every change is checked by **GitHub Actions** and applied through **HCP Terraform** with human approval.

## Tools

| Tool | Role |
|---|---|
| Terraform | Defines identity resources as code |
| Microsoft Entra ID | Cloud directory (users, groups) |
| Okta | Identity provider (groups, policies, apps) |
| Git + GitHub | Version control and pull requests |
| GitHub Actions | Automatic code checks |
| HCP Terraform | Remote state and approved applies |

## Steps

### Step 1: Repository setup
**Goal:** Create the repository with protection for secrets and a project README.

**What I did:**
1. Created a `.gitignore` file before writing any code.
2. Confirmed with `git status` that ignored files (like `.DS_Store`) no longer appear.
3. Created this README to explain the project.
4. Committed and pushed both files to GitHub.

**Why it matters:**
- Terraform creates a **state file** (`terraform.tfstate`): its memory of everything it built. The state file contains **passwords** in readable form.
- **Git never forgets.** A file committed once stays in the history even after it's deleted, and this repository is public.
- So the protection has to exist **before** any secret file does.

**What `.gitignore` blocks:**

| Pattern | What it blocks | Why |
|---|---|---|
| `*.tfstate`, `*.tfstate.*` | State files and backups | Contain passwords and resource IDs |
| `**/.terraform/*` | Downloaded provider plugins (any folder) | Large; re-downloaded by `terraform init` |
| `*.tfvars` | Variable files | Often hold tokens |
| `.env` | Local secret files | Tokens and passwords |
| `.DS_Store` | macOS Finder settings | Not part of the project |

**Key terms:**
- **Repository:** a folder Git tracks, with the full history of every file.
- **Commit:** a save point with a message describing the change.
- **Push:** upload commits to GitHub.
- **Tag:** a bookmark on one commit that never moves. I tagged the first version as `v1-first-build` before rebuilding.
- **`.gitignore`:** a list of files Git must never track.
- **Hidden file:** a file whose name starts with a dot (`.gitignore`, `.git`); `ls -a` shows them.
