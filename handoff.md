# Linux Fleet Handoff

Last updated: 2026-11-08 (second session, ended on the Garuda desktop; the user is continuing from another computer)

## How to use this file

This is the running record for the project. Read it fully at the start of a session. It supersedes any memory notes an earlier session kept, because those live on one machine and this file travels. Update it at the end of each session: decisions into the table, anything undecided into "Open questions", and the "Current state" sections to match reality. Keep it factual and short.

## Goal

Build the host-management layer for a Water SCADA fleet: 200+ industrial PCs, one per site, connected to critical-infrastructure PLCs. Each PC runs Ignition in Docker on an Ubuntu or Debian LTS golden image.

About 60 PCs are already deployed on Windows with RDP. They move to Linux once a few units on the new standard are established, and SSH then replaces RDP for hands-on access.

`~/Ansible` is the user's early practice repo for this work. They test it on VMs.

## Working preferences

- Don't change the user's playbook or system config unless they ask. They wanted `local.yml` reviewed, not edited, and they apply fixes themselves.
- They are learning Ansible, so explain what things do and why.
- Size advice for 200+ critical sites, not a home lab.
- They said not to worry about cellular details or lease-timer alarms. Ignition alarms cover lease timers. Don't raise those again unless asked.

## Plan documents

- Water SCADA Fleet Architecture (diagram), now in the user's org and private: https://claude.ai/artifact/RM1aP3fio7FDfVaKdpvQXG
  - The old copy, https://claude.ai/artifact/7zajgsuXcinN1rAJjC6zqT, reads as created outside the org. It's frozen at version 11. Don't edit it.
- Ignition Image and License Flow: https://claude.ai/artifact/No8gPgHpKcv6qvmNgJX9af

The old diagram copy and the license flow are set to "anyone with the link". The new diagram is private until shared. The license flow artifact hasn't been recreated in the org yet.

## Decisions made

| Area | Decision |
|---|---|
| Host config | `ansible-pull -U` on a cron task. The user chose cron over a systemd timer. No push over SSH. |
| Hands-on access | System admin workstation has SSH to field PCs for troubleshooting. |
| Admin workstations | Primary and secondary (a spare with the same role), both in the Supervisory zone. Both are in the same user group. User access rules for the workstation area are TBD (noted on the diagram). Neither has proxy or internet access. They monitor and control the apt mirror, Zabbix, GitLab, Portainer and Ansible. Explicitly no link to the MQTT broker. |
| Service VMs | Every service gets its own VM: Portainer, Zabbix, apt mirror, and so on. |
| sudo | Passwordless sudo is removed. Admins use a sudo password that the system admins set. |
| SSH server | OpenSSH from the distro packages, via the apt mirror. Plain `authorized_keys` for now; SSH certificates deferred until later. |
| Firewall | Managed by the cell router at each site. |
| Monitoring | Zabbix agent 2 in active mode, outbound only. SSH is not used for monitoring. |
| Package updates | Internal apt mirror using aptly and nginx, fed by approved transfer. |
| Gateway backup | OperaMetrix Git module commits the gateway and Perspective project folders from the head gateway to GitLab. Commit and push only, drawn one-way. |
| Containers | Portainer Edge stacks, images from the GitLab registry. |
| Provisioning | New PCs are built on a bench in the office on a staging VLAN, not at the site. The golden image is a recipe (Ubuntu `autoinstall` / Debian preseed kept in GitLab), not a cloned disk. Proposed, not yet built; details below. |
| Apt mirror design | Filtered `aptly` mirror outside, delta transfer of `.deb` files, plain `aptly` local repo inside, one distribution published per ring. Proposed, agreed in discussion, not yet built; details below. |

## Credentials map

Who holds what. Nothing secret goes in Git.

