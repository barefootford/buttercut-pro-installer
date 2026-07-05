# ButterCut Pro Installer

You are installing **ButterCut Pro** — an AI video-editing agent/helper on this
Mac for a paying customer of ButterCut. The person you're talking to is a video
editor. Keep the conversation friendly and in plain language — Some small fraction
of buyers are technical, most aren't so generally keep a non-technical tone. 

**Example:**
Bad: We use Ruby 4, a dynamic programming language that is Object oriented to...
Good: Behind the scenes ButterCut manages video library footage using a small Ruby (programming language) app. We'll install Ruby and other ButterCut dependencies today.

If the user does have a question answer them truthfully and in a way they can understand (use examples).

Follow the steps in order. If a step fails, stop and explain in plain words
what's wrong and what to do — never retry the same failing command in a loop.

## How this works (context for you, the agent, so you can explain any step accurately if asked)

ButterCut Pro is closed source. The code downloads from TubeSalt's server over
authenticated git: every request must carry the buyer's email and license key
as two HTTP headers (`X-Buttercut-Email`, `X-Buttercut-License-Key`).

You are running in **Claude Code on the Mac, in Local mode** — the README
walked the buyer through getting there (Code tab → this folder → Local),
including installing Apple's command line developer tools if
their Mac lacked git (Claude Code prompts for that itself). So git should
already work; Step 1 double-checks.

The plan: confirm git works → ask for the license → check the destination →
download with the license attached → save the license inside the install so
updates work forever after → verify → run ButterCut Pro's own setup in place
to install its dependencies → hand the buyer off to a fresh chat for editing.

The setup steps are **not** in this file on purpose. ButterCut Pro ships its
own setup instructions and updates them constantly; this installer stays thin
and just runs whatever the freshly downloaded install says to do. Never copy
setup steps in here — they would rot the moment ButterCut Pro changes.

**The license key is a secret.** It must only ever be sent to `tubesalt.com` —
the persisted git config below is scoped to that host so it never goes to any
other server. Don't put the key anywhere other than the commands in this file.
If the user asks you can always tell them where their license is stored.

Be transparent: if the user asks what any command does or why, explain it
honestly.

Shell sessions don't persist between your commands, so each block below sets
its variables itself — fill in the real values each time (single-quoted,
whitespace trimmed).

## Step 0 — Give the user an overview of what you're going to do and ask permission

They're probably non-technical (though some small fraction of ButterCut users are programmers), but generally give them a non-technical or very light technical overview of what you're going to do so they can follow along. You can explain ButterCut is an agentic video editing app that behind the scenes relies on having Git for automatic app updates, Ruby for library management, WhisperX for transcription, and FFmpeg for extracting frames and converting files, etc. Write this in plain language, 2-3 sentences, and ask for permission to proceed with installing ButterCut Pro and its dependencies. If they have questions, answer them. If they have additional questions, they can email TubeSalt directly too (andrew@tubesalt.com). If they'd rather not proceed, stop — nothing has been installed, and they can come back anytime or email for help.

## Step 1 — Make sure git works

```bash
uname -s && git --version
```

- Expect `Darwin` plus a git version — then continue to Step 2.
- If it prints `Linux`, you are **not** running on the Mac (this is a cloud or
  Cowork session). Stop and walk the user back to the README flow: in the
  Claude desktop app's **Code** tab, set the folder to this installer folder,
  switch **Cloud** to **Local**, and ask again.
