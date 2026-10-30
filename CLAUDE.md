# Personal fork: Alt+Tab per workspace on COSMIC

This fork of `pop-os/launcher` carries one patch on the branch
`cosmic-alt-tab-current-workspace`. It is for personal use on Pop!_OS with
COSMIC. It only needs the `cosmic-toplevel` plugin, so nothing under `/usr`
is replaced.

## Desired behavior

- `Alt+Tab` switches only between windows on the current workspace.
- `Super+Tab` and `Super+Shift+Tab` open the COSMIC workspace overview.
- Text typed in the launcher still finds windows on every workspace.
- A window with missing workspace data stays listed, so it can't become
  unreachable.

COSMIC has no setting for this. Alt+Tab runs `cosmic-launcher alt-tab`, which
sends an empty search to pop-launcher, so the plugin filters empty queries.
The launcher's empty window list is filtered too, which is accepted.

## Install

Run from a COSMIC session in the repo root:

```sh
cargo test -p pop-launcher-plugins cosmic_toplevel
cargo build --release -p pop-launcher-bin
P=~/.local/share/pop-launcher/plugins/cosmic_toplevel
mkdir -p "$P"
install -m755 target/release/pop-launcher-bin "$P/cosmic-toplevel"
cp plugins/src/cosmic_toplevel/plugin.ron "$P/plugin.ron"
pkill -x pop-launcher
```

The binary picks its role from its file name, so it must be named
`cosmic-toplevel`. pop-launcher loads plugins by name and looks in the user
directory first, so this one overrides the system copy and apt leaves it alone.
`cosmic-launcher` starts pop-launcher again the next time it is opened.

The shortcuts aren't in this repo. Add these entries to the map in
`~/.config/cosmic/com.system76.CosmicSettings.Shortcuts/v1/custom`, keeping any
that already exist:

```ron
(modifiers: [Super], key: "Tab"): System(WorkspaceOverview),
(modifiers: [Super, Shift], key: "Tab"): System(WorkspaceOverview),
```

## Rebase after a COSMIC update

Remotes: `origin` is `pop-os/launcher` and `fork` is this repository. In a new
clone, add `origin` by hand, because a clone of the fork names the fork `origin`.

1. Compare the commit in the suffix of `dpkg -l pop-launcher` with
   `origin/master`. If upstream is ahead of the installed package, rebase onto
   the installed commit, since it's the one known to match the desktop.
2. Run `git fetch origin`, then `git rebase <commit>` on the patch branch.
3. Run the install steps above, then `git push --force-with-lease fork`.

If the patch stops working after a clean rebase, check these assumptions:

- `cosmic-launcher` (`src/app.rs`) still sends `Request::Search(String::new())`
  for Alt+Tab.
- `ToplevelInfo.workspace` in `cosmic-client-toolkit` is still populated, which
  requires this plugin to bind the workspace manager through `WorkspaceState`.

## Verify

Open windows on two workspaces, then run the built binary directly under the
name `cosmic-toplevel`, from a temporary directory:

```sh
(sleep 2; echo '{"Search":""}'; sleep 1; echo '{"Search":"e"}'; sleep 1; echo '"Exit"') \
  | timeout 8 ./cosmic-toplevel
```

The output is one JSON line per window, and `"Finished"` ends each search. The
empty search must list only the current workspace. The system plugin at
`/usr/lib/pop-launcher/plugins/cosmic_toplevel/cosmic-toplevel` lists all of
them. Confirm the running copy with `pgrep -af cosmic-toplevel`, then have the
user try the real keys, which can't be pressed from a tool.

## Undo

Delete `~/.local/share/pop-launcher/plugins/cosmic_toplevel`, remove the two
shortcut entries, and run `pkill -x pop-launcher`.