| Credential | Purpose | Scope | Where it lives |
|---|---|---|---|
| GitLab deploy token A | `ansible-pull` clones the playbook repo | `read_repository`, Ansible project only | Root-only `/root/.git-credentials` on each PC, placed at imaging |
| GitLab deploy token B | Portainer pulls Ignition images | `read_registry`, image group only | In Portainer under Registries. Never on the PCs. Passed to the Edge Agent per pull, not persisted. |
| GitLab deploy token C (optional) | Only if Edge stacks are sourced from a Git repo | `read_repository` | In Portainer. Git-sourced Edge stacks may be a Portainer Business Edition feature; confirm before relying on it. |
| Portainer Edge ID + Edge Key | Edge Agent identifies its site | Per environment | On the PC, placed at imaging |
| Zabbix PSK | Agent to server encryption | Per host | On the PC, placed at imaging |
| Sudo password hash | Admin `sudo` on the PC | Per host | On the PC, placed at imaging from the vault. Can't go through `ansible-vault` under pull (every PC would need the vault password). |
| LUKS recovery passphrase | Disk unlock when TPM unlock fails or the board is replaced | Per host | Vault only |
| Ignition activation / EAM | License lease from the head gateway | Per gateway | Placed at imaging; see the license flow artifact |
| aptly signing key | Signs the published apt repo | One | Inside only, on the apt mirror VM |
| Admin SSH private keys | Admin login to PCs | One per admin, `ed25519`, passphrase | Only on the admin workstations |

Keep tokens A and B separate so rotating or revoking one doesn't break the other. Give every token an expiry and put renewals on one calendar. The field PC's outbound allow list must include the registry endpoint, not just GitLab's web port.

## Current state of `~/Ansible`

- Remote: https://github.com/awilliamson26/Ansible-Work.git (public). Latest commit `4e3609f`, in sync with origin.
- `local.yml` and `vars/main.yml` both parse as valid YAML. The syntax errors from the review are fixed.
- Parse-checked only. Ansible isn't installed on this desktop, and no VM run results have been reported in this session.
- Tasks: packages, timezone, admin user, passwordless sudo for the `sudo` group, message of the day, ssh service, cron for `ansible-pull`.

## Recommended next steps for the playbook

None of these are applied yet.

1. Point `ansible_pull_repo` at the internal GitLab. Field PCs have no internet.
2. Use a read-only GitLab deploy token in a root-only `/root/.git-credentials`, not in the URL.
3. Replace `state: latest` with `present` and pinned versions from the apt mirror.
4. Replace the cron job with:
   ```yaml
   job: "/usr/bin/flock -n /run/ansible-pull.lock /usr/bin/ansible-pull -s 900 -o -C ring1 -U {{ansible_pull_repo}} >> /var/log/ansible-pull.log 2>&1"
   ```
5. Add a logrotate rule for that log.
6. Add a nightly run without `-o`, because `-o` doesn't retry a failed run until the next commit.
7. Use branches per rollout group, for example `ring1`, `ring2`, `stable`, to match the 5 sites, 20%, everyone rollout.
8. Add tasks for Docker, the Portainer Edge Agent and the Zabbix agent (`StartAgents=0`, remote commands off, TLS).
9. Harden SSH (decided: OpenSSH, plain keys). `admin` currently has no way to log in. The agreed shape:
   - One named account per admin, not a shared `admin`. Keys are `ed25519` with a passphrase, generated on the workstations.
   - Public keys in the repo, applied with `ansible.posix.authorized_key` and `exclusive: true`, so removing a key from Git revokes it on the next pull.
   - `sshd` settings in a drop-in under `/etc/ssh/sshd_config.d/` via `template`, with `validate: /usr/sbin/sshd -t -f %s` and a restart handler. Contents: `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin no`, `PubkeyAuthentication yes`, `AllowGroups <admin group>`, `MaxAuthTries 3`, `LoginGraceTime 30`, `X11Forwarding no`, `AllowAgentForwarding no`, `LogLevel VERBOSE`. Leave `AllowTcpForwarding` on: `ssh -L 8088:localhost:8088` reaches the Ignition gateway UI, which replaces most RDP use.
   - Skip `fail2ban` and non-standard ports; password auth is off and the cell router restricts sources.
10. Remove passwordless sudo (decided). Use a sudo password that the system admins set. The user is writing this change.
11. Add `--verify-commit` with signed commits and protect the branches. Anyone who can push to a pulled branch gets root on those PCs.
12. Bake `git`, `ansible` and the deploy credential into the golden image. Install `ansible` from `apt`, because the job calls `/usr/bin/ansible-pull`.

## Open questions and loose ends

- **Router reachability:** the user said to drop cellular details, so don't push this. Inbound SSH from the workstation does still depend on the sites being reachable from it.
- **License flow text is out of date in two places:**
  - It says field PCs accept no inbound management connections. It now needs "except SSH from the admin workstation".
  - It says to keep gateway backups out of Git. Check what the module commits. The README describes project resources plus tag, theme and image snapshots. Search the repo for credentials if the gateway folder holds more.
