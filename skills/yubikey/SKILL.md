---
name: yubikey
description: Set up, test, troubleshoot and maintain a YubiKey for OpenPGP/GnuPG (subkeys on the card, offline master key), SSH (gpg-agent or FIDO2 ed25519-sk), git signing and FIDO2 2FA, including Windows + WSL2 via usbipd-win. Use for "configure my YubiKey", keytocard, card-status, "Bad PIN", PIN counters, unblock/reset, touch policy, backups/restore of ~/.gnupg, renewing expiring subkeys, or moving the master key offline.
---

# YubiKey (OpenPGP + FIDO2) skill

You are helping someone with a hardware key that holds private keys that **cannot be recovered** if the card is reset or locked and the backup is bad. Be careful first, fast second.

## Safety rules (always)

1. **Audit before changing anything.** Start with the read-only commands in [Audit](#audit). Base advice on what they print, not on assumptions.
2. **Check the PIN counter before *every* PIN use** (`ykman openpgp info`). Never guess PINs in a loop. At 1 try left, stop and ask the user.
3. **Never run, without explicit user confirmation for that exact command:** `ykman openpgp reset`, `ykman reset`/`ykman fido reset`, `gpg --delete-secret-keys`, `gpg --delete-secret-and-public-key`, deleting `~/.gnupg/private-keys-v1.d/*`, `key-attr` on filled slots, `set-touch ... fixed`/`cached-fixed`.
4. **Backup before `keytocard`.** `save` after `keytocard` permanently deletes the local private subkey. If no tested backup exists, stop and make one first.
5. **Never overwrite a known-good backup.** New backups get a new dated filename and must be restored and tested before they count.
6. **Don't put secrets in files, logs or chat summaries.** Prefer interactive prompts (pinentry, ykman without `-P/-n/-a`). If the user hands you a PIN/passphrase and asks you to run something, use it only on that command line (e.g. `--pinentry-mode loopback --passphrase ...`) and never write it into docs.
7. **Test on a copy.** Restore backups into `GNUPGHOME=$(mktemp -d)`, never into `~/.gnupg`. Kill that agent (`gpgconf --kill all`) and delete the dir afterwards.
8. **One change at a time.** Say what it does and whether it's reversible before running it.
9. **Distinguish facts from guesses.** "Slots are filled" ≠ "signing works". Only a real sign/decrypt test proves it.

## Key facts

| Thing | Fact |
|---|---|
| Default User PIN / Admin PIN | `123456` / `12345678` (Reset Code unset) |
| Tries | 3 each by default. User PIN blocked → `ykman openpgp access unblock-pin` (Admin PIN or Reset Code). Admin PIN blocked → only `ykman openpgp reset` (**wipes OpenPGP keys**) |
| Which PIN when | User PIN: sign, decrypt, SSH auth. Admin PIN: `keytocard`, `key-attr`, touch policy, retries, unblock. GPG passphrase: protects keys on disk; unrelated to card PINs |
| `Bad PIN` but counter unchanged | The PIN never reached the card. It's a pinentry/agent issue, **not** a wrong PIN. Fix `GPG_TTY`, run `gpg-connect-agent updatestartuptty /bye`, `gpgconf --kill gpg-agent` |
| PIN change gotcha | Extra chars typed after the current PIN are prepended to the new PIN, and the change succeeds anyway |
| `gpg -K` markers | `sec` = master secret present; `sec#` = master offline; `ssb>` = stub, key is on the card |
| OpenPGP vs FIDO2 | Separate apps. Google/WebAuthn 2FA and `ssh-keygen -t ed25519-sk` use FIDO2 (its own PIN). Resetting one does not touch the other |
| ECC on card | YubiKey 5 firmware ≥ 5.2.3: `ed25519` (sign/auth), `cv25519` (encrypt) |
| WSL2 | Windows owns USB. `usbipd attach --wsl` gives it to Linux; **Windows loses it until `usbipd detach`** (so browser 2FA fails while attached) |

## Audit

Read-only, safe to run any time:

```bash
lsusb | grep -i yubico              # USB visible (WSL: needs usbipd attach first)
ykman info                          # model, firmware, enabled apps
ykman openpgp info                  # PIN tries, slots, touch policy, signature policy, KDF
gpg --card-status                   # what GnuPG sees on the card
gpg -K --keyid-format long          # sec/sec#/ssb>
cat ~/.gnupg/gpg-agent.conf ~/.gnupg/scdaemon.conf ~/.gnupg/gpg.conf 2>/dev/null
echo "GPG_TTY=$GPG_TTY SSH_AUTH_SOCK=$SSH_AUTH_SOCK"
```

## Diagnose in layers (stop at the first failing layer)

1. **USB** (WSL): Windows `usbipd list` shows it as Attached? Linux `lsusb` shows `1050:xxxx`?
2. **ykman**: `ykman info` works? If not → permissions (udev/plugdev) or USB.
3. **PC/SC**: `pcsc_scan` sees the reader? Is `pcscd` running? Polkit allows the user?
4. **GnuPG**: `gpg --card-status` works? If ykman works but gpg doesn't, put `disable-ccid` in `~/.gnupg/scdaemon.conf`, then `gpgconf --kill scdaemon`. If it still fails, try `gpgconf --kill all`.
5. **PIN/pinentry**: only now look at PINs. Check the counter first.

## Procedures

### Fresh setup (order matters)

1. (WSL) Install `usbipd-win` (`winget install --interactive --exact dorssel.usbipd-win`). Then `usbipd bind --busid X` (admin, once) and `usbipd attach --wsl --busid X` (each time).
2. `sudo apt install yubikey-manager pcscd libccid pcsc-tools gnupg pinentry-curses`
3. Permissions: add the user to `plugdev`; add udev rule `KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="1050", MODE="0660", GROUP="plugdev"`; add a Polkit rule allowing `org.debian.pcsc-lite.access_pcsc` and `access_card` for the user.
4. `~/.gnupg/scdaemon.conf`: `disable-ccid` (needed when pcscd is in use, e.g. WSL2). `~/.gnupg/gpg-agent.conf`: `pinentry-program /usr/bin/pinentry-curses`. Shell rc: `export GPG_TTY=$(tty)`.
5. Optional and **only now**, before PINs or keys: KDF (`gpg --card-edit` → `admin` → `kdf-setup`), `ykman openpgp access set-retries`.
6. Change PINs if the user wants (`ykman openpgp access change-pin` / `change-admin-pin`, interactive).
7. Master key: `gpg --expert --full-generate-key` → ECC set-own-capabilities → **Certify only**, Curve 25519.
8. Subkeys: `gpg --expert --edit-key <KEYID>` → `addkey` ×3: sign only, encrypt only, authenticate only (Curve 25519, e.g. 2y expiry) → `save`.
9. **Backup + test restore** (see below). Also export the public key and keep `~/.gnupg/openpgp-revocs.d/<FPR>.rev`.
10. Card key types: `gpg --card-edit` → `admin` → `key-attr` → ECC/Curve 25519 for all three. Expect `ed25519 cv25519 ed25519`.
11. `gpg --edit-key <KEYID>`, then `key 1`, `keytocard` (1), `key 1` (deselect), `key 2`, `keytocard` (2), `key 2`, `key 3`, `keytocard` (3), `save`. `key N` **toggles**; exactly one `*` per `keytocard`. Asks for the passphrase + Admin PIN.
12. Verify: `gpg -K` shows 3× `ssb>`; `ykman openpgp info` shows 3 fingerprints.
13. Test (below).

### Test

```bash
echo test > /tmp/t; gpg -u '<SIG_SUBKEY_ID>!' --detach-sign -o /tmp/t.sig /tmp/t && gpg --verify /tmp/t.sig /tmp/t
echo secret | gpg -e -r <KEYID> | gpg -d          # decryption is the real test of slot 2
gpg --export-ssh-key <KEYID>                        # auth subkey as an SSH public key
ykman openpgp info | grep tries                     # counter must be unchanged
rm -f /tmp/t /tmp/t.sig
```

Non-interactive (agent shell without a TTY), only when the user provided the PIN for this purpose: add `--batch --pinentry-mode loopback --passphrase '<PIN>'`. gpg-agent forwards it to the card.

### Backup and test restore

```bash
DIR=<backup dir>; set -o pipefail
tar --exclude='S.gpg-agent*' --exclude='*.lock' -czf - -C "$HOME" .gnupg \
  | gpg --symmetric --cipher-algo AES256 -o "$DIR/gnupg-home-$(date +%F).tar.gz.gpg"

T=$(mktemp -d); chmod 700 "$T"
gpg -d "$DIR/gnupg-home-<date>.tar.gz.gpg" | tar -xzf - -C "$T"
GNUPGHOME="$T/.gnupg" gpg -K       # expect sec + ssb (no ">", these are full secrets)
echo ok | GNUPGHOME="$T/.gnupg" gpg --trust-model always -e -r <KEYID> | GNUPGHOME="$T/.gnupg" gpg -d
GNUPGHOME="$T/.gnupg" gpgconf --kill all; rm -rf "$T"
```

Test **every** subkey in a backup, not just "import succeeded". An ASCII `--export-secret-keys` can import fine and still have one subkey that won't unlock.

### Move the master key offline

Only after the backup is tested and signing + decryption via the card work:

```bash
gpg -K --with-colons | awk -F: '$1=="fpr"{print $10; exit}'   # primary FPR
gpg --delete-secret-keys '<PRIMARY_FPR>!'    # "!" = only the primary secret; stubs stay
gpg -K                                       # expect sec# + 3x ssb>
```

Without `!`, the subkey stubs are deleted too. You'd then need `gpg --card-status` to recreate them.

### Use the card on a new machine

`gpg --import public.asc` → `gpg --card-status` (creates stubs) → `gpg --edit-key <KEYID>` → `trust` → 5.

### Renew before expiry

Restore the master into a temp `GNUPGHOME` (or use the local keyring if it has `sec`):

```bash
gpg --quick-set-expire <PRIMARY_FPR> 2y                 # master
gpg --quick-set-expire <PRIMARY_FPR> 2y '*'             # all subkeys
gpg --armor --export <KEYID> > public-$(date +%F).asc   # import this on every machine; re-upload to GitHub/keyservers
```

The keys on the card do not change; only the public key's expiry does.

### SSH via gpg-agent (OpenPGP auth subkey)

`gpg-agent.conf`: `enable-ssh-support`. Shell rc:

```bash
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent; gpg-connect-agent updatestartuptty /bye >/dev/null
```

`ssh-add -L` should show the card key. This replaces the default ssh-agent; warn the user.

### SSH via FIDO2

`ssh-keygen -t ed25519-sk -f ~/.ssh/id_ed25519_sk` (add `-O resident` to store it on the key). Not covered by GPG backups; recommend a second key.

### Git signing

```bash
git config --global user.signingkey '<SIG_SUBKEY_ID>!'
git config --global commit.gpgSign true
```

The commit email must match a UID. Upload the public key to GitHub for "Verified".

### Touch policy

`ykman openpgp keys set-touch {sig|dec|aut} {off|on|cached|fixed|cached-fixed}` (Admin PIN). `cached` = 15 s. `fixed` variants **can't be undone** without replacing the key. Confirm before using them.

### Google / WebAuthn 2FA

Browser: myaccount.google.com → Security → 2-Step Verification → Security keys → Add. On WSL setups, `usbipd detach` first so Windows sees the key.

## Common errors

| Error | Fix |
|---|---|
| `You must select exactly one key` | `key N` toggles; leave exactly one `*` |
| `Bad PIN`, counter unchanged | pinentry/agent: `GPG_TTY`, `updatestartuptty`, kill gpg-agent |
| `Bad PIN`, counter dropped | Wrong PIN. Is it the User PIN (not the Admin PIN)? Re-check the counter |
| `card not available` / `No such device` | USB layer (usbipd attach), then `gpgconf --kill scdaemon` |
| ykman OK, gpg fails | `disable-ccid` in scdaemon.conf; is pcscd running? |
| `Bad passphrase` restoring one subkey | That export is bad for that subkey. Use another backup (full `~/.gnupg` tarball) |
| Browser can't see the key on Windows | It's attached to WSL. `usbipd detach --busid X` |

## References

- ykman OpenPGP: https://docs.yubico.com/software/yubikey/tools/ykman/OpenPGP_Commands.html
- Yubico PGP: https://developers.yubico.com/PGP/
- WSL USB: https://learn.microsoft.com/en-us/windows/wsl/connect-usb
- drduh/YubiKey-Guide: https://github.com/drduh/YubiKey-Guide
- `man gpg`, `man gpg-agent`
