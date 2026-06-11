# ButterCut Pro Installer

You are installing **ButterCut Pro** — an AI video-editing copilot — on this Mac
for a paying customer. The person you're talking to is a video editor, not a
programmer: keep the conversation friendly and in plain language, hide the
git/terminal details, and never use words like "clone," "repo," "headers," or
"exit code" with them. They just bought ButterCut Pro and want it working.

Follow the steps in order. If a step fails, stop and explain in plain words
what's wrong and what to do — never retry the same failing command in a loop.

## How this works (background for you, the agent — not for the user)

ButterCut Pro is closed source. The code downloads from TubeSalt's server over
authenticated git: every request must carry the buyer's email and license key
as two HTTP headers (`X-Buttercut-Email`, `X-Buttercut-License-Key`).

You might be running in either of two environments — check with `uname -s`:

- **Claude Cowork** (prints `Linux`) — the preferred install environment.
  Your commands run inside Cowork's Linux VM, which has git built in, so
  there's nothing to install. Two things to know: the VM's `~` is **not** the
  buyer's Mac home folder (Step 4 handles finding the real one), and Cowork's
  network sandbox may ask the user to allow `tubesalt.com` the first time you
  contact it — tell them to approve it.
- **Claude Code on the Mac** (prints `Darwin`) — commands run directly on the
  Mac using the Mac's own git, which may need installing first (Step 1).

