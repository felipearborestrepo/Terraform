<img width="857" height="483" alt="Screenshot 2026-10-27 at 15 30 49" src="https://github.com/user-attachments/assets/f7620c4e-6482-4db9-8da5-38728b4faecb" />
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

<img width="790" height="388" alt="Screenshot 2026-10-27 at 15 31 16" src="https://github.com/user-attachments/assets/abd03539-2ab3-47d1-be00-1888633d1b64" />

**Key terms:**
- **Repository:** a folder Git tracks, with the full history of every file.
- **Commit:** a save point with a message describing the change.
- **Push:** upload commits to GitHub.
- **Tag:** a bookmark on one commit that never moves. I tagged the first version as `v1-first-build` before rebuilding.
- **`.gitignore`:** a list of files Git must never track.
- **Hidden file:** a file whose name starts with a dot (`.gitignore`, `.git`); `ls -a` shows them.

### Step 2: Terraform providers

**Goal:** Tell Terraform which systems it will manage and download the plugins it needs to talk to them.

**What I did:**
1. Confirmed Terraform and the Azure CLI were installed.
2. Logged in to Azure with `az login`.
3. Created an `entra` folder for the Entra ID configuration.
4. Wrote `versions.tf` to declare the providers.
5. Ran `terraform init` to download them.
6. Committed `versions.tf` and the lock file; `.gitignore` kept the downloaded plugins out of Git.

**Why it matters:**
Terraform doesn't know how to talk to Entra ID, Okta or any other system by itself. It uses **providers**: plugins that translate Terraform code into API calls for one system. Declaring them first means anyone who copies the repo gets the same plugins and versions.

**The code, line by line:**

| Code | Meaning |
|---|---|
| `terraform { }` | Settings about Terraform itself |
| `required_providers { }` | The list of plugins this project needs |
| `azuread = { ... }` | The plugin for Microsoft Entra ID (formerly Azure AD) |
| `source = "hashicorp/azuread"` | Where to download it: the Terraform Registry, publisher `hashicorp` |
| `version = "~> 3.0"` | Any 3.x version (3.1, 3.5...) but not 4.0, which could break the code |
| `random = { ... }` | A plugin that generates random values; used later for user passwords |
| `provider "azuread" {}` | Turns the Entra plugin on. Empty braces mean "use my Azure CLI login" |

<img width="779" height="406" alt="Screenshot 2026-10-27 at 15 44 57" src="https://github.com/user-attachments/assets/784ffd92-53b6-4ada-9fb4-eebb929ab16c" />

**What `terraform init` does:**
- Reads `required_providers` and downloads each plugin into a hidden `.terraform/` folder.
- Writes `.terraform.lock.hcl`, which records the **exact** versions downloaded.

<img width="626" height="366" alt="Screenshot 2026-10-27 at 15 49 18" src="https://github.com/user-attachments/assets/529458bc-f14d-44fa-b794-cbe0dba8715d" />

| File | Committed to Git? | Why |
|---|---|---|
| `versions.tf` | Yes | The code |
| `.terraform.lock.hcl` | Yes | Everyone (and the pipeline later) uses the exact same plugin versions |
| `.terraform/` | No | Large downloaded files, recreated by `terraform init` |


**How Terraform signs in to Entra:**
The empty `provider "azuread" {}` block uses the account from `az login`. No password or secret is written in the code.

**Key terms:**
- **Provider:** a plugin that lets Terraform manage one system (Entra, Okta, AWS...).
- **Terraform Registry:** the public catalog where providers are downloaded from.
- **Version constraint:** a rule for which versions are allowed (`~> 3.0` = 3.x only).
- **Lock file:** a record of the exact plugin versions used.
- **`terraform init`:** the first command in any Terraform folder; prepares it to run.

List all files, including hidden ones, to see what `init` created (`.terraform/` and `.terraform.lock.hcl`).
```bash
   ls -a
```
Go back to the top of the repository, where Git commands run.
```bash
   cd ..
```

See what changed since the last commit; Git shows `entra/` as new.
```bash
    git status
```

