# secure-dev-vm

Ansible playbook that turns a fresh Ubuntu 24.04 VM into a development box built on one assumption: sooner or later you will install a malicious package, so the box has to make that event cheap.

It is the companion to the LinkedIn guide "Your dev machine is part of the supply chain" by Flavio Domingos.

## What it sets up

| Role | What it does |
|---|---|
| `base` | Security updates on autopilot, an unprivileged `dev` user with no sudo, safer git defaults |
| `ssh_hardening` | Keys only, no root login, no agent forwarding, optional FIDO2-only keys |
| `node_toolchain` | Node.js 24, npm 12 and pnpm 11. Unreviewed install scripts fail the install, and new versions wait 3 days |
| `containers` | Rootless Podman. No Docker daemon, no `docker.sock` |
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

## Daily use

- After cloning anything: `repo-inspect` before `npm install`, before opening it in an editor, and before starting an agent in it.
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
- The Claude Code managed settings keys follow the docs as of late 2026. Check them against the current settings reference after upgrades.

## License

MIT
