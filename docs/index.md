---
layout: page
title: Obsidian + Syncthing
permalink: /docs/
---

# Obsidian + Syncthing

This page documents the setup for keeping an Obsidian vault available even when the workstation is offline.

## Recommended approach

- Keep the Obsidian vault in a local folder on your main device
- Sync that folder with the workstation using Syncthing
- Point Obsidian at the local folder, not the remote machine

## Why this works

- Obsidian gets a normal local path
- The vault stays usable when the workstation is off
- Changes propagate automatically once both devices are online

## Suggested folder layout

- Local vault: `~/Documents/ObsidianVault`
- Remote mirror on workstation: the same relative folder name inside your synced area

## Notes

- Try to keep one machine as the primary editor to avoid conflicts
- If you edit on both devices, Syncthing may create conflict files
- For always-on access, sshfs is nice, but for an intermittently available workstation, Syncthing is the better fit

## Next steps

1. Create the local vault folder
2. Install and pair Syncthing on both devices
3. Share the vault folder
4. Set `OBSIDIAN_VAULT_PATH` to the local folder
5. Add notes here as the setup evolves