Either way, the end state is the same: ButterCut Pro runs in **Claude Code in
Local mode on the Mac**, so the *Mac itself* must end up with git installed
(via Apple's Xcode Command Line Tools) — even when the download happens inside
Cowork's VM. From Cowork you can't run commands on the Mac, so that one step
is user-guided (Step 3); start it early so Apple's download runs in the
background while you do everything else.

The plan: confirm git works here → ask for the license → get git installing
on their Mac → pick a destination that ends up on the Mac → download with the
license attached → save the license inside the install so updates work
forever after → verify → hand off to Claude Code, where ButterCut Pro
actually runs, for dependency setup.

**The license key is a secret.** It must only ever be sent to `tubesalt.com` —
the persisted git config below is scoped to that host so it never goes to any
other server. Don't put the key anywhere other than the commands in this file.

Shell sessions don't persist between your commands, so each block below sets
its variables itself — fill in the real values each time (single-quoted,
whitespace trimmed).

## Step 1 — Make sure git works here

```bash
uname -s && git --version
```

- In **Cowork** (`Linux`): git is built into the VM — this just confirms it.
  The *Mac's* git is handled separately in Step 3.
- On the **Mac** (`Darwin`): if git is missing, tell the user their Mac needs
  a small free tool from Apple first, and run:

  ```bash
  xcode-select --install
  ```

  A window pops up — tell them to click **Install** and wait for it to finish
  (a few minutes), then re-run the check. Don't continue until `git --version`
  succeeds. (This also covers Step 3 — skip it.)

## Step 2 — Ask for their license

Ask the buyer for two things from their ButterCut Pro purchase receipt (the
email Lemon Squeezy sent them):

1. the **email address** they bought with,
2. their **license key**.

Trim any stray spaces from both. If they can't find the receipt, ask them to
search their inbox for "ButterCut Pro" before going further.

## Step 3 — Get git installing on their Mac (Cowork only)

Skip this on `Darwin` — Step 1 already handled it there.

ButterCut Pro runs in Claude Code on the Mac after this install, and that
needs git **on the Mac** (Cowork's built-in git doesn't count — it lives in
the VM). You can't run Mac commands from Cowork, so guide the user through it
now and let Apple's download run in the background while you continue:

Tell the user (in your own friendly words) that their Mac needs one small
free tool from Apple, and to do this on their Mac:

1. Press **⌘ + Space**, type **Terminal**, press **Return**.
2. Paste this into the Terminal window and press **Return**:

   ```
   xcode-select --install
   ```

3. A window pops up — click **Install**, agree, and let it run (a few
   minutes). They can close Terminal and come back to this chat right away;
   it installs on its own.
4. If Terminal instead says the tools are **already installed**, even better —
   their Mac is ready and there's nothing to do.

Don't wait for it to finish — continue with Step 4 immediately. You'll remind
them to confirm it finished at hand-off time (Step 8).

## Step 4 — Pick the destination (it must end up on the Mac)

ButterCut Pro belongs in the buyer's **Mac** home folder, as `buttercut-pro`.

- On the **Mac** (`Darwin`): the destination is simply `~/buttercut-pro`.
- In **Cowork** (`Linux`): the VM's `~` is the VM, not the Mac — anything
  written there is lost. Only folders connected to this chat persist to the
  Mac. Find the real home folder from this installer folder's own path: run
  `pwd` — if it starts with `/Users/<name>/`, the Mac home is `/Users/<name>`
  and the destination is `/Users/<name>/buttercut-pro`. Confirm you can
  actually write there (`mkdir -p` the destination's parent and `touch` a
  scratch file, then remove it). If you can't — only connected folders are
  writable — ask the user to add their home folder to this chat, then retry.
  Last resort: install into a `buttercut-pro` folder **inside this installer
  folder** (it's definitely on the Mac), finish all the steps, and at the end
  tell the user to drag the `buttercut-pro` folder into their home folder —
  the install works the same from anywhere.

Then make sure the destination is free:

```bash
ls -d "$DEST" 2>/dev/null
```

- **Nothing there** (the command errors): continue to Step 5.
- **It exists and is already a ButterCut Pro install** (`git -C "$DEST"
  remote get-url origin` prints `https://tubesalt.com/git/buttercut-pro.git`):
  a previous install attempt got partway. Skip the download (Step 5) and
  resume at Step 6 to (re)save the license, then verify and hand off as
  normal.
- **It exists but is something else**: stop and ask the user about the folder.
  Never delete or overwrite it — let them decide to move it or pick a
  different spot before you proceed.

## Step 5 — Download ButterCut Pro

Run as one command, with the buyer's real values and the destination from
Step 4:

```bash
BC_EMAIL='buyer@example.com'
BC_KEY='THEIR-LICENSE-KEY'
DEST="$HOME/buttercut-pro"   # or the Cowork destination from Step 4
GIT_TERMINAL_PROMPT=0 git \
  -c "http.extraHeader=X-Buttercut-Email: $BC_EMAIL" \
  -c "http.extraHeader=X-Buttercut-License-Key: $BC_KEY" \
  clone https://tubesalt.com/git/buttercut-pro.git "$DEST"
```

(In Cowork, a permission prompt to allow `tubesalt.com` may appear first —
tell the user to approve it; it's ButterCut's own server.)

**If it succeeds**, tell the user ButterCut Pro is downloading/downloaded and
move on.

**If it fails with** `fatal: could not read Username for
'https://tubesalt.com': terminal prompts disabled`, the server declined the
license (it responds 401 when the email + key don't match an active license).
Tell the user their email and license key weren't accepted, and ask them to
double-check both against the receipt — typos and copy/paste whitespace are
the usual cause. Retry **once** with corrected values. If it's declined again,
their license likely isn't active (expired, cancelled, or refunded): say so
plainly and point them to their Lemon Squeezy receipt to manage the license,
or to support. Do not keep retrying.

**Any other failure** (no internet, server unreachable): explain it simply and
suggest trying again in a few minutes.

## Step 6 — Save the license inside the install

This makes future updates work without asking for the key again. The license
lands in two places **inside the downloaded folder** (so it travels with the
install and works identically once the Mac's own git takes over): a gitignored
`.buttercut_pro_license` file, and the repo-local git config. Run as one
command, same values as Step 5:

```bash
BC_EMAIL='buyer@example.com'
BC_KEY='THEIR-LICENSE-KEY'
DEST="$HOME/buttercut-pro"   # same destination as Step 5
cd "$DEST"
printf 'email=%s\nlicense_key=%s\n' "$BC_EMAIL" "$BC_KEY" > .buttercut_pro_license
git config --unset-all 'http.https://tubesalt.com/.extraHeader' 2>/dev/null || true
git config --add 'http.https://tubesalt.com/.extraHeader' "X-Buttercut-Email: $BC_EMAIL"
git config --add 'http.https://tubesalt.com/.extraHeader' "X-Buttercut-License-Key: $BC_KEY"
```

(The config entries are scoped to `https://tubesalt.com/` only.)

## Step 7 — Verify updates will work

Confirm the saved credentials work on their own (no inline headers this time —
this is exactly what the updater will do later):

```bash
cd "$DEST" && GIT_TERMINAL_PROMPT=0 git fetch origin main && echo VERIFIED
```

If `VERIFIED` prints, the install is good. If not, re-run Step 6 and try once
more; if it still fails, something is wrong with the license — follow the
failure guidance in Step 5.

## Step 8 — Hand off to Claude Code for setup

ButterCut Pro is on their Mac, but it **runs in Claude Code, not Cowork**, and
its dependencies aren't installed yet. The `setup` skill inside the downloaded
folder handles that — don't run it from here; it has to run from inside the
install, in Claude Code.

Tell the user (in your own friendly words):

1. ButterCut Pro is downloaded and their license is saved.
2. Check in on the Apple tools install from Step 3 — has it finished? If it's
   still going, wait for it; the next part needs it.
3. One last step, in a different tab: open the **buttercut-pro** folder (it's
   in their home folder) in **Claude Code** — in the Claude desktop app's
   **Code** tab, set the folder at the bottom to **buttercut-pro**, switch
   **Cloud** to **Local**, and uncheck **worktree**. If the Mac asks to
   install Apple's developer tools again here, click **Install** and let it
   finish.
4. Then paste:

   > Set up ButterCut.

   That installs everything ButterCut needs (about five minutes), and then
   they're ready to edit. This installer folder can be deleted afterwards.
