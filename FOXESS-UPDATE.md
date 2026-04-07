# FoxESS H3 Smart — Update Guide

This fork adds Force Charge and Battery Lock support for the FoxESS H3-Pro/Smart Series Hybrid Inverter.

## Branch Structure

```
master                  ← always in sync with evcc-io/evcc (never commit here)
fox-ess-h3-smart-fix    ← our single commit on top of master
```

## Sync with upstream evcc

```bash
cd ~/GitHub/evcc

# 1. Fetch upstream
git fetch --depth=1 upstream master

# 2. Reset master to upstream
git checkout master
git reset --hard upstream/master
git push origin master --force

# 3. Rebase our branch onto the new master
git checkout fox-ess-h3-smart-fix
git rebase master

# 4. Push (force required after rebase)
git push --force origin fox-ess-h3-smart-fix
```

If rebase has a conflict (only in `fox-ess-h3-smart.yaml`):
```bash
# Accept our version
git checkout --ours templates/definition/meter/fox-ess-h3-smart.yaml
git add templates/definition/meter/fox-ess-h3-smart.yaml
git rebase --continue
git push --force origin fox-ess-h3-smart-fix
```

## Create a New Release

After syncing, bump the version and trigger a new build:

```bash
cd ~/GitHub/evcc
git checkout fox-ess-h3-smart-fix

# Delete old tag if needed
git tag -d fox-ess-h3-smart-v5
git push origin :refs/tags/fox-ess-h3-smart-v5

# Create new tag (increment version)
git tag fox-ess-h3-smart-v6
git push origin fox-ess-h3-smart-v6
```

GitHub Actions builds automatically on tag push.
Download URL after build: `https://github.com/fabian1512/evcc/releases`

## Install on LXC (evcc container)

```bash
ssh -i ~/.ssh/id_rsa_nopass root@192.168.1.10
pct enter 108

wget -O /tmp/evcc.zip https://github.com/fabian1512/evcc/releases/download/fox-ess-h3-smart-vX/evcc-linux-amd64.zip
unzip -o /tmp/evcc.zip -d /tmp/
tar -xzf /tmp/evcc-linux-amd64.tar.gz -C /tmp/
systemctl stop evcc
cp /tmp/evcc /usr/bin/evcc
systemctl start evcc
journalctl -u evcc -f
```

Replace `vX` with the current release version.

## What Our Patch Does

File: `templates/definition/meter/fox-ess-h3-smart.yaml`

Replaces `limitsoc` (Register 46611, broken) with `batterymode` switch:

| Mode | Action |
|------|--------|
| Normal (1) | Disable Group 24, restore MaxSoC (Reg 46620), restore discharge (Reg 46608=500) |
| Hold (2) | Disable Group 24, MaxSoC=100%, lock discharge (Reg 46608=0) |
| Charge (3) | Enable Force Charge via Time Period Group 24 (Regs 48240–48249) |

Key register findings:
- **46608** — Max Discharge Current: requires FC 06 (`writesingle`), NOT FC 10
- **46620** — MaxSoC FromGrid (new in Modbus Protocol V1.05.04.00)
- **48249** — Enable Flag (undocumented, must be set to 1 for Force Charge to work)
- **48000** — Time Period Enable: kept at 1 permanently to preserve user schedules
