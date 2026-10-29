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

<img width="910" height="1098" alt="image" src="https://github.com/user-attachments/assets/93064c4b-9865-45cc-9f87-363623868ba5" />

**Key terms:**
- **YAML:** a human-readable format for data, built on indentation.
- **Source of truth:** the single authoritative place where something is defined.
- **List:** an ordered set of items (`departments`).
- **Map:** a set of named entries (`users`, keyed by username).
- **`terraform console`:** an interactive prompt to test expressions.

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

Read the YAML file and convert it into Terraform data; confirms the file has no syntax errors.
```
   yamldecode(file("users.yaml"))
```

Show only the list of departments.
```
   yamldecode(file("users.yaml")).departments
```

Show one user's job title, to confirm individual values can be reached by username.
```
   yamldecode(file("users.yaml")).users["sam.chen"].job_title
```

Leave the console.
```
   exit
```

Go back to the top of the repository.
```bash
   cd ..
```

See what changed; Git shows the new `users.yaml`.
```bash
   git status
```

Stage the file for the next save point.
```bash
    git add .
```

Create the save point with a message.
```bash
    git commit -m "Step 3: users.yaml source of truth"
```

Upload the commit to GitHub.
```bash
    git push
```
<img width="1572" height="702" alt="EC95E62C-0F15-4E27-A1A6-345532A7C499" src="https://github.com/user-attachments/assets/87e763c7-74af-4109-a6ed-6701feda842a" />

### Step 4: Entra users

**Goal:** Turn each person in `users.yaml` into an Entra ID user account, without listing anyone in the code.

**What I did:**
1. Wrote `entra/main.tf`: a data source, locals, random passwords, and users.
2. Formatted and validated the code.
3. Ran `terraform plan` to preview the result: 10 resources (5 passwords, 5 users).

**Why it matters:**
The code has **no names in it**. It says "for each person in the YAML file, make a user." Adding a sixth employee means adding 4 lines to `users.yaml`; the code doesn't change. This is the "joiner" process automated.

**The code, block by block:**

`data "azuread_domains" "tenant"`
- A **data source** *reads* something that already exists; it never creates anything.
- It asks Entra for the tenant's built-in domain (`arbofelipe2003outlook.onmicrosoft.com`), so the domain isn't typed by hand.

`locals { }`
- **Local values**: named shortcuts used throughout the file.
- `domain`: the domain found by the data source.
- `config`: the whole YAML file, read with `file()` and converted with `yamldecode()`.
- `users`: just the `users` part of it.
- `path.module` = "the folder this file is in," so the path works from anywhere.

`resource "random_password" "user"`
- Generates a 20-character password with special characters, one per user.
- Passwords are never written by a person and never appear in the code.

`resource "azuread_user" "user"`
- Creates the actual Entra accounts.

| Line | Meaning |
|---|---|
| `for_each = local.users` | Repeat this block once per user in the YAML |
| `each.key` | The username, like `alex.rivera` |
| `each.value` | That user's details (display name, department, title) |
| `"${each.key}@${local.domain}"` | Builds the sign-in name: `alex.rivera@...onmicrosoft.com`. `${ }` inserts a value into text |
| `password = random_password.user[each.key].result` | A **reference**: use the password generated for this same user |
| `force_password_change = true` | The user must set their own password at first sign-in |
| `account_enabled = try(each.value.enabled, true)` | Use `enabled` if the user has it; otherwise default to `true`. Only Taylor is disabled |

<img width="894" height="743" alt="Screenshot 2026-10-28 at 17 57 17" src="https://github.com/user-attachments/assets/f2ca2ec7-93a2-4e49-b02a-6a8a35111660" />

**Resource names:** `resource "azuread_user" "user"` has two labels:
- `azuread_user` = the **type** (what kind of thing; defined by the provider).
- `user` = my **name** for it inside the code, used in references. With `for_each`, each copy is addressed by key: `azuread_user.user["sam.chen"]`.

**Reading the plan:**
- `+` = will be created.
- `(sensitive value)` = Terraform hides passwords, even in the preview.
- `(known after apply)` = a value Entra assigns when the user is created, like the object ID.

