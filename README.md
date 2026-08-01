# 🦝 sofi-scripts — Dev Environment Automation Scripts

A collection of battle-tested Bash scripts to automate Git identity management, SSH key configuration, repository operations, system maintenance, and local WordPress setup — designed for developers managing multiple GitHub accounts on a single machine.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/sofidev)

> [!IMPORTANT]
> Scripts are created **without** the `.sh` extension so they can be called directly by name from any terminal session. This follows the Unix convention for command-line tools.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [SSH Key Convention](#ssh-key-convention)
- [Quick Install](#quick-install)
- [Script Reference](#script-reference)
  - [set-scripts](#set-scripts)
  - [setup-git-users](#setup-git-users)
  - [add-git-user](#add-git-user)
  - [clone-repo](#clone-repo)
  - [clone-project](#clone-project)
  - [add-repo](#add-repo)
  - [push-repo](#push-repo)
  - [new-branch](#new-branch)
  - [gpull](#gpull)
  - [gpush](#gpush)
  - [clean-system](#clean-system)
  - [install-wordpress](#install-wordpress)
- [Use Case Workflows](#use-case-workflows)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
~/.local/bin/          ← Scripts directory (must be in $PATH)
~/.gitconfig           ← Main Git config (primary user + includeIf blocks)
~/.gitconfig-<name>    ← Per-user Git config (overrides inside specific dirs)
~/.ssh/config          ← SSH host aliases (gh-<keyname> → github.com)
~/.ssh/<keyname>       ← ed25519 private key
~/.ssh/<keyname>.pub   ← ed25519 public key (add to GitHub → Settings → SSH Keys)
```

### How multi-account Git identity works

Git's `[includeIf "gitdir:~/dir/"]` directive automatically activates a different `[user]` block when you are inside a matching directory. This means no manual `git config` commands are needed — identity switching is **automatic and directory-scoped**.

```
~/.gitconfig
├── [user] name = Personal Dev        ← Used everywhere by default
├── [includeIf "gitdir:~/work/"]
│     └── path = ~/.gitconfig-work   ← Overrides user when inside ~/work/
└── [init] defaultBranch = main
```

```
~/.ssh/config
├── Host gh-personal
│     HostName github.com             ← Resolves to github.com
│     IdentityFile ~/.ssh/personal    ← Uses personal key
└── Host gh-work
      HostName github.com
      IdentityFile ~/.ssh/work        ← Uses work key
```

Clone URLs use the alias instead of `github.com`:

```
git@gh-personal:SofiDevO/my-repo.git   ← Uses personal SSH key
git@gh-work:MyCompany/project.git      ← Uses work SSH key
```

---

## SSH Key Convention

All scripts follow the same naming pattern consistently:

| Component        | Pattern                          | Example                             |
| ---------------- | -------------------------------- | ----------------------------------- |
| Private key file | `~/.ssh/<keyname>`               | `~/.ssh/personal`                   |
| Public key file  | `~/.ssh/<keyname>.pub`           | `~/.ssh/personal.pub`               |
| SSH config host  | `Host gh-<keyname>`              | `Host gh-personal`                  |
| Clone URL format | `git@gh-<keyname>:user/repo.git` | `git@gh-personal:SofiDevO/repo.git` |

> The `gh-` prefix is a deliberate namespace to avoid collisions with system-level SSH hosts and make all GitHub aliases immediately recognizable.

---

## Quick Install

### 1. Create the directory and clone

```bash
mkdir -p ~/.local/bin
git clone https://github.com/SofiDevO/sofi-scripts ~/.local/bin
```

> [!TIP]
> If you don't need a particular script, delete it from `~/.local/bin/` before running `set-scripts`. It won't affect the others.

### 2. Bootstrap with `set-scripts`

```bash
chmod +x ~/.local/bin/set-scripts
~/.local/bin/set-scripts
```

This is the **only script you need to configure manually**. It handles all others.

### 3. Apply shell changes

```bash
source ~/.zshrc    # zsh users
# or
source ~/.bashrc   # bash users
```

### 4. Verify installation

```bash
which setup-git-users   # → /home/<user>/.local/bin/setup-git-users
which clone-repo        # → /home/<user>/.local/bin/clone-repo
which push-repo         # → /home/<user>/.local/bin/push-repo
```

---

## Script Reference

---

### `set-scripts`

**Configure all scripts as executable and register the directory in `$PATH`**

The bootstrapper for the entire toolkit. It iterates over every file in `~/.local/bin`, applies `chmod +x`, and ensures the directory is present in the `$PATH` of the detected shell.

#### What it does internally

1. Reads all files in `$HOME/.local/bin`
2. Skips itself, `.md` files, and subdirectories
3. Applies `chmod +x` to each file
4. Detects the current shell via `$SHELL`
5. If `~/.local/bin` is not in `$PATH`, appends `export PATH="~/.local/bin:$PATH"` to the appropriate config file

#### Shell detection matrix

| Shell  | Config file modified         |
| ------ | ---------------------------- |
| `bash` | `~/.bashrc`                  |
| `zsh`  | `~/.zshrc`                   |
| `fish` | `~/.config/fish/config.fish` |
| other  | `~/.profile`                 |

#### Usage

```bash
chmod +x ~/.local/bin/set-scripts
~/.local/bin/set-scripts
```

#### Example output

```
╔════════════════════════════════════════════════════════╗
║            Configure Scripts as Executable           ║
╚════════════════════════════════════════════════════════╝

📁 Looking for scripts in: /home/sofi/.local/bin
🔧 Processing: add-git-user
✅ add-git-user is now executable
🔧 Processing: clone-repo
✅ clone-repo is now executable
...

📊 Summary:
   • Scripts processed: 11
   • Directory: /home/sofi/.local/bin
✅ Directory /home/sofi/.local/bin is already in PATH

🎉 Setup complete!
📋 Available scripts:
   • add-git-user
   • add-repo
   • clone-project
   • clone-repo
   ...
```

---

### `setup-git-users`

**Full Git + SSH environment setup from scratch**

The primary onboarding script. Configures `~/.gitconfig`, `~/.ssh/config`, and generates one or more `ed25519` SSH keys in a single interactive session.

#### Inputs collected

| Prompt           | Stored as                         | Notes                         |
| ---------------- | --------------------------------- | ----------------------------- |
| Username         | `[user] name` in `~/.gitconfig`   | Display name in commits       |
| Email            | `[user] email` in `~/.gitconfig`  | Must be valid format          |
| SSH key name     | `~/.ssh/<name>`, `Host gh-<name>` | Alphanumeric + hyphens only   |
| SSH passphrase   | Encrypts the private key          | Confirmed twice               |
| Additional users | Repeatable                        | Each gets a dir-scoped config |

#### Generated files

```
~/.gitconfig
~/.ssh/config
~/.ssh/<main_key_name>          (chmod 600)
~/.ssh/<main_key_name>.pub      (chmod 644)
~/.ssh/<extra_key>              (one per additional user)
~/.ssh/<extra_key>.pub
~/.gitconfig-<extra_dir>        (one per additional user)
```

#### File permissions applied

| File               | Permission | Reason                                 |
| ------------------ | ---------- | -------------------------------------- |
| `~/.ssh/`          | `700`      | SSH daemon rejects world-readable dirs |
| `~/.ssh/config`    | `600`      | Sensitive host config                  |
| `~/.ssh/<key>`     | `600`      | Private key — never share this         |
| `~/.ssh/<key>.pub` | `644`      | Public key — safe to share             |

#### Usage

```bash
setup-git-users
```

#### Interactive session example (single user)

```
Enter the username for gitconfig: Sofia Dev
Enter the email for gitconfig: sofia@personal.com
Enter the SSH key name: personal

Do you want to add an additional user to gitconfig?
  1) Yes, add another user
  2) No, continue
Answer: 2

🔐 Set a password for the SSH key of Sofia Dev
Password: ••••••••
Confirm password: ••••••••

✅ Git and SSH configuration completed successfully.
📋 Configuration summary:
   • Main user: Sofia Dev (sofia@personal.com)
🔑 SSH keys:
   • ~/.ssh/personal (gh-personal) for Sofia Dev
```

#### Generated `~/.gitconfig`

```ini
[user]
    name = Sofia Dev
    email = sofia@personal.com

[init]
    defaultBranch = main
```

#### Generated `~/.ssh/config`

```
# Configuration for Sofia Dev
Host gh-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/personal
```

#### Interactive session example (two users — personal + work)

```
Enter the username for gitconfig: Sofia Dev
Enter the email for gitconfig: sofia@personal.com
Enter the SSH key name: personal

Do you want to add an additional user to gitconfig?
  1) Yes, add another user
  2) No, continue
Answer: 1

Enter the username: Sofia Corp
Enter the email: sofia@company.com
Enter the directory name for this user: work

Do you want to add an additional user to gitconfig?
  1) Yes, add another user
  2) No, continue
Answer: 2
```

#### Generated files for two-user setup

```
~/.gitconfig
├── [user] name = Sofia Dev
├── [user] email = sofia@personal.com
├── [includeIf "gitdir:~/work/"]
│     path = ~/.gitconfig-work
└── [init] defaultBranch = main

~/.gitconfig-work
└── [user] name = Sofia Corp
    [user] email = sofia@company.com

~/.ssh/config
├── Host gh-personal → ~/.ssh/personal
└── Host gh-work → ~/.ssh/work
```

> [!NOTE]
> After running this script, copy the content of `~/.ssh/<key>.pub` and add it to GitHub → **Settings → SSH and GPG keys → New SSH key**.

---

### `add-git-user`

**Add an additional SSH identity to an existing setup**

Use this after `setup-git-users` when you need to onboard a new GitHub account without reconfiguring everything from scratch.

#### Prerequisites

- `~/.gitconfig` must exist (created by `setup-git-users`)

#### What it does

1. Validates email format with regex `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
2. Validates key name characters: only `[a-zA-Z0-9_-]`
3. Checks for existing key conflicts and prompts to skip or rename
4. Inserts `[includeIf "gitdir:~/dir/"]` into `~/.gitconfig` (before `[init]` if present)
5. Creates `~/.gitconfig-<keyname>`
6. Appends a new `Host gh-<keyname>` block to `~/.ssh/config`
7. Generates an `ed25519` key with passphrase confirmation
8. Applies correct permissions
9. Prints the public key for immediate copy-paste into GitHub

#### Usage

```bash
add-git-user
```

#### Interactive session example

```
📝 Enter the username: FreelanceClient
📧 Enter the user email: sofia@client.dev
🔑 Enter the SSH key name (will be used as gh-<name>): client
📁 Enter the directory associated with this user (e.g. work): freelance

📄 Updating Git configuration...
✅ includeIf added to ~/.gitconfig
✅ Created ~/.gitconfig-client

🔐 Updating SSH configuration...
✅ Host gh-client added to ~/.ssh/config

🔐 Generating SSH key...
SSH key password: ••••••••
Confirm password: ••••••••

✅ Additional SSH user configured successfully.
📋 Summary:
   • Name:      FreelanceClient
   • Email:     sofia@client.dev
   • SSH host:  gh-client
   • Key:       ~/.ssh/client
   • Directory: ~/freelance/
   • gitconfig: ~/.gitconfig-client

📎 Remember to add the public key to GitHub:

ssh-ed25519 AAAAC3Nza... sofia@client.dev
```

#### Edge case: key already exists

If `~/.ssh/client` already exists:

```
⚠️  Key ~/.ssh/client already exists.
  1) Choose a different name
  2) Continue (existing key will not be overwritten)
Answer: 2
```

The script will skip key generation but still update `.gitconfig` and `~/.ssh/config`.

---

### `clone-repo`

**Clone a GitHub repository using a specific SSH identity**

Dynamically lists available SSH keys from `~/.ssh/` and rewrites the clone URL to use the correct host alias.

#### How URL rewriting works

```
Input:   git@github.com:SofiDevO/my-app.git
Output:  git@gh-personal:SofiDevO/my-app.git
```

This tells SSH to use the `gh-personal` host entry in `~/.ssh/config`, which maps to `github.com` with the `~/.ssh/personal` key. GitHub authenticates and serves the repo as usual.

#### Usage

```bash
# Navigate to the parent directory first
cd ~/projects
clone-repo
```

#### Interactive session

```
🔑 Select the SSH key to use:
  1) gh-personal
  2) gh-work
Answer: 1

Enter the repository URL (e.g. git@github.com:user/repo.git):
git@github.com:SofiDevO/my-app.git

🔑 Loading SSH key...
🔗 Cloning from: git@gh-personal:SofiDevO/my-app.git

Cloning into 'my-app'...
✅ Repository cloned successfully.
```

#### URL validation

The script enforces that the URL starts with `git@github.com:` using Bash regex. HTTPS URLs will be rejected:

```
❌ URL format must start with git@github.com:
```

---

### `clone-project`

**Clone a specific branch of a GitHub repository**

Identical to `clone-repo` but adds branch selection and uses `--single-branch` for a lighter, faster clone.

#### Key difference from `clone-repo`

| Feature             | `clone-repo`    | `clone-project`     |
| ------------------- | --------------- | ------------------- |
| Clones all branches | ✅              | ❌                  |
| Branch selection    | ❌              | ✅                  |
| Clone size          | Full            | Minimal             |
| Use case            | General cloning | Feature branch work |

#### Usage

```bash
cd ~/work
clone-project
```

#### Interactive session

```
🔑 Select the SSH key to use:
  1) gh-personal
  2) gh-work
Answer: 2

🦝 Enter the branch name you want to clone: feature/auth-module

Enter the repository URL (e.g. git@github.com:user/repo.git):
git@github.com:MyCompany/api.git

🔗 Cloning from: git@gh-work:MyCompany/api.git
🌱 Branch: feature/auth-module

Cloning into 'api'...
✅ Repository cloned successfully.
```

#### Equivalent manual command

```bash
git clone -b feature/auth-module --single-branch git@gh-work:MyCompany/api.git
```

---

### `add-repo`

**Initialize a local project and push to a completely empty GitHub repository**

> [!IMPORTANT]
> Use this when the GitHub repo was created with **no initial files** (no README, no LICENSE, no .gitignore). If the remote already has commits, use `push-repo` instead.

#### What it does

```
git init
git add .
git commit -m "<message>"
git branch -M main
git remote add origin git@gh-<key>:<user>/<repo>.git
git push -u origin main
```

#### Usage

```bash
cd ~/my-new-project
add-repo
```

#### Interactive session

```
🔑 Select the SSH key to use:
  1) gh-personal
  2) gh-work
Answer: 1

📝 Enter the first commit message: Initial commit — project scaffold

🔗 Enter the repository URL (e.g. git@github.com:user/repo.git):
git@github.com:SofiDevO/new-project.git

🔑 Loading SSH key...
📦 Initializing Git repository...
📁 Staging all files...
💾 Creating first commit...
🌿 Switching to main branch...
📤 Setting remote origin to: git@gh-personal:SofiDevO/new-project.git
🚀 Pushing to repository...

✅ Repository initialized and pushed successfully.
📋 Summary of actions performed:
   • git init
   • git add .
   • git commit -m "Initial commit — project scaffold"
   • git branch -M main
   • git remote add origin git@gh-personal:SofiDevO/new-project.git
   • git push -u origin main
```

---

### `push-repo`

**Initialize and push a local project to a remote repository that already has commits**

> [!IMPORTANT]
> Use this when the GitHub repo was created **with** a default README, LICENSE, or .gitignore. If the remote is empty, use `add-repo` instead.

#### Key difference from `add-repo`

`push-repo` includes an extra step:

```bash
git pull origin main --allow-unrelated-histories --no-rebase --no-edit
```

This merges the remote's initial commit (e.g., a GitHub auto-generated README) with the local history before pushing — preventing the "refusing to push non-fast-forward" error.

#### Usage

```bash
cd ~/existing-local-project
push-repo
```

#### Interactive session

```
🔑 Select the SSH key to use:
  1) gh-personal
  2) gh-work
Answer: 1

📝 Enter the first commit message: feat: add base project structure

🔗 Enter the repository URL:
git@github.com:SofiDevO/project-with-readme.git

🔑 Loading SSH key...
📦 Initializing Git repository...
📁 Staging all files...
💾 Creating first commit...
🌿 Switching to main branch...
📤 Setting remote origin...
🔄 Pulling remote changes (README/LICENSE)...
🚀 Pushing to repository...

✅ Code pushed successfully to the repository.
```

#### When to use which script

| Scenario                                    | Script      |
| ------------------------------------------- | ----------- |
| GitHub repo created empty (no files)        | `add-repo`  |
| GitHub repo created with README/LICENSE     | `push-repo` |
| Repo already initialized, just push changes | `gpush`     |

---

### `new-branch`

**Create and switch to a new Git branch**

A minimal wrapper around `git checkout -b` with input validation.

#### Usage

```bash
# Must be inside a Git repository
cd ~/my-project
new-branch
```

#### Interactive session

```
🦝  Enter the new branch name: feature/user-auth

✅ Branch 'feature/user-auth' created and checked out successfully.
```

#### Equivalent manual command

```bash
git checkout -b feature/user-auth
```

#### Validation

- Exits with error if the branch name is empty
- Inherits Git's own branch name validation (special characters, spaces, etc. will fail at the `git checkout` level)

---

### `gpull`

**Pull the current branch from origin**

A shorthand that auto-detects the current branch, eliminating the need to remember or type the branch name.

#### Usage

```bash
# Inside any Git repository
gpull
```

#### Output

```
Pulling branch 'feature/user-auth' from origin...
From github.com:SofiDevO/my-app
 * branch            feature/user-auth -> FETCH_HEAD
Already up to date.
```

#### Equivalent manual command

```bash
git pull origin $(git rev-parse --abbrev-ref HEAD)
```

#### Use cases

- Quick sync after a teammate pushes updates
- Daily pull before starting work
- CI/CD pre-scripts that need branch-agnostic pull

---

### `gpush`

**Push the current branch to origin**

Mirror of `gpull` — auto-detects the current branch and pushes.

#### Usage

```bash
gpush
```

#### Output

```
Pushing branch 'feature/user-auth' to origin...
Enumerating objects: 5, done.
...
To github.com:SofiDevO/my-app.git
   a1b2c3d..e4f5g6h  feature/user-auth -> feature/user-auth
```

#### Equivalent manual command

```bash
git push origin $(git rev-parse --abbrev-ref HEAD)
```

> [!NOTE]
> `gpush` does not stage or commit files. Use `git add` and `git commit` before running it.

---

### `clean-system`

**Clean apt cache, user cache, trash, and optionally kill background processes**

A maintenance script for Debian/Ubuntu-based systems. Runs non-destructive cleanup automatically, then enters an interactive process manager if requested.

#### What it cleans

| Action                         | Command                         | Requires sudo |
| ------------------------------ | ------------------------------- | :-----------: |
| APT package cache              | `apt-get clean`                 |      ✅       |
| Remove orphaned packages       | `apt-get autoremove -y`         |      ✅       |
| Remove old downloaded packages | `apt-get autoclean`             |      ✅       |
| Thumbnail cache                | `rm -rf ~/.cache/thumbnails/*`  |      ❌       |
| Trash                          | `rm -rf ~/.local/share/Trash/*` |      ❌       |

#### Usage

```bash
clean-system
```

#### Session example (with process management)

```
Starting system cleanup...
Cleaning package cache (apt)...
Cleaning user cache (~/.cache/thumbnails)...
Emptying trash...
File cleanup complete.
--------------------------------
Do you want to close background processes? (Y/n): Y

Closing background processes...
Top 15 active processes for current user:
  PID COMMAND          %CPU %MEM
 1234 chrome           12.3  8.1
 5678 node             4.2   3.0
 ...

Enter the PID of the process to close (or press Enter to finish): 1234
Process 1234 closed successfully.

Enter the PID of the process to close (or press Enter to finish):
Process management complete.
Maintenance complete!
```

#### Session example (skip processes)

```
Do you want to close background processes? (Y/n): n
Operation cancelled. No processes were closed.
Maintenance complete!
```

> [!CAUTION]
> `kill <PID>` sends `SIGTERM` by default. If a process doesn't stop, use `kill -9 <PID>` manually. The script does not force-kill.

---

### `install-wordpress`

**Fully automated local WordPress installation**

An end-to-end setup script that installs and configures a complete WordPress development environment: database creation, WordPress download, `wp-config.php` configuration, security keys, WP-CLI install, and a ready-to-run PHP development server.

#### Requirements

| Dependency       | Auto-installed? | Notes                                                                       |
| ---------------- | :-------------: | --------------------------------------------------------------------------- |
| PHP + extensions | ✅ (if missing) | `php-cli php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-zip` |
| MariaDB or MySQL |       ❌        | Must be installed and running                                               |
| `wget`, `curl`   |       ❌        | Standard on most systems                                                    |
| Internet access  |        —        | For WordPress download and WordPress.org salt API                           |

#### Installation flow (7 steps)

```
[1/6] Download WordPress (wordpress.org/latest.tar.gz → /tmp → $WP_DIR)
[2/6] Configure database (root password prompt, up to 3 retries)
[3/6] Configure wp-config.php (sed substitution + security salt from API)
[4/6] Download WP-CLI (/tmp/wp-cli.phar, reused if already present)
[5/6] Install WordPress (wp core install via WP-CLI)
[6/7] Install Adminer (lightweight DB web interface)
[7/7] Create start-server.sh (auto-detects available port ≥ 8080)
```

#### Usage

```bash
# Navigate to where you want the WordPress directory created
cd ~/projects
install-wordpress
```

#### Interactive session example

```
Select the database system:
  1) MariaDB
  2) MySQL
Option (1 or 2): 1

WordPress administrator user configuration:

WordPress username: sofidev
WordPress password: ••••••••
Confirm WordPress password: ••••••••
WordPress administrator email: sofia@dev.com
WordPress site name (title): My Dev Blog

Installation directory: /home/sofi/projects/my-dev-blog

[1/6] Downloading WordPress...
WordPress downloaded and extracted to /home/sofi/projects/my-dev-blog

[2/6] Configuring MariaDB database...
Enter MariaDB root password: ••••••••
Database created: my-dev-blog_db
Database user: my-dev-blog_user

[3/6] Configuring wp-config.php...
wp-config.php configured

[4/6] Installing WP-CLI...
WP-CLI installed

[5/6] Installing WordPress...
WordPress installed successfully

[6/7] Installing Adminer (database manager)...
Adminer installed successfully

[7/7] Creating server startup script...
Server script created

╔════════════════════════════════════════════════╗
║   WordPress installed successfully!            ║
╚════════════════════════════════════════════════╝

Access information:
  Directory: /home/sofi/projects/my-dev-blog
  WordPress: http://localhost:8080
  Admin user: sofidev
  Admin password: ••••••••

Database:
  Web manager: http://localhost:8080/adminer.php
  DB name: my-dev-blog_db
  DB user: my-dev-blog_user
  DB host: localhost

To start the server run:
  cd /home/sofi/projects/my-dev-blog && ./start-server.sh
```

#### Site slug generation (directory naming)

The site title is sanitized to create the directory name and database name:

| Input title            | Generated slug        | Directory                | DB name                  |
| ---------------------- | --------------------- | ------------------------ | ------------------------ |
| `My Dev Blog`          | `my-dev-blog`         | `./my-dev-blog/`         | `my-dev-blog_db`         |
| `Client Project 2025!` | `client-project-2025` | `./client-project-2025/` | `client-project-2025_db` |
| `WordPress`            | `wordpress`           | `./wordpress/`           | `wordpress_db`           |

#### Files generated inside `$WP_DIR`

```
my-dev-blog/
├── wp-config.php        ← Configured with your DB credentials + API salts
├── adminer.php          ← Database web UI (Adminer 4.8.1)
├── start-server.sh      ← chmod +x, auto-detects port, runs php -S
├── CREDENTIALS.txt      ← All access info saved locally
└── ...                  ← Full WordPress core files
```

#### Starting the server

```bash
cd ~/projects/my-dev-blog && ./start-server.sh
```

```
════════════════════════════════════════════════
  WordPress server started successfully
════════════════════════════════════════════════

  🌐 WordPress: http://localhost:8080
  🗄️  Database: http://localhost:8080/adminer.php
  📁 Directory: /home/sofi/projects/my-dev-blog

Press Ctrl+C to stop the server
```

> [!NOTE]
> `start-server.sh` uses `ss -tlnp` to scan for occupied ports starting at 8080. If 8080 is in use, it automatically increments (8081, 8082...) until a free port is found.

#### Error recovery

The script registers a `trap cleanup_on_error EXIT` handler. If any step fails, it automatically removes the partially-created `$WP_DIR` directory, leaving a clean state for re-running.

#### Database retry logic

If the root password is incorrect, the script retries up to **3 times** before exiting with a diagnostic message:

```
❌ Error: Incorrect password or connection problem
Please try again...

Attempt 2 of 3
Enter MariaDB root password: ••••••••
```

On exhausting retries:

```
╔════════════════════════════════════════════════╗
║   Error: Could not connect to MariaDB          ║
╚════════════════════════════════════════════════╝
Please verify that:
  1. MariaDB is running: sudo systemctl status mariadb
  2. The root password is correct
  3. The root user has adequate permissions
```

---

## Use Case Workflows

### 🔰 Workflow 1: First-time developer setup (single GitHub account)

```bash
# Step 1: Install the scripts
git clone https://github.com/SofiDevO/sofi-scripts ~/.local/bin
chmod +x ~/.local/bin/set-scripts && ~/.local/bin/set-scripts
source ~/.zshrc

# Step 2: Configure Git identity + SSH key
setup-git-users
# → Enter: name, email, key name (e.g. "personal"), passphrase

# Step 3: Add the public key to GitHub
# Copy output of:
cat ~/.ssh/personal.pub
# Paste into: GitHub → Settings → SSH and GPG keys → New SSH key

# Step 4: Test the connection
ssh -T git@gh-personal
# → Hi SofiDevO! You've successfully authenticated...

# Step 5: Clone your first repo
cd ~/projects
clone-repo
# → Select key: gh-personal
# → URL: git@github.com:SofiDevO/my-app.git
```

---

### 🏢 Workflow 2: Dual-account setup (personal + work)

```bash
# Configure both identities in one run
setup-git-users
# → Main user: Personal Dev | sofia@personal.com | key: personal
# → Additional user: Work Dev | sofia@company.com | dir: work

# Add public keys to both GitHub accounts
cat ~/.ssh/personal.pub   # → Add to personal GitHub account
cat ~/.ssh/work.pub       # → Add to work GitHub account

# Test both connections
ssh -T git@gh-personal    # → Hi PersonalAccount!
ssh -T git@gh-work        # → Hi WorkAccount!

# Clone a personal repo
cd ~/projects
clone-repo
# → Select: gh-personal

# Clone a work repo (identity auto-switches because dir is ~/work/)
mkdir -p ~/work
cd ~/work
clone-repo
# → Select: gh-work
```

---

### ➕ Workflow 3: Add a third account later (freelance client)

```bash
# You already have personal + work set up
add-git-user
# → Name: Client Corp
# → Email: sofia@clientcorp.com
# → Key name: client
# → Directory: freelance

cat ~/.ssh/client.pub  # → Add to client's GitHub org

mkdir -p ~/freelance
cd ~/freelance
clone-repo  # → Select: gh-client
```

---

### 🚀 Workflow 4: Start a new project from scratch

```bash
# Scenario A: Empty GitHub repo (no files)
mkdir ~/projects/new-api && cd ~/projects/new-api
# ... create your files ...
add-repo
# → Select key, enter commit message, enter repo URL

# Scenario B: GitHub repo created with a default README
mkdir ~/projects/backend && cd ~/projects/backend
# ... create your files ...
push-repo
# → Same flow, but pulls remote README before pushing
```

---

### 🔄 Workflow 5: Daily development loop

```bash
cd ~/projects/my-app

# Start of day — pull latest
gpull

# Create a feature branch
new-branch
# → feature/login-system

# ... write code ...

git add .
git commit -m "feat: add login form"

# Push the branch
gpush

# Switch to main for a hotfix
git checkout main
gpull
new-branch
# → hotfix/null-check
```

---

### 🌐 Workflow 6: Spin up a local WordPress site

```bash
cd ~/projects
install-wordpress
# → DB system: MariaDB
# → Admin: sofidev | Password: MyP@ss | Email: me@dev.com
# → Site name: Dev Portfolio

# Start the server
cd ~/projects/dev-portfolio && ./start-server.sh
# → Open: http://localhost:8080/wp-admin
# → DB manager: http://localhost:8080/adminer.php
```

---

### 🧹 Workflow 7: Weekly system maintenance

```bash
clean-system
# → Cleans apt cache, thumbnails, trash automatically
# → Prompt: close background processes? Y
# → Review top processes by CPU
# → Kill any memory hogs by PID
```

---

## Troubleshooting

### Scripts not found after install

```bash
# 1. Check executability
ls -la ~/.local/bin/ | grep -v "^d"

# 2. Check PATH
echo $PATH | tr ':' '\n' | grep local

# 3. Reload shell config
source ~/.zshrc    # or ~/.bashrc

# 4. Re-run setup
set-scripts
```

---

### SSH permission denied (publickey)

```bash
# 1. Verify the agent has the key loaded
ssh-add -l

# 2. Load the key manually
ssh-add ~/.ssh/personal

# 3. Test the connection
ssh -T git@gh-personal
# → Should say: Hi username! You've successfully authenticated...

# 4. Check the key is registered on GitHub
# GitHub → Settings → SSH and GPG keys
```

---

### Wrong Git identity on commits

```bash
# Check which config is active
git config user.email

# Verify includeIf is working
git config --show-origin user.email

# Make sure you're inside the correct directory
# e.g. work identity only applies inside ~/work/**
```

---

### WordPress database connection refused

```bash
# Check service status
sudo systemctl status mariadb
# or
sudo systemctl status mysql

# Start if stopped
sudo systemctl start mariadb

# Check root login
mariadb -u root -p
```

---

### WordPress port conflict

The `start-server.sh` script handles this automatically — but if you want to specify a port manually:

```bash
cd ~/projects/my-site
php -S localhost:9090
```

---

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/sofidev)
