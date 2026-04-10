# Dev Environment Automation Scripts

A collection of bash scripts to automate common Git, SSH, and system tasks.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/sofidev)

> [!IMPORTANT]
> Scripts are created **without** the `.sh` extension so they can be called directly by name.

---

## Quick Install

### 1. Clone the repository into the correct path:

```bash
# Create the directory if it doesn't exist
mkdir -p ~/.local/bin

# Clone directly into the path
git clone <REPOSITORY_URL> ~/.local/bin
```

> [!TIP]
> If you don't need a particular script, just delete it from `~/.local/bin/` before running `set-scripts`.

### 2. Set up scripts automatically:

```bash
# Make the setup script executable
chmod +x ~/.local/bin/set-scripts

# Run the automatic setup
~/.local/bin/set-scripts
```

> [!NOTE]
> `set-scripts` is the **only** script you need to configure manually. It handles making all other scripts executable and adding the directory to your PATH.

### 3. Apply the changes:

```bash
source ~/.zshrc # or ~/.bashrc depending on your shell
```

### 4. Verify the install:

```bash
which setup-git-users
which clone-repo
which push-repo
which new-branch
```

---

## Available Scripts

### `set-scripts`
**Set all scripts as executable and configure PATH**
- Makes every script in `~/.local/bin` executable automatically
- Adds the directory to PATH in your shell config (bash, zsh, fish)
- Detects your shell automatically
- This is the only script you need to configure manually

---

### `setup-git-users`
**Full Git + SSH environment setup**
- Prompts for your username, email, and SSH key name (decoupled — your display name and key filename can differ)
- Generates `~/.gitconfig` with `[user]` block and optional `[includeIf]` entries for per-directory user switching
- Creates `~/.ssh/config` with a `Host gh-<keyname>` alias pointing to `github.com`
- Generates an `ed25519` SSH key with a passphrase
- Supports adding multiple additional users in a single run
- Sets correct file permissions on all generated files

```
Enter the username for gitconfig: → stored in [user] name
Enter the email for gitconfig: → stored in [user] email
Enter the SSH key name: → e.g. sofidev → key at ~/.ssh/sofidev, host gh-sofidev
```

---

### `add-git-user`
**Add an additional SSH user to an existing setup**
- Requires `setup-git-users` to have been run first
- Appends a new `[includeIf]` block to `~/.gitconfig`
- Creates a new `~/.gitconfig-<keyname>` file
- Adds a new `Host gh-<keyname>` block to `~/.ssh/config`
- Generates a new `ed25519` SSH key
- Validates email format, key name characters, and existing file conflicts

---

### `clone-repo`
**Clone a GitHub repository using SSH**
- Dynamically detects all SSH keys in `~/.ssh/` and presents a numbered menu
- Rewrites the clone URL to use the selected `gh-<keyname>` host alias
- Loads the SSH key via `ssh-add` before cloning

---

### `clone-project`
**Clone a specific branch of a GitHub repository**
- Same dynamic SSH key selection as `clone-repo`
- Prompts for the branch name
- Uses `--single-branch` for a lighter clone

---

### `push-repo`
**Push a local project to a remote repository that already has commits (e.g. created with a README or LICENSE)**
- Runs `git init`, stages all files, and makes the first commit
- Dynamic SSH key selection
- Pulls remote history with `--allow-unrelated-histories` to avoid rejection
- Sets `origin` remote, renames branch to `main`, and pushes

> Use this when you created the repo on GitHub with a default README or LICENSE.

---

### `add-repo`
**Initialize a local project and push it to a completely empty remote repository**
- Runs `git init`, stages all files, and makes the first commit
- Dynamic SSH key selection
- Pushes directly without pulling (no remote history conflicts)

> Use this when you created the repo on GitHub with no initial files (no README, no LICENSE).

---

### `new-branch`
**Create and switch to a new Git branch**
- Prompts for a branch name
- Runs `git checkout -b <name>`

---

### `gpull`
**Pull the current branch from origin**
- Detects the current branch automatically
- Runs `git pull origin <branch>`

---

### `gpush`
**Push the current branch to origin**
- Detects the current branch automatically
- Runs `git push origin <branch>`

---

### `clean-system`
**Clean apt cache, user cache, and trash**
- Runs `apt-get clean`, `autoremove`, `autoclean`
- Removes `~/.cache/thumbnails` and trash
- Optionally lists and kills background processes by PID

---

### `install-wordpress`
**Full local WordPress setup**
- Installs PHP, MySQL, and required extensions if missing
- Creates the database and user
- Downloads and configures WordPress
- Generates a `start-server.sh` with automatic port detection (avoids conflicts if 8080 is in use)

---

## SSH Key Convention

All scripts follow the same pattern:

| File | SSH config host | Used in clone URLs |
|------|-----------------|--------------------|
| `~/.ssh/<keyname>` | `Host gh-<keyname>` | `git@gh-<keyname>:user/repo.git` |

Example: key named `sofidev` → `Host gh-sofidev` → clone with `git@gh-sofidev:SofiDevO/repo.git`

---

## Recommended Workflow

1. **First time**: Run `setup-git-users` to configure your Git identity and SSH key
2. **Add a work account**: Run `add-git-user` to add a second SSH user
3. **Clone a repo**: Use `clone-repo` or `clone-project`
4. **New project**: Use `add-repo` to initialize and push
5. **Daily Git**: Use `gpull` / `gpush` / `new-branch`

---

## Troubleshooting

**If scripts don't work after install:**

1. Check they are executable: `ls -la ~/.local/bin/`
2. Check PATH: `echo $PATH | grep .local/bin`
3. Reload shell config: `source ~/.zshrc`
4. Re-run setup: `set-scripts`

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/sofidev)