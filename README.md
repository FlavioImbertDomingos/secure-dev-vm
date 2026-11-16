# secure-dev-vm

Ansible playbook that turns a fresh Ubuntu 24.04 VM into a development box built on one assumption: sooner or later you will install a malicious package, so the box has to make that event cheap.

It is the companion to the LinkedIn guide "Your laptop is part of the supply chain" by Flavio Domingos.

## What it sets up

| Role | What it does |
|---|---|
| `base` | Security updates on autopilot, an unprivileged `dev` user with no sudo, safer git defaults |
| `ssh_hardening` | Keys only, no root login, no agent forwarding, optional FIDO2-only keys |
| `node_toolchain` | Node.js 24, npm 12 and pnpm 11. Unreviewed install scripts fail the install, and new versions wait 3 days |
| `containers` | Rootless Podman. No Docker daemon, no `docker.sock` |
| `security_tools` | Trivy, TruffleHog, Semgrep and NVIDIA SkillSpector at pinned, checksum-verified versions, a `preflight` command that runs every check on a repo, and a pre-commit hook that blocks commits containing secrets |
| `credential_audit` | auditd logs every read of `~/.ssh`, `~/.aws`, `~/.npmrc`, `gh` and `gcloud` configs, plus changes to shell startup files. `sudo cred-audit` shows the report |
| `agent_guardrails` | `repo-inspect` command, and root-owned Claude Code managed settings that block reading secret files and bypass mode |
| `egress_allowlist` | Default-deny outbound firewall. Only a local proxy may leave, and it only connects to allowlisted domains |

## Quick start