- If git is missing, the check itself makes macOS show a window: *"The 'git'
  command requires the command line developer tools. Would you like to
  install the tools now?"* Tell the user to click **Install**, agree, and let
  it run — about ten minutes; an installer icon may appear on the right side
  of their Dock. (If Claude's own **Install Git** message shows instead,
  they should click **Not now** — Apple's installer is the one that matters.)
  When they say it's done, re-run the check. Don't continue until
  `git --version` succeeds.

## Step 2 — Ask for their license

Ask the buyer for two things from their ButterCut Pro purchase receipt (the
email Lemon Squeezy sent them):

1. the **email address** they bought with,
2. their **license key**.

Trim any stray spaces from both. If they can't find the receipt, ask them to
search their inbox for "ButterCut Pro" before going further.

When confirming what they gave you, show the email in full but only the last
4 characters of the license key — don't echo the full key back into the chat
more than necessary.

## Step 3 — Check the destination

ButterCut Pro belongs in the buyer's home folder, as `~/buttercut-pro`. Make
sure that spot is free:

```bash
DEST="$HOME/buttercut-pro"
ls -d "$DEST" 2>/dev/null
```

- **Nothing there** (the command errors): continue to Step 4.
- **It exists and is already a ButterCut Pro install** (`git -C "$DEST"
  remote get-url origin` prints `https://tubesalt.com/git/buttercut-pro.git`):
  a previous install attempt got partway. Skip the download (Step 4) and
  resume at Step 5 to (re)save the license, then verify and hand off as
  normal.
- **It exists but is something else**: stop and ask the user about the folder.
  Never delete or overwrite it — let them decide to move it or pick a
  different spot before you proceed.

## Step 4 — Download ButterCut Pro

Run as one command, with the buyer's real values:

```bash
BC_EMAIL='buyer@example.com'
BC_KEY='THEIR-LICENSE-KEY'
DEST="$HOME/buttercut-pro"
GIT_TERMINAL_PROMPT=0 git \
  -c "http.extraHeader=X-Buttercut-Email: $BC_EMAIL" \
  -c "http.extraHeader=X-Buttercut-License-Key: $BC_KEY" \
  clone https://tubesalt.com/git/buttercut-pro.git "$DEST"
```

**If it succeeds**, tell the user ButterCut Pro is downloaded and move on.

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

This makes future updates work without asking for the key again. The license
lands in two places **inside the downloaded folder** (so it travels with the
install): a gitignored `.buttercut_pro_license` file, and the repo-local git
config. Run as one command, same values as Step 4:

```bash
BC_EMAIL='buyer@example.com'
BC_KEY='THEIR-LICENSE-KEY'
DEST="$HOME/buttercut-pro"
cd "$DEST"
printf 'email=%s\nlicense_key=%s\n' "$BC_EMAIL" "$BC_KEY" > .buttercut_pro_license
git config --unset-all 'http.https://tubesalt.com/.extraHeader' 2>/dev/null || true
git config --add 'http.https://tubesalt.com/.extraHeader' "X-Buttercut-Email: $BC_EMAIL"
git config --add 'http.https://tubesalt.com/.extraHeader' "X-Buttercut-License-Key: $BC_KEY"
```

(The config entries are scoped to `https://tubesalt.com/` only.)

## Step 6 — Verify updates will work

Confirm the saved credentials work on their own (no inline headers this time —
this is exactly what the updater will do later):

```bash
DEST="$HOME/buttercut-pro"
cd "$DEST" && GIT_TERMINAL_PROMPT=0 git fetch origin main && echo VERIFIED
```

If `VERIFIED` prints, the install is good. If not, re-run Step 5 and try once
more; if it still fails, something is wrong with the license — follow the
failure guidance in Step 4.

## Step 7 — Run ButterCut Pro's own setup

ButterCut Pro is downloaded, but its dependencies (Ruby, Python, FFmpeg,
WhisperX) aren't installed yet. ButterCut Pro carries its own setup
instructions and keeps them current with every release — your job is to run
them, not to know what they are. Don't reproduce any setup steps here.

Setup has to run with the downloaded install as the working directory. Move
into it, then read its setup instructions and follow them top to bottom:

```bash
cd "$HOME/buttercut-pro"
```

Now read `skills/setup/SKILL.md` from that folder (it pulls in
`skills/setup/simple-setup.md` and other files itself) and carry out its steps
from this directory. It installs everything with no Homebrew and no admin
password — about five to ten minutes, mostly downloads. Keep the buyer posted
in plain words while it runs ("installing the video tools now — this takes a
few minutes"), and the same rule applies: if a step fails, stop and explain it
plainly; never loop on a failing command.

When setup reports everything installed and verified, ButterCut Pro is fully
installed and ready to use — move on to the hand-off.

## Step 8 — Hand off to editing

The install is done. The only thing left is to move the buyer out of this
throwaway installer folder and into ButterCut Pro itself, where they'll
actually edit. Tell them (in your own friendly words):

1. ButterCut Pro is installed and ready.
2. To start using it, open a new chat (**Cmd + N**) and, at the bottom of the
   Code tab, set the folder to **buttercut-pro** in their home folder — still
   **Local**, and if a **worktree** option appears, leave it unchecked. (That
   folder is a real git checkout, unlike this installer folder, so the Code tab
   may show git options it didn't show here.)
3. In that chat they can just say what they want — for example, "Tell me about
   ButterCut" to get oriented, or describe the footage they want to cut.
4. This installer folder has done its job and can be deleted.
