# YubiKey: GPG + Google 2FA on Windows + WSL2

How the YubiKey was set up on 2026-11-06 (YubiKey on Windows, used from Ubuntu in WSL2), and how to redo it from scratch.

Personal identifiers are partly masked (`****`). In commands, `<KEYID>` is the full master key ID (see `gpg -K`) and `<user>` is the Windows/Linux username.

This guide is meant for me, for other people setting up their own YubiKey, and for AI agents helping with it. The steps are generic; the [Status](#status), [Overview](#overview) and backup results describe one real setup as an example.

---

## Rules for agents (and humans)

1. **Audit before changing anything.** Run `ykman openpgp info`, `gpg --card-status` and `gpg -K` first, and read the current config files.
2. **Check the PIN counter before every PIN attempt.** Never guess PINs in a loop. If tries are at 1, stop and ask.
3. **Never reset** the OpenPGP app or the whole YubiKey, and never run `gpg --delete-secret-keys` or delete files in `private-keys-v1.d`, without an explicit OK from the owner **and** a tested backup.
4. **Never overwrite a known-good backup.** New backups get a new filename and must be restored and tested before they count.
5. **Back up before `keytocard`.** `save` after `keytocard` deletes the local private subkeys for good.
6. **Don't write PINs or passphrases** into docs, scripts, logs or shell history. Use interactive prompts. If a secret must go on a command line, use it only for that command.
7. **Debug in layers**: USB → ykman → PC/SC → gpg → PIN. A `Bad PIN` with an unchanged counter is a pinentry problem, not a wrong PIN.
8. **One change at a time.** Explain what it does and confirm before anything irreversible.
9. **Test on a copy.** Restore backups into a temp `GNUPGHOME`, never into `~/.gnupg`.

---

## Status

- [x] YubiKey passed through from Windows to WSL2 with `usbipd-win`
- [x] Linux access without `sudo` (group `plugdev`, udev rule, Polkit rule)
- [x] PC/SC + `scdaemon` working together (`disable-ccid`)
- [x] Card key types set to ed25519 / cv25519 / ed25519
- [x] All 3 subkeys moved to the card
- [x] Full backup created **before** `keytocard`, restored and tested
- [x] User PIN confirmed (factory default)
- [x] Signing test with the YubiKey: `Good signature`
- [x] Encrypt + decrypt test with the YubiKey: OK
- [x] OpenPGP SSH public key exported from the auth subkey
- [x] Public key fingerprint and revocation certificate checked
- [x] Google 2FA: YubiKey added as a security key (FIDO2)
- [x] FIDO2 SSH key (`ed25519-sk`) generated
- [ ] FIDO2 SSH key added to GitHub/servers (not confirmed)
- [ ] Master key moved offline (still in the local keyring, see [Pending: offline master key](#pending-offline-master-key))

---

## Overview

| Item | Value |
|---|---|
| Device | YubiKey 5C NFC, firmware 5.8.0, USB `1050:0407`, serial `****4273` |
| Host | Windows + WSL2 Ubuntu 26.04 (kernel `6.18.x-microsoft-standard-WSL2`) |
| Tools | GnuPG 2.4.8, ykman 5.8.0, OpenPGP app 3.4 |
| Identity | `H**** <h****@gmail.com>` |
| Master key (Certify) | `ed25519/****3DEF1928`, **not** on the card |
| Signing subkey (S) | `ed25519/****0B6ED501`, card slot 1 |
| Encryption subkey (E) | `cv25519/****4F5A29AE`, card slot 2 |
| Authentication subkey (A) | `ed25519/****CE79C71E`, card slot 3 |
| Expires | 2028-11-05 |

```
Master key [C]  ── offline (backup only, ideally) ── certifies / renews / revokes
 ├── [S] sign     ─┐
 ├── [E] encrypt   ├── on the YubiKey (OpenPGP app, CCID)
 └── [A] auth     ─┘
FIDO2 app (separate) ── Google 2FA + ssh ed25519-sk
```

YubiKey applications enabled: OTP, FIDO U2F, FIDO2, OATH, PIV, OpenPGP, YubiHSM Auth. Only OpenPGP and FIDO2 are in use. **Never** run a full YubiKey reset to fix an OpenPGP problem, because it wipes the other applications too.

---

## PINs and passwords

| Credential | Used for | Value |
|---|---|---|
| User PIN | signing, decrypting, SSH via GPG | factory default `123456` |
| Admin PIN | card configuration, `keytocard`, unblocking the User PIN | factory default `12345678` |
| Reset Code | alternative way to unblock the User PIN | **not set** (0 tries) |
| GPG key passphrase | protects the master key/subkeys on disk and in backups | (yours) |
| Backup passphrase | symmetric encryption of `gnupg-home-*.tar.gz.gpg` | (yours) |
| FIDO2 PIN | Google and `ed25519-sk` SSH | separate from OpenPGP |

Each OpenPGP PIN allows 3 tries. **Always check the counter before guessing:**

```bash
ykman openpgp info          # "PIN tries remaining" / "Admin PIN tries remaining"
```

If the User PIN gets blocked (0 tries), unblock it with the **Admin PIN**:

```bash
ykman openpgp access unblock-pin
```

If the Admin PIN gets blocked too, the only way out is an OpenPGP reset (`ykman openpgp reset`). It **erases all OpenPGP keys on the card** and restores the default PINs. You'd then restore the keys from the backup.

### Changing PINs (optional)

```bash
ykman openpgp access change-pin          # User PIN, min 6 chars
ykman openpgp access change-admin-pin    # Admin PIN, min 8 chars
ykman openpgp access change-reset-code   # optional Reset Code, min 6 chars
```

Leave out `-P`/`-n` so the PINs are prompted for and stay out of shell history.

**Gotcha (OpenPGP card spec):** the card receives the current PIN and the new PIN glued together and splits them by the current PIN's length. If you type extra characters in the "current PIN" field (e.g. `1234567` instead of `123456`), the change still **succeeds**, but the extras get prepended to the new PIN (`7` + new PIN). Type the current PIN carefully.

Other PIN settings (need the Admin PIN):

```bash
ykman openpgp access set-retries 5 5 5         # User / Reset Code / Admin tries (default 3 3 3)
ykman openpgp access set-signature-policy once  # "once" per session (default) or "always"
```

On some older YubiKey models, `set-retries` also resets the PINs to their defaults. To be safe, run it **before** changing PINs, and check `ykman openpgp info` afterwards.

---

## Backups

All backups live **only** in `C:\Users\<user>\Desktop\Backup\keys` (WSL: `/mnt/c/Users/<user>/Desktop/Backup/keys`), with an extra copy on offline media.

| File | Contents | Checked 2026-11-06 |
|---|---|---|
| `gnupg-home-2026-11-06.tar.gz.gpg` | AES-256 encrypted copy of the whole `~/.gnupg`, taken **before** `keytocard` | ✅ **Main recovery backup.** Decrypted, extracted into a temp dir; master + 3 subkeys present; decryption with the restored E subkey worked |
| `master-secret.asc` | ASCII export of master + 3 subkeys | ⚠️ Imports fine and the S subkey works, but the **E subkey fails with `Bad passphrase`**. Don't rely on it for the encryption subkey; use the tarball |
| `master-public.asc` | public key | ✅ fingerprint `93AE CAA1 … 3DEF 1928` matches |
| `revocation.asc` | key revocation certificate (copy of `~/.gnupg/openpgp-revocs.d/<FPR>.rev`) | ✅ valid revocation (sigclass `0x20`) for the master key |

`revocation.asc` has a `:` in front of `-----BEGIN PGP PUBLIC KEY BLOCK-----` on purpose, so it can't be imported by accident. To actually revoke the key, remove that `:` and run `gpg --import revocation.asc`.

### How the main backup was made

Do this **before** `keytocard`. After `keytocard` + `save`, the private subkeys only exist on the card.

```bash
DIR="/mnt/c/Users/<user>/Desktop/Backup/keys"
set -o pipefail
tar --exclude='S.gpg-agent*' --exclude='*.lock' -czf - -C "$HOME" .gnupg \
  | gpg --symmetric --cipher-algo AES256 --output "$DIR/gnupg-home-$(date +%F).tar.gz.gpg"
```

### How to test a backup (without touching your real keyring)

```bash
T=$(mktemp -d); chmod 700 "$T"
gpg -d "$DIR/gnupg-home-2026-11-06.tar.gz.gpg" | tar -xzf - -C "$T"
export GNUPGHOME="$T/.gnupg"; chmod 700 "$GNUPGHOME"
gpg -K                                    # sec + 3 ssb (no ">", these are full keys)
echo ok | gpg --trust-model always -e -r <KEYID> | gpg -d
gpgconf --kill all; unset GNUPGHOME; rm -rf "$T"
```

---

## Setup step by step

### 1. Windows → WSL2: USB passthrough (usbipd-win)

Requirements: WSL 2 with kernel ≥ 5.10.60.1 (`wsl --update`) and usbipd-win ≥ 5.0. Install it with:

```powershell
winget install --interactive --exact dorssel.usbipd-win
```

Keep a WSL terminal open while attaching; it keeps the WSL VM alive.

In PowerShell on Windows:

```powershell
usbipd list                          # find the YubiKey BUSID (it was 2-2, but it can change)
usbipd bind --busid <BUSID>          # first time only; needs an ADMIN PowerShell
usbipd attach --wsl --busid <BUSID>  # every time you plug in the key or restart WSL (no admin needed)
```

While attached to WSL, **Windows can't use the YubiKey** (e.g. for Google login in the Windows browser). Detach it to give it back:

```powershell
usbipd detach --busid <BUSID>
```

In Ubuntu, check that it shows up:

```bash
lsusb | grep -i yubico     # 1050:0407 Yubico.com Yubikey 4/5 OTP+U2F+CCID
```

The attach is manual. Nothing automates it after a reboot.

### 2. Packages (Ubuntu)

```bash
sudo apt update
sudo apt install yubikey-manager pcscd libccid pcsc-tools gnupg pinentry-curses
```

### 3. Permissions (use the key without sudo)

```bash
sudo usermod -aG plugdev <user>      # then restart the WSL session
```

`/etc/udev/rules.d/70-yubikey.rules`:

```udev
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="1050", MODE="0660", GROUP="plugdev"
```

`/etc/polkit-1/rules.d/90-pcsc-<user>.rules`:

```javascript
polkit.addRule(function(action, subject) {
    if (subject.user == "<user>" &&
        (action.id == "org.debian.pcsc-lite.access_pcsc" ||
         action.id == "org.debian.pcsc-lite.access_card")) {
        return polkit.Result.YES;
    }
});
```

### 4. PC/SC and scdaemon

Make GnuPG's `scdaemon` go through `pcscd` instead of its own CCID driver. Without this they fight over the card on WSL2:

```bash
printf 'disable-ccid\n' > ~/.gnupg/scdaemon.conf
gpgconf --kill scdaemon; gpgconf --kill gpg-agent
pcsc_scan          # should show the Yubico reader (Ctrl+C to exit)
gpg --card-status  # should show the OpenPGP card
```

### 5. gpg-agent and pinentry

`~/.gnupg/gpg-agent.conf`:

```conf
pinentry-program /usr/bin/pinentry-curses
```

`~/.zshrc` (already added):

```bash
export GPG_TTY=$(tty)
```

If the PIN prompt still doesn't show up in the current terminal:

```bash
gpg-connect-agent updatestartuptty /bye
```

### 6. Generate the master key (Certify only)

```bash
gpg --expert --full-generate-key
# (11) ECC (set your own capabilities) -> keep only Certify
# (1) Curve 25519
# expiration: 2y
```

### 7. Add the 3 subkeys

```bash
gpg --expert --edit-key <KEYID>
gpg> addkey   # (10) ECC (sign only)        -> Curve 25519 -> 2y
gpg> addkey   # (12) ECC (encrypt only)     -> Curve 25519 -> 2y
gpg> addkey   # (11) ECC (set capabilities) -> Authenticate only -> Curve 25519 -> 2y
gpg> save
```

### 8. Back up (see [Backups](#backups)), then export the public key

```bash
gpg --armor --export <KEYID> > master-public.asc
cp ~/.gnupg/openpgp-revocs.d/<FINGERPRINT>.rev revocation.asc
```

### 9. Set the card key types

The card ships with `rsa2048 rsa2048 rsa2048`. Change it to Curve 25519:

```bash
gpg --card-edit
gpg/card> admin
gpg/card> key-attr   # Signature, Encryption, Authentication: (2) ECC -> (1) Curve 25519
gpg/card> quit
gpg --card-status    # Key attributes ...: ed25519 cv25519 ed25519
```

### 10. Move the subkeys to the card (one at a time!)

`key N` **toggles** the selection. GPG needs **exactly one** subkey marked with `*` for each `keytocard`, otherwise you get `You must select exactly one key`.

```bash
gpg --edit-key <KEYID>

gpg> key 1          # select S
gpg> keytocard      # (1) Signature key
gpg> key 1          # deselect
gpg> key 2          # select E
gpg> keytocard      # (2) Encryption key
gpg> key 2          # deselect
gpg> key 3          # select A
gpg> keytocard      # (3) Authentication key
gpg> save
```

- It asks for the key passphrase and the **Admin PIN**.
- Nothing is committed until `save`, which also deletes the local private subkeys.
- Saving after each subkey and reopening `--edit-key` also works (that's how it was done here).
- gpg only offers the slot that matches the subkey's usage.
- Once a subkey is on the card, it shows `card-no: 0006 ****4273` under it.

### 11. Check

```bash
gpg --card-status
gpg -K               # subkeys show as "ssb>": the ">" means "stub, the real key is on the card"
ykman openpgp info   # 3 slots with fingerprints, touch policy, PIN counters
```

### 12. Test

```bash
# signing (asks for the User PIN)
echo "test" > /tmp/t.txt
gpg --local-user '<SIGNING_SUBKEY_ID>!' --detach-sign --armor -o /tmp/t.asc /tmp/t.txt
gpg --verify /tmp/t.asc /tmp/t.txt          # "Good signature"

# encrypt + decrypt (decrypting is the real test of card slot 2)
echo "secret" | gpg -e -r <KEYID> -o /tmp/t.gpg
gpg -d /tmp/t.gpg                            # "secret"

# OpenPGP SSH public key (auth subkey)
gpg --export-ssh-key <KEYID>                 # ssh-ed25519 AAAA... openpgp:0x****C71E
```

---

## Google 2FA (FIDO2)

This uses the FIDO2 app, which is independent of GPG. On Windows, **detach the key from WSL first** (`usbipd detach`) so the browser can see it.

1. Open <https://myaccount.google.com/signinoptions/two-step-verification>
2. Turn on 2-Step Verification
3. **Security keys** → **Add security key**
4. Touch the YubiKey when prompted and give it a name

## SSH with FIDO2 (`ed25519-sk`)

This is a separate SSH key backed by the FIDO2 app. It's **not** the OpenPGP auth subkey, and it's **not** in the GPG backup.

```bash
ssh-keygen -t ed25519-sk -f ~/.ssh/id_ed25519_sk -C "yubikey-5c"
cat ~/.ssh/id_ed25519_sk.pub     # add this to GitHub / servers
ssh -T git@github.com            # touch the key when asked
```

`~/.ssh/id_ed25519_sk` is only a handle; it's useless without the physical key. If you lose the YubiKey, you lose this SSH key: register a second key or keep another login method.

---

## SSH with the OpenPGP auth subkey (gpg-agent)

This is an alternative to the FIDO2 SSH key: gpg-agent acts as the SSH agent and uses card slot 3. Not set up yet.

`~/.gnupg/gpg-agent.conf`, add:

```conf
enable-ssh-support
```

`~/.zshrc`, add:

```bash
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
gpg-connect-agent updatestartuptty /bye > /dev/null
```

```bash
gpgconf --kill gpg-agent      # reload
ssh-add -L                    # should list "ssh-ed25519 AAAA... cardno:..." (the card's auth key)
gpg --export-ssh-key <KEYID>  # same public key, for authorized_keys / GitHub
```

Notes:
- `SSH_AUTH_SOCK` replaces your normal ssh-agent, so keys in `~/.ssh` need `ssh-add` again or an `IdentityFile` in `~/.ssh/config`.
- Don't set it on remote hosts you reach with `ssh -A`.

## Git commit signing

```bash
git config --global user.signingkey '<SIGNING_SUBKEY_ID>!'   # "!" = use exactly this subkey
git config --global commit.gpgSign true
git config --global tag.gpgSign true
git commit -S -m "msg"            # -S (capital) signs; -s only adds Signed-off-by
git log --show-signature -1       # verify
```

Add the public key (`gpg --armor --export <KEYID>`) to GitHub → Settings → SSH and GPG keys so commits show as **Verified**. The key's email must match the commit email.

## Card settings

| Setting | Value | Meaning |
|---|---|---|
| Key attributes | `ed25519 cv25519 ed25519` | Sign / Encrypt / Auth |
| Signature PIN | not forced (`Require PIN for signature: Once`) | the User PIN is asked once per session |
| PIN retry counter | `3 0 3` | User PIN / Reset Code (not set) / Admin PIN |
| KDF | off | |
| Touch policy | off for all 3 slots | no touch needed to use the keys |
| Cardholder name, URL, login data | not set | optional |

### Touch policy (optional, not applied)

Requiring a physical touch stops malware from silently using the card while it's plugged in and unlocked.

```bash
ykman openpgp keys set-touch sig on     # touch to sign
ykman openpgp keys set-touch dec on     # touch to decrypt (older ykman versions call it "enc")
ykman openpgp keys set-touch aut on     # touch for SSH
ykman openpgp info                      # check
```

| Policy | Meaning |
|---|---|
| `off` | no touch (default) |
| `on` | touch every time |
| `cached` | touch, then free for 15 s (good for mail clients that decrypt a lot) |
| `fixed` | like `on`, but **can't be turned off** without deleting the key |
| `cached-fixed` | like `cached`, but **can't be turned off** |

Avoid `fixed`/`cached-fixed` unless you mean it.

### KDF (optional, not applied)

KDF makes the card store and compare a hash of the PIN instead of the PIN itself.
- Enable it **before** changing PINs or moving keys to the card (drduh guide). On a card that's already set up, it's safer to leave it off.
- Old GnuPG versions and some mobile clients (e.g. older OpenKeychain) don't support it and will always reject the PIN.

```bash
gpg --card-edit     # admin -> kdf-setup
```

### Other

```bash
gpg --card-edit     # admin -> name / url / login
```

Don't run `key-attr` again now that the slots have keys in them. Changing a slot's key type makes the key in it unusable.

---

## Using it on another computer

```bash
gpg --import master-public.asc
gpg --card-status        # creates the "ssb>" stubs pointing to the card
gpg --edit-key <KEYID>
gpg> trust               # 5 = ultimate
gpg> quit
```

## Renewing before 2028-11-05

Restore the master key from the backup (see [How to test a backup](#how-to-test-a-backup-without-touching-your-real-keyring)) or use the local keyring while it still has it. Then:

```bash
gpg --edit-key <KEYID>
gpg> expire              # master key
gpg> key 1               # then key 2, key 3: expire each subkey
gpg> expire
gpg> save
gpg --armor --export <KEYID> > master-public.asc   # re-publish the public key
```

The subkeys don't need to go back onto the card. Only their expiration date (stored in the public key) changes.

---

## Pending: offline master key

The master key secret is **still in `~/.gnupg`** (`gpg -K` shows `sec`, not `sec#`). The goal is to keep only the card stubs locally, with the master key in the backup only. **Not done yet.**

GnuPG supports this directly: with a `!` after the **primary key fingerprint**, `--delete-secret-keys` deletes **only the primary secret** and leaves the subkey stubs alone (see `man gpg`).

```bash
# 0. make a NEW backup first (different filename, don't overwrite the old one) and test it
gpg -K --with-colons | awk -F: '$1=="fpr"{print $10; exit}'   # primary fingerprint
gpg --delete-secret-keys '<PRIMARY_FINGERPRINT>!'             # the "!" is essential
gpg -K       # should now show "sec#" (master offline) + 3x "ssb>"
```

Then sign and decrypt again to confirm everything still works.

**Without the `!`**, `--delete-secret-keys` removes the master **and** the subkey stubs. You'd then have to run `gpg --card-status` to recreate the stubs.

To use the master key later (renew, certify), restore the backup into a temporary `GNUPGHOME`, do the work there, then export the updated public key and import it into your daily keyring.

---

## Quick diagnostics

Work down the layers. Don't touch PINs or reset anything if the problem is just USB.

```powershell
# Windows
usbipd list               # is the key attached to WSL?
```

```bash
# WSL
lsusb | grep -i yubico    # 1. hardware visible?
ykman info                # 2. ykman sees it?
pcsc_scan                 # 3. PC/SC sees the reader? (Ctrl+C)
gpg --card-status         # 4. GnuPG reads the card?
ykman openpgp info        # 5. PIN counters
gpgconf --kill scdaemon; gpgconf --kill gpg-agent   # reset gpg's card state
```

## Troubleshooting

| Problem | Cause / fix |
|---|---|
| `You must select exactly one key` | More than one subkey has a `*`. `key N` toggles. Deselect the extra one. |
| `signing failed: Bad PIN`, counter still 3 | The PIN never reached the card: pinentry couldn't open. Check `GPG_TTY`, run `gpg-connect-agent updatestartuptty /bye`, `gpgconf --kill gpg-agent`. This happened during setup; the User PIN was the factory default all along. |
| `Bad PIN` and the counter went down | Wrong PIN. Make sure it's the **User PIN**, not the Admin PIN. |
| Card not found | Usually USB. `usbipd attach` again on Windows, then `gpgconf --kill scdaemon`. |
| Card works in `ykman` but not in `gpg` | Check `~/.gnupg/scdaemon.conf` has `disable-ccid` and that `pcscd` is running. |
| Google doesn't detect the key | It's attached to WSL. `usbipd detach --busid <BUSID>`. |
| `Bad passphrase` on the E subkey from `master-secret.asc` | Known issue with that export. Restore from `gnupg-home-*.tar.gz.gpg` instead. |

## Files touched

| Path | Purpose |
|---|---|
| `/etc/udev/rules.d/70-yubikey.rules` | hidraw access for `plugdev` |
| `/etc/polkit-1/rules.d/90-pcsc-<user>.rules` | PC/SC access without sudo |
| `~/.gnupg/scdaemon.conf` | `disable-ccid` |
| `~/.gnupg/gpg-agent.conf` | `pinentry-curses` |
| `~/.gnupg/openpgp-revocs.d/<FPR>.rev` | auto-generated revocation certificate |
| `~/.zshrc` | `export GPG_TTY=$(tty)` |
| `~/.ssh/id_ed25519_sk{,.pub}` | FIDO2 SSH key |

## Risks

| Risk | Mitigation |
|---|---|
| Lose the YubiKey | Restore the subkeys from the tarball onto a new key, or revoke and create new subkeys with the master key |
| Backup leaks | It holds full private keys. Keep it AES-256 encrypted, with the passphrase stored apart from it |
| Block both PINs | Check `ykman openpgp info` before any attempt. The only fix after that is an OpenPGP reset + restore |
| Master key stolen from this PC | Finish [moving it offline](#pending-offline-master-key) |
| Lose the FIDO2 SSH key | Register a second key or keep another way to log in |

## References

- Yubico, ykman OpenPGP commands: <https://docs.yubico.com/software/yubikey/tools/ykman/OpenPGP_Commands.html>
- Yubico, PGP overview: <https://developers.yubico.com/PGP/>
- Yubico, PIN change behavior: <https://developers.yubico.com/PGP/PGP_PIN_Change_Behavior.html>
- Yubico, Git signing: <https://developers.yubico.com/PGP/Git_signing.html>
- Microsoft, connect USB devices to WSL: <https://learn.microsoft.com/en-us/windows/wsl/connect-usb>
- drduh, YubiKey-Guide (community best practices): <https://github.com/drduh/YubiKey-Guide>
- `man gpg` (`--delete-secret-keys`, `!` syntax), `man gpg-agent` (`enable-ssh-support`)