List every file inside new folders, to confirm `.terraform/` is ignored and only `versions.tf` and the lock file will be committed.
```bash
    git status --untracked-files=all
```
<img width="645" height="366" alt="Screenshot 2026-10-27 at 15 57 14" src="https://github.com/user-attachments/assets/6fa8dd49-c076-4522-b0c3-cd89be23ff21" />

Stage the new files for the next save point.
```bash
    git add .
```

Create the save point with a message.
```bash
    git commit -m "Step 2: Terraform providers"
```

Upload the commit to GitHub.
```bash
    git push
```

<img width="559" height="244" alt="Screenshot 2026-10-27 at 17 53 51" src="https://github.com/user-attachments/assets/5493a7ca-0774-4f25-8afe-82ddfa748077" />

### Step 3: users.yaml, the source of truth

**Goal:** Define NovaLab's employees and departments once, in one file, that both Entra ID and Okta will read.

**What I did:**
1. Created `entra/users.yaml` with the company name, 4 departments and 5 employees.
2. Tested it in `terraform console` to confirm Terraform reads it correctly before writing any resources.
3. Committed it to GitHub.

**Why it matters:**
- This file is the **source of truth**: the one place where people are defined. To hire someone, I add a block; to change a title, I edit a line; to disable someone, I set `enabled: false`.
- **Data is separate from logic.** The `.tf` files describe *how* to create users; this file describes *who*. Someone in HR could update this file without knowing Terraform.
- Both Entra ID and Okta read the **same** file, so the two systems can never disagree about who works here.

**How the file is structured:**

| Part | Type | Meaning |
|---|---|---|
| `company: NovaLab` | single value | The company name, stamped on every user |
| `departments:` with `- Finance` ... | list | Each `-` is one item; used to create one group per department |
| `users:` | map | A set of named entries; each key (`alex.rivera`) is a username |
| `display_name`, `department`, `job_title` | values | Details about each user |
| `enabled: false` | optional value | Only on Taylor; a disabled account (a "leaver" scenario). Everyone else defaults to enabled |

<img width="835" height="748" alt="Screenshot 2026-10-28 at 17 03 03" src="https://github.com/user-attachments/assets/dcacaae3-3e4f-4838-bc3d-37aa15c08576" />

**YAML rules:**
- **Indentation is the structure.** 2 spaces = "belongs to the line above." One wrong space changes the meaning or breaks the file.
- **Spaces only, never Tab characters.**
- `key: value` (a space after the colon).
- `- item` makes a list item.

**Why I tested in `terraform console`:**
`terraform console` is an interactive prompt to try Terraform functions without creating anything. `yamldecode(file("users.yaml"))` is the exact expression the Terraform code will use in the next step, so if it works here, the data is ready.

**Key terms:**
- **YAML:** a human-readable format for data, built on indentation.
- **Source of truth:** the single authoritative place where something is defined.
- **List:** an ordered set of items (`departments`).
- **Map:** a set of named entries (`users`, keyed by username).
- **`terraform console`:** an interactive prompt to test expressions.

**What I learned:**
- _(2 or 3 points in your own words)_

**Commands used (in order):**

1. *(In VS Code: created `entra/users.yaml`, then saved.)*

2. Go into the Entra folder, where the file is.
```bash
   cd entra
```

3. Open Terraform's interactive prompt to test expressions without creating anything.
```bash
   terraform console
```

4. Read the YAML file and convert it into Terraform data; confirms the file has no syntax errors.
```
   yamldecode(file("users.yaml"))
```

5. Show only the list of departments.
```
   yamldecode(file("users.yaml")).departments
```

6. Show one user's job title, to confirm individual values can be reached by username.
```
   yamldecode(file("users.yaml")).users["sam.chen"].job_title
```

7. Leave the console.
```
   exit
```

8. Go back to the top of the repository.
```bash
   cd ..
```

9. See what changed; Git shows the new `users.yaml`.
```bash
   git status
```

10. Stage the file for the next save point.
```bash
    git add .
```

11. Create the save point with a message.
```bash
    git commit -m "Step 3: users.yaml source of truth"
```

12. Upload the commit to GitHub.
```bash
    git push
```