- **Gateway settings backup:** the module's README doesn't list database connections, device connections or security settings, and commits are manual. Keep scheduled gateway backups outside Git.
- **Secrets on field PCs:** the Ignition activation token can't go in Git. Placing it during imaging was the suggested answer.
- **Admin workstations:** the secondary covers losing the primary. User access for the area is TBD; both still need MFA and admin-only use.
- **SSH keys:** plain keys chosen for now. Revocation latency equals the pull interval, so revisit certificates once the fleet grows. Golden image must not ship host keys (regenerate on first boot); keep a `known_hosts` for the fleet in Git for both workstations.
- **Sudo password via ansible-pull:** an `ansible-vault` secret can't be used, because every field PC would need the vault password. Preferred answer: place the hash during imaging from the vault (provisioning step 2) and have the playbook read it from the root-only secrets file via `vars_files`. Not formally decided.
- **Portainer edition:** Git-sourced Edge stacks, async Edge mode and bulk pre-staging may be Business Edition features, licensed per node. At 200+ nodes that is a real cost. Confirm which features the design depends on in CE. Pasted compose in CE Edge stacks works and needs only the registry token.
- **Disk encryption unlock:** TPM-backed LUKS via `clevis` is the proposal so sites reboot unattended. Not tested on the chosen Ubuntu/Debian release yet.
- **Ubuntu vs Debian:** still open. `autoinstall` (cloud-init YAML) favours Ubuntu for the recipe approach; Debian preseed also works.
- **Web UI for Ansible:** not needed under pull. ARA was suggested for run history, not decided.
- **Pending offers the user hasn't answered:** recreate the license flow artifact in the org with its two stale statements fixed; restrict or delete the old public diagram copy.

## Apt mirror design

Goal set by the user: minimise how many packages are stored. Commands follow aptly's documented workflow and have not been run. Expected size: 2–3 GB initial, a few hundred MB a month after pruning; a 100 GB disk on the mirror VM is plenty.

**Outside (internet-connected), aptly as a filtered downloader**

- `aptly.conf`: `"architectures": ["amd64"]`, `downloadSourcePackages: false`, `dependencyFollowRecommends: false`, `dependencyFollowSuggests: false`, `dependencyFollowSource: false`, `dependencyFollowAllVariants: false`, `skipContentsPublishing: true`. The architecture restriction alone halves the size.
- Build the filter from the real image: on a VM built from the golden image after Ansible has run, `dpkg-query -W -f='${binary:Package}\n' | sort > scada-packages.txt`. Join with `|` into the aptly query.
- `aptly mirror create -filter="$FILTER" -filter-with-deps` for `noble`, `noble-updates`, `noble-security` (main universe), Docker (`docker-ce | docker-ce-cli | containerd.io | docker-buildx-plugin | docker-compose-plugin`, `noble stable`) and Zabbix (`zabbix-agent2 | zabbix-agent2-plugin-*`). The explicit list is exact; `-filter-with-deps` catches new dependencies an update introduces.
- Later runs: `aptly mirror update -skip-existing-packages`.
- Kernel and firmware are most of the bytes. Prefer the GA kernel (`linux-generic`) over HWE unless the IPC hardware needs it. Keep `linux-firmware`; a missing NIC blob is a truck roll.

**Transfer: delta only**

- Snapshot after each update. `aptly snapshot diff <old> <new>` lists what changed; copy only those files from `~/.aptly/pool/`. First load is the full set, then a few hundred MB a month.

**Inside: a plain local repo, no mirror definitions**

- `aptly repo create -distribution=noble -component=main scada-noble`, then `aptly repo add scada-noble /transfer/<date>/*.deb`.
- `aptly snapshot create scada-<date> from repo scada-noble`, `aptly publish snapshot -gpg-key=<inside-key> -distribution=noble scada-<date> ring1`. Promote with `aptly publish switch ring2 scada-<date>`, then `stable`.
- One repo, one distribution, one `sources.list` line per PC. Ubuntu pockets, Docker and Zabbix all collapse into it.
- Serve `public/` with nginx. Ansible installs the public key and the ring's repo line.

**Pruning (where the space actually goes)**

- Keep about four snapshots (one per ring plus a rollback). `aptly snapshot drop` older ones.
- Then `aptly db cleanup`; nothing is freed until it runs.
- Remove superseded versions before snapshotting, e.g. `aptly repo remove scada-noble 'Name (% linux-image-6.8.0-*), $Version (<< 6.8.0-50)'`.
- Script the prune after each promotion to `stable`.