You need [Multipass](https://multipass.run) (macOS, Windows, Linux) and Ansible on your laptop (`pipx install ansible-core`).

```bash
git clone https://github.com/FlavioImbertDomingos/secure-dev-vm
cd secure-dev-vm

# 1. Put your public key in cloud-init.yaml, then create the VM
multipass launch 24.04 --name devbox --cpus 4 --memory 8G --disk 40G --cloud-init cloud-init.yaml
multipass info devbox            # note the IPv4 address

# 2. Set the address in inventory.ini and your key(s) in group_vars/all.yml
#    (dev_user_ssh_pubkeys). A YubiKey key is best: ssh-keygen -t ed25519-sk

# 3. Apply and verify
ansible-playbook site.yml
ansible-playbook verify.yml

# 4. Work inside the VM
ssh dev@<vm-ip>
```

Point VS Code Remote-SSH, JetBrains Gateway, or Neovim over SSH at `dev@<vm-ip>`. Your editor UI stays on the laptop; code, dependencies and agents run in the VM.

Run one part only with tags: `ansible-playbook site.yml --tags egress`.

## Windows

Everything the playbook sets up lives in the Linux VM, so the VM and the controls are the same on Windows. Three things differ.

Create the VM with Multipass (it uses Hyper-V on Windows Pro and Enterprise, VirtualBox on Home) or with Hyper-V directly. Don't use WSL as the dev environment. By default WSL mounts your Windows drive inside Linux at `/mnt/c` and lets Linux run Windows programs, so a malicious package can read your Windows user folder. All WSL distros also share one kernel. If you use WSL for anything, turn those off in `/etc/wsl.conf`:

```ini
[automount]
enabled = false

[interop]
enabled = false
appendWindowsPath = false
```

Ansible doesn't run natively on Windows. Run the playbook inside the VM against itself, with Ansible from Ubuntu's own repository:

```bash
multipass shell devbox
sudo apt-get update && sudo apt-get install -y ansible-core git
git clone https://github.com/FlavioImbertDomingos/secure-dev-vm && cd secure-dev-vm
# set dev_user_ssh_pubkeys in group_vars/all.yml, then:
ansible-playbook -i inventory-local.ini site.yml
ansible-playbook -i inventory-local.ini verify.yml
```

For the SSH key, Windows' built-in OpenSSH 9.x supports FIDO2 keys, so the `ssh-keygen` command in the YubiKey section works in PowerShell without Homebrew (check with `ssh -V`). Set the key's PIN in Settings > Accounts > Sign-in options > Security key, or with Yubico's `ykman` for Windows. Keep Node, Python and AI agents off Windows itself; the point is that nothing you install runs where your browser sessions and saved credentials live.

## Daily use

- After cloning anything: run `preflight` in the folder before `npm install`, before opening it in an editor, and before starting an agent in it. It runs `repo-inspect`, TruffleHog, Trivy and SkillSpector and prints a pass/fail summary. Add `--code` to include a Semgrep scan.
- Before installing an agent skill from anywhere: `skillspector scan <path-or-url> --no-llm`. A clean result lowers the risk; it doesn't prove the skill is safe.
- A commit is blocked by the secret hook: remove the secret and rotate it if it was real. For a confirmed false positive, `git commit --no-verify`. Repos that set their own `core.hooksPath` (Husky, for example) skip the shared hook, so run `trufflehog git file://. --no-update --no-verification` in those before pushing.
- A package needs its install script (esbuild, sharp, and similar): list pending ones with `npm approve-scripts --allow-scripts-pending`, read them, then `npm approve-scripts <pkg>`. The approval is pinned to that version. pnpm uses `allowBuilds` in `pnpm-workspace.yaml`.
- An urgent security fix is newer than the gate: `npm install pkg@x.y.z --min-release-age=0` for that one package. pnpm users can run `pnpm audit --fix`, which adds patched versions to `minimumReleaseAgeExclude`.
- A tool needs a new domain: add it to `egress_allowlist_extra` and rerun with `--tags egress`. Blocked attempts are in `/var/log/tinyproxy/tinyproxy.log` and the kernel log (`journalctl -k | grep devbox-egress-drop`).
- Something feels wrong: `sudo cred-audit recent`, then rotate tokens and rebuild. Rebuilding is the point of the design:

```bash
multipass delete --purge devbox
multipass launch 24.04 --name devbox --cpus 4 --memory 8G --disk 40G --cloud-init cloud-init.yaml
ansible-playbook site.yml
```

Keep work in git and pushed, and the VM holds nothing you would miss.

## Scanner versions

The scanners are security tools, and security tools get attacked too: in March 2026 a stolen credential was used to publish a malicious Trivy 0.69.4 that stole CI secrets. So `roles/security_tools/defaults/main.yml` pins every tool to a release that was at least 3 days old when picked, and records the SHA-256 of each download. A release that is replaced or re-tagged later fails the install. Semgrep's Python dependencies are resolved with `uv --exclude-newer` (nothing newer than 3 days), and SkillSpector's come from its own hash-checked `uv.lock` at a pinned commit.

To upgrade a tool: pick a release older than 3 days, copy the checksums for the `x86_64` and `aarch64` Linux files from its release page, update the version and both hashes, then rerun `ansible-playbook site.yml --tags tools` and `verify.yml`.

Trivy and Semgrep run with telemetry and update checks turned off.

## YubiKey

Keep your SSH login key on a YubiKey so it can't be copied off your laptop:

```bash
brew install openssh ykman          # macOS only: Apple's ssh can't use FIDO2 keys
ykman fido access change-pin
ssh-keygen -t ed25519-sk -O resident -O verify-required -O application=ssh:devbox -f ~/.ssh/id_ed25519_sk_yubi
```

Put the `.pub` file in `dev_user_ssh_pubkeys`, and once login works set `ssh_require_sk_keys: true`. Register a second YubiKey as a backup, or keep `multipass shell devbox` as your way back in.

Agent forwarding is off, so the YubiKey can't sign commits made inside the VM. Give the VM its own signing key and replace it on every rebuild:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/signing
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/signing.pub
git config --global commit.gpgsign true
```

Add `~/.ssh/signing.pub` on GitHub as a Signing key.

## Secrets

Nothing in this repo stores secrets, and the VM should hold as few as possible:

- Dev and test credentials only. Production keys belong in CI and your secrets manager.
- Short-lived credentials where the provider supports them (AWS IAM Identity Center, `gh auth login` with fine-grained tokens, Vault dynamic secrets).
- Load secrets per command from a manager (`op run`, `vault kv get`) instead of leaving `.env` files on disk.
- Publish packages from CI with OIDC trusted publishing, so no long-lived npm token exists on any laptop.

## Limits

Be clear about what this does not stop:

- A stealer can still read what the `dev` user can read and send it to an allowlisted domain. GitHub is allowlisted and has been used for exfiltration before (the 2025 Nx compromise pushed stolen data to public GitHub repos). Scoped, short-lived tokens limit the damage.
- DNS lookups still leave through the system resolver.
- A kernel or hypervisor escape defeats any VM. Keep the host and the VM patched.
- Scanners catch known problems. A brand-new malicious package, or a skill written to slip past SkillSpector's checks, can still pass `preflight`. The VM, the egress allowlist and short-lived tokens are what limit the damage when that happens.
- The Claude Code managed settings keys follow the docs as of late 2026. Check them against the current settings reference after upgrades.

## License

MIT
