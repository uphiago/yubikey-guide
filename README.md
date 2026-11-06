# YubiKey guide: GPG, SSH and 2FA on Windows + WSL2

A practical guide to setting up a YubiKey 5 with OpenPGP subkeys (master key kept offline), FIDO2 2FA and SSH, including USB passthrough to WSL2 with `usbipd-win`.

The guide lives in [`skills/yubikey/SKILL.md`](skills/yubikey/SKILL.md). It's a [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) with safety rules, diagnostics and step-by-step procedures, and it reads fine as a plain guide too.

## Using the skill

```bash
git clone https://github.com/uphiago/yubikey-guide.git
mkdir -p ~/.claude/skills
ln -s "$PWD/yubikey-guide/skills/yubikey" ~/.claude/skills/yubikey
```

Then ask Claude Code something like "help me set up my YubiKey" or "gpg says Bad PIN".

## Before you start

- **Back up before `keytocard`.** After `save`, the private subkeys exist only on the card.
- **Check `ykman openpgp info` before any PIN attempt.** 3 wrong Admin PINs means a reset that wipes the card.
- Factory defaults: User PIN `123456`, Admin PIN `12345678`.