**Process rule:** when a package is added to the fleet, add it to `scada-packages.txt` and run a mirror update before the Ansible change ships, or `ring1` fails the install. When hardware revisions change, build one unit, diff `dpkg -l` against the list, update the filter, then release.

## Provisioning workflow for a new PC

Proposed in discussion, not yet built. Principle: bench in the office, recipe not clone, secrets enter once from the vault.

0. **Before unboxing.** Site ID and hostname (e.g. `wtp-042`), IP and VLAN in `host_vars/` in the Ansible repo. Portainer Edge environment created (Edge ID + Key). Zabbix host created (PSK). Per-host sudo hash and LUKS recovery passphrase generated into the vault. A provisioning script on the admin workstation renders the generic `autoinstall` `user-data` plus a per-unit secrets file from these.
1. **Bench, firmware.** BIOS password, boot order, AC power recovery on, hardware watchdog on, TPM 2.0 on. Record serial and MAC in the inventory.
2. **Bench, unattended install** from the stock Ubuntu Server ISO with the `user-data` (USB first, PXE/HTTP boot on the staging VLAN later). It does: LUKS root unlocked by TPM (`clevis` TPM2 pin on Ubuntu's `initramfs-tools`; recovery passphrase in the vault); apt sources pointed at the internal mirror's `stable` ring with the signing key; the minimal package set plus `git` and `ansible`; admin accounts with public keys; hostname from the per-unit file; `late-commands` copying the per-unit secrets to root-only paths (`/etc/scada/site.env`, `/root/.git-credentials`) and writing the `ansible-pull` cron entry. Fresh install means unique host keys and `machine-id` by construction.
3. **First boot, Ansible takes over** (against `stable`, a new unit is not a canary): sshd drop-in, sudo policy, MOTD, timezone, logrotate, watchdog daemon, Docker pinned from the mirror, Portainer Edge Agent (reads Edge ID/Key from the secrets file), Zabbix agent 2 active mode with the PSK. From here the unit is managed like the rest; `host_vars/<hostname>.yml` carries site specifics.
4. **Bench, application and acceptance.** Assign the Edge environment to its site group; the Edge stack deploys Ignition from the GitLab registry. EAM agent registers with the head gateway; approve it; lease issued; MQTT Transmission connects. Checklist, every unit: `ansible-pull` log clean, `docker ps` shows the gateway, Zabbix green, EAM licensed; `ssh-keyscan` into the fleet `known_hosts` and commit; SSH in from the workstation and `sudo -v` with the vault password; pull the power twice and confirm unattended unlock and return; 24–48 h burn-in on the staging VLAN; label; ship.
5. **Site.** Mount, cable the cell router and PLC network, power on. It checks in to Zabbix and Portainer on its own. If it hasn't within a set time, troubleshoot while someone is still there.

Boundaries: the installer owns disk, apt sources and accounts (SSH must work before Ansible so a failed first run is recoverable); Ansible owns everything else so day-0 and day-2 are the same code; Portainer owns the application; secrets enter only in step 2.

## Editing the diagram

- It's a Design canvas artifact. Use the Artifact tool to read `project/canvas.json` and `project/Main.dc.html`, and read `artifact-type/reference/diagrams.md` before changing layout.
- Canvas is 2320 x 1456. Every box, arrow segment and label is a separately placed element. Arrows don't follow boxes, so recompute them when moving anything.
- Logos are uploaded as assets of the new artifact (`RM1aP3fio7FDfVaKdpvQXG`). Reuse the `/_blob/...` URLs already in its `Main.dc.html`; the old artifact's blob ids don't resolve there.
- The Cirrus Link logo is a CSS crop of the icon at the far left of the source image. Keep `left: -8px; top: -111px` at 267px.
- The live page is behind a Cloudflare bot check, so automated browsers can't load it. Don't try to get around that.
- To check layout, render locally: strip the `support.js` line from `Main.dc.html` and screenshot with headless Chromium (`chromium --headless=new --no-sandbox --hide-scrollbars --window-size=2320,1456 --screenshot=out.png file://…`). Logo boxes render empty locally because `/_blob/` isn't served; that's expected. No Playwright needed.
- Version 3 of the new artifact (current) was rendered and checked: two-workstation group in the Supervisory band, "User access: TBD" note in its header, "Each service runs on its own VM" in Operations, admin SSH line leaving the group's right edge, monitoring drop to the management bus. Everything fits.
- Known leftover: the diagram doesn't show the cell routers, because the user chose not to add that information.