**Key terms:**
- **Resource:** something Terraform creates and manages.
- **Data source:** something Terraform only reads.
- **Local value:** a named shortcut inside the configuration.
- **`for_each`:** creates one copy of a block per item in a map or set.
- **Reference:** using one resource's value in another (`random_password.user[each.key].result`).
- **`terraform fmt`:** fixes spacing and alignment automatically.
- **`terraform validate`:** checks the code for errors without contacting Entra.
- **`terraform plan`:** a preview of what would change; nothing is created.


**Commands used (in order):**

1. *(In VS Code: created `entra/main.tf`, then saved.)*

2. Go into the Entra folder.
```bash
   cd entra
```

3. Fix spacing and alignment automatically, so the code follows Terraform's standard style.
```bash
   terraform fmt
```

4. Check the code for errors without contacting Entra.
```bash
   terraform validate
```

<img width="488" height="101" alt="Screenshot 2026-10-28 at 18 00 02" src="https://github.com/user-attachments/assets/0b31440c-77a6-44b7-9799-877c3724ea7e" />

5. Preview what would be created; nothing changes yet. Expected: 10 to add.
```bash
   terraform plan
```

<img width="846" height="598" alt="Screenshot 2026-10-28 at 18 01 35" src="https://github.com/user-attachments/assets/8248556e-da89-4203-89bf-9190c8284e96" />

6. Go back to the top of the repository.
```bash
   cd ..
```

7. See what changed; Git shows the new `main.tf`.
```bash
   git status
```

8. Stage the file for the next save point.
```bash
   git add .
```

9. Create the save point with a message.
```bash
   git commit -m "Step 4: Entra users"
```

10. Upload the commit to GitHub.
```bash
    git push
```



### Step 5: Entra groups and the dynamic group

**Goal:** Organize users into groups automatically: one static group per department, plus one dynamic group that Entra keeps up to date by itself.

**What I did:**
1. Added three resources to `entra/main.tf`: department groups, group memberships, and a dynamic group.
2. Formatted, validated and previewed the code: 20 resources in total.

**Why it matters:**
In identity, **access is given to groups, not to individual people**. An app is assigned to `SG-IT`, and everyone in `SG-IT` gets it. So putting people in the right groups automatically is the core of access management:
- **Joiner:** a new employee lands in their department group on day one.
- **Mover:** change `department` in the YAML, and they move to the new group.
- **Leaver:** set `enabled: false`, and the dynamic group drops them on its own.

**The code, block by block:**

`resource "azuread_group" "department"`
- Creates one **security group** per department: `SG-Finance`, `SG-HR`, `SG-IT`, `SG-Security`.
- `toset(local.config.departments)`: `for_each` only accepts a **map** or a **set**, and `departments` is a **list**, so `toset()` converts it. With a set, `each.key` is the department name.
- `security_enabled = true`: a group used to **grant access** (as opposed to an email distribution group).

`resource "azuread_group_member" "department"`
- Links each user to their department's group, one membership per user.

| Line | Meaning |
|---|---|
| `azuread_group.department[each.value.department]` | Find the group matching this user's department (for Alex: `SG-Finance`) |
| `.object_id` | Entra's unique ID for that group |
| `azuread_user.user[each.key].object_id` | This user's unique ID |

- These references also tell Terraform the **order**: create the users and groups first, then the memberships.

`resource "azuread_group" "synced_users"`
- A **dynamic group**: I don't add members. Entra adds and removes them by itself, based on a **rule**.
- `types = ["DynamicMembership"]`: makes it dynamic.
- The rule: `(user.companyName -eq "NovaLab") and (user.accountEnabled -eq true)` = every user whose company is NovaLab **and** whose account is enabled.
- Taylor Morgan is disabled (`enabled: false`), so Taylor is **excluded automatically**.
- `\"` = a quote mark **inside** text. Without the backslash, the quote would end the text early.

