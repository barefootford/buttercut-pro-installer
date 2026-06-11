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
as two HTTP headers (`X-Buttercut-Email`, `X-Buttercut-License-Key`). You will:

1. check that git is available,
2. ask the buyer for their purchase email and license key,
3. download ButterCut Pro to `~/buttercut-pro` using those credentials,
4. save the license inside the install — a gitignored `.buttercut_pro_license`
   file plus host-scoped git config — so updates keep working from then on,
5. hand the buyer off to ButterCut Pro's own `setup` skill for dependencies.

**The license key is a secret.** It must only ever be sent to `tubesalt.com` —
the persisted git config below is scoped to that host so it never goes to any
other server. Don't put the key anywhere other than the commands in this file.

Shell sessions don't persist between your commands, so each block below sets
`BC_EMAIL` / `BC_KEY` itself — fill in the buyer's real values each time
(single-quoted, whitespace trimmed).

## Step 1 — Check for git

```bash
git --version
```

If that works, move on. If git is missing, tell the user their Mac needs a
small free tool from Apple first, and run:

```bash
xcode-select --install
```

A window pops up — tell them to click **Install** and wait for it to finish
(a few minutes), then re-run the check. Don't continue until `git --version`
succeeds.

## Step 2 — Ask for their license

Ask the buyer for two things from their ButterCut Pro purchase receipt (the
email Lemon Squeezy sent them):

1. the **email address** they bought with,
2. their **license key**.

Trim any stray spaces from both. If they can't find the receipt, ask them to
search their inbox for "ButterCut Pro" before going further.

## Step 3 — Make sure the destination is free

ButterCut Pro installs to `~/buttercut-pro`.

```bash
ls -d ~/buttercut-pro 2>/dev/null
```

- **Nothing there** (the command errors): continue to Step 4.
- **It exists and is already a ButterCut Pro install** (`git -C ~/buttercut-pro
  remote get-url origin` prints `https://tubesalt.com/git/buttercut-pro.git`):
  a previous install attempt got partway. Skip the download (Step 4) and
  resume at Step 5 to (re)save the license, then verify and hand off as
  normal.
- **It exists but is something else**: stop and ask the user about the folder.
  Never delete or overwrite it — let them decide to move it or pick a
  different state before you proceed.

## Step 4 — Download ButterCut Pro

Run as one command, with the buyer's real values:

```bash
BC_EMAIL='buyer@example.com'
BC_KEY='THEIR-LICENSE-KEY'
GIT_TERMINAL_PROMPT=0 git \
  -c "http.extraHeader=X-Buttercut-Email: $BC_EMAIL" \
  -c "http.extraHeader=X-Buttercut-License-Key: $BC_KEY" \
  clone https://tubesalt.com/git/buttercut-pro.git ~/buttercut-pro
```

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

## Step 5 — Save the license inside the install

This makes future updates work without asking for the key again. Run as one
command, same values as Step 4:

```bash
BC_EMAIL='buyer@example.com'
BC_KEY='THEIR-LICENSE-KEY'
cd ~/buttercut-pro
printf 'email=%s\nlicense_key=%s\n' "$BC_EMAIL" "$BC_KEY" > .buttercut_pro_license
git config --unset-all 'http.https://tubesalt.com/.extraHeader' 2>/dev/null || true
git config --add 'http.https://tubesalt.com/.extraHeader' "X-Buttercut-Email: $BC_EMAIL"
git config --add 'http.https://tubesalt.com/.extraHeader' "X-Buttercut-License-Key: $BC_KEY"
```

(`.buttercut_pro_license` is gitignored, and the config entries are scoped to
`https://tubesalt.com/` only.)

## Step 6 — Verify updates will work

Confirm the saved credentials work on their own (no inline headers this time —
this is exactly what the updater will do later):

```bash
cd ~/buttercut-pro && GIT_TERMINAL_PROMPT=0 git fetch origin main && echo VERIFIED
```

If `VERIFIED` prints, the install is good. If not, re-run Step 5 and try once
more; if it still fails, something is wrong with the license — follow the
failure guidance in Step 4.

## Step 7 — Hand off to setup

ButterCut Pro is on their Mac, but its dependencies aren't installed yet. The
`setup` skill inside the downloaded folder handles that — don't run it from
here; it has to run from inside the install.

Tell the user (in your own friendly words):

1. ButterCut Pro is downloaded and their license is saved.
2. One last step: open the **buttercut-pro** folder (it's in their home
   folder) in this app — in the Claude desktop app's **Code** tab, set the
   folder at the bottom to **buttercut-pro**, switch **Cloud** to **Local**,
   and uncheck **worktree** — then paste:

   > Set up ButterCut.

3. That installs everything ButterCut needs (about five minutes), and then
   they're ready to edit. This installer folder can be deleted afterwards.