<img width="840" height="676" alt="Screenshot 2026-10-29 at 16 57 55" src="https://github.com/user-attachments/assets/3c837acb-0dbc-42de-acf2-783ccdaad915" />

**Static vs dynamic groups:**

| | Static (department groups) | Dynamic (Synced Users) |
|---|---|---|
| Who adds members | Terraform (or an admin) | Entra, by a rule |
| Updates when a user changes | When Terraform runs | Automatically, within minutes |
| Licensing | Free | Requires Entra ID P1 |

**Key terms:**
- **Security group:** a group used to grant access to apps and resources.
- **Dynamic group:** a group whose members are calculated from user attributes.
- **Object ID:** Entra's unique identifier for a user or group.
- **`toset()`:** converts a list into a set, so `for_each` can use it.
- **Joiner / Mover / Leaver:** the three identity lifecycle events: new hire, role change, departure.

**Commands used (in order):**

1. *(In VS Code: added the three group resources to the bottom of `entra/main.tf`, then saved.)*

2. Go into the Entra folder.
```bash
   cd entra
```

3. Fix spacing and alignment automatically.
```bash
   terraform fmt
```

4. Check the code for errors without contacting Entra.
```bash
   terraform validate
```
<img width="477" height="86" alt="Screenshot 2026-10-29 at 16 59 41" src="https://github.com/user-attachments/assets/29c76de9-3e34-4dcb-b1fe-557ff7df7747" />

5. Preview what would be created; nothing changes yet. Expected: 20 to add.
```bash
   terraform plan
```

<img width="1126" height="790" alt="Screenshot 2026-10-29 at 17 01 51" src="https://github.com/user-attachments/assets/65858ba6-fc18-4804-b4c4-5de600b2e171" />

<img width="471" height="282" alt="Screenshot 2026-10-29 at 17 02 40" src="https://github.com/user-attachments/assets/bc9935aa-7cd2-41a9-bf9f-7396593f53f2" />

6. Go back to the top of the repository.
```bash
   cd ..
```

7. See what changed; Git shows `main.tf` as modified.
```bash
   git status
```

8. Stage the change for the next save point.
```bash
   git add .
```

9. Create the save point with a message.
```bash
   git commit -m "Step 5: Entra groups and dynamic group"
```

10. Upload the commit to GitHub.
```bash
    git push
```

<img width="620" height="336" alt="Screenshot 2026-10-29 at 17 06 01" src="https://github.com/user-attachments/assets/21435b11-dee2-4dd0-90f2-973cc61b41f3" />


### Step 6: Outputs, apply, and state

**Goal:** Create everything in Entra ID, confirm the result, and understand how Terraform remembers what it built.

**What I did:**
1. Added `outputs.tf` to display useful results after apply.
2. Ran `terraform apply`: 20 resources created.
3. Verified the users and groups in the Entra admin center.
4. Ran `terraform plan` again: no changes.
5. Explored Terraform's state with `state list`, `state show` and `output`.

**Why it matters:**
One command created 5 users, 4 groups, 5 memberships and a dynamic group, the same way every time. Clicking this through the portal takes about 30 minutes and invites mistakes.

**Outputs:**
An output prints a value after apply, for example each user's sign-in name.

`{ for key, user in azuread_user.user : key => user.user_principal_name }` is a **for expression**: "for each user, make a pair of username → sign-in name." The result:

```
user_upns = {
  "alex.rivera" = "alex.rivera@arbofelipe2003outlook.onmicrosoft.com"
  ...
}
```

**The Terraform loop:**

| Command | What it does | Changes anything? |
|---|---|---|
| `terraform plan` | Compares the code, the state, and the real world; shows the difference | No |
| `terraform apply` | Shows the plan, asks for `yes`, then makes the changes | Yes |

**Plan, then read, then apply.** The plan is the moment to catch mistakes, before they reach real accounts.

**Why the second plan said "No changes":**
Terraform is **declarative**: the code describes the **end result** ("these 5 users should exist"), not the steps. Running it again doesn't create duplicates; it sees everything already matches. This is called **idempotent**.

**State: Terraform's memory**
- After apply, Terraform writes `terraform.tfstate`, recording every resource it created and its real ID in Entra.
- That's how it knows `azuread_user.user["sam.chen"]` is a specific user, and what to update or destroy later.
- The state contains the generated **passwords**, which is why `.gitignore` blocks it (Step 1). `git status` confirmed it never reached GitHub.

**The dynamic group in action:**
NovaLab Synced Users has 4 members; Taylor Morgan, whose account is disabled, was excluded by Entra's rule, with no manual work.

**Key terms:**
- **Output:** a value displayed after apply.
- **For expression:** builds a new map or list from an existing one.
- **Declarative:** describe the end result, not the steps.
- **Idempotent:** running it again gives the same result, with no duplicates.
- **State:** Terraform's record of what it manages and the real IDs.
- **`terraform state list` / `state show`:** list everything in state / show one item in detail.
- 

**Commands used (in order):**

*(In VS Code: created `entra/outputs.tf`, then saved.)*

<img width="1111" height="383" alt="Screenshot 2026-10-29 at 17 54 43" src="https://github.com/user-attachments/assets/ae77bf43-48d4-4f8d-93a0-6a088861145f" />

Go into the Entra folder.
```bash
   cd entra
```

Fix spacing and alignment automatically.
```bash
   terraform fmt
```

Check the code for errors.
```bash
   terraform validate
```

<img width="488" height="96" alt="Screenshot 2026-10-29 at 17 57 00" src="https://github.com/user-attachments/assets/6c098c2a-7512-461c-be21-887d7d32a7e8" />

Show the plan, then create everything after I typed `yes` (20 resources).
```bash
   terraform apply
```

<img width="1084" height="777" alt="Screenshot 2026-10-29 at 17 58 44" src="https://github.com/user-attachments/assets/df09c891-ac37-4f6f-bc9c-277b7c3a9038" />

<img width="994" height="801" alt="Screenshot 2026-10-29 at 18 00 54" src="https://github.com/user-attachments/assets/5ff6c51c-f23c-4523-afed-17505e4d8308" />

<img width="1099" height="642" alt="Screenshot 2026-10-29 at 18 02 26" src="https://github.com/user-attachments/assets/ef53b019-5859-43a8-9f9f-17603297f02c" />

*(In the Entra admin center: checked the users, SG-IT members, and the dynamic group.)*

Run the plan again, to confirm the real world matches the code: no changes.
```bash
   terraform plan
```

<img width="1117" height="358" alt="Screenshot 2026-10-29 at 18 03 55" src="https://github.com/user-attachments/assets/89270981-ddee-4e0a-b33f-ff73f388777d" />

List everything Terraform is tracking in its state.
```bash
   terraform state list
```

<img width="822" height="795" alt="Screenshot 2026-10-29 at 18 05 28" src="https://github.com/user-attachments/assets/cf145fa8-0ee7-4eac-9754-eb1d4cec35d2" />

Show everything Terraform knows about one user.
```bash
   terraform state show 'azuread_user.user["sam.chen"]'
```

<img width="961" height="752" alt="Screenshot 2026-10-29 at 18 06 36" src="https://github.com/user-attachments/assets/5486c562-947a-47d0-a1bd-c12c4055044b" />

Print the outputs again, without re-applying.
```bash
    terraform output
```
<img width="529" height="211" alt="Screenshot 2026-10-29 at 18 07 33" src="https://github.com/user-attachments/assets/16de443e-9670-4ca9-af0f-554fd2df73c7" />


Go back to the top of the repository.
```bash
    cd ..
```

See what changed; confirm `terraform.tfstate` is not listed.
```bash
    git status
```

Stage the changes for the next save point.
```bash
    git add .
```

Create the save point with a message.
```bash
    git commit -m "Step 6: Outputs, apply, and state"
```

Upload the commit to GitHub.
```bash
    git push
```

<img width="770" height="419" alt="Screenshot 2026-10-29 at 18 09 16" src="https://github.com/user-attachments/assets/92ac7f9b-b905-44d6-8112-2debda90113b" />
