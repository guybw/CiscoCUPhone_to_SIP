# Cisco IP Phone 8841 → Generic SIP (e.g. sipgate), no MPP licence, no PBX

I wrote this guide to help myself as I was nearly going to buy a license to enable SIP calling when it was already able to be done, I hope this helps someone!

Register a **Cisco IP Phone 8841 (or 8811 / 8851 / 8861)** running the standard
**Enterprise "SIP" firmware** directly to a generic SIP provider such as
**sipgate**, using nothing more than a small TFTP server and a config file.

No Multiplatform (MPP) conversion licence. No 3CX. No Asterisk / FreePBX in the
middle. The phone talks SIP straight to the provider.

> Tested on an 8841 that had been wiped from a previous Cisco phone system,
> running Enterprise SIP firmware, registering to a free sipgate account, on a
> UK home LAN. The same approach is widely reported to work on the 7811 / 7821 /
> 7841 / 7861 and the rest of the 88xx audio range.

---

## 1. Background — why this is fiddly

Cisco 78xx/88xx phones come in two firmware worlds:

| Firmware | Load name | Talks to | Licence to convert? |
|---|---|---|---|
| **Enterprise** ("SIP") | `sip88xx.*` | Cisco CUCM **or** a generic SIP server via a config file | n/a (already on it) |
| **Multiplatform (MPP)** | `cp-88xx.*` | Webex Calling / generic 3PCC, configured via web UI | **Yes** — `L-CP-E2M-88XX-CNV`, ~£40/device |

A lot of guides tell you that you *must* buy the MPP migration licence, or that
Enterprise firmware "needs CUCM". Neither is true for basic SIP registration: the
Enterprise **SIP** firmware can register to any SIP registrar if you feed it a
`SEP<MAC>.cnf.xml` config over TFTP. (The **SCCP** firmware variant is the one
that genuinely needs CUCM — don't use that.)

This bundle is the Enterprise-SIP-over-TFTP route. It costs nothing.

---

## 2. What you need

- A Cisco 8841 (or compatible 78xx/88xx) on Enterprise **SIP** firmware.
  Check **Settings → Phone Information**: the load should start `sip88xx.` If it
  shows `cp-88xx` you're on MPP — different process, not covered here.
- SIP account credentials from your provider:
  - **SIP User ID** (sipgate calls this the SIP-ID, e.g. `1234567e0`)
  - **SIP Password**
  - **Registrar / server hostname** (sipgate basic: `sipgate.co.uk`)
  - Your **phone number** (for the on-screen label)
- An always-on TFTP server reachable from the phone (a Raspberry Pi is ideal).
- The ability to set **DHCP option 66** (TFTP server) on your router/DHCP server.
- The files in this bundle.

> ⚠️ **sipgate gotcha:** sipgate has two products. Basic per-device SIP
> credentials register to **`sipgate.co.uk`**. The *trunking* product uses
> **`sipconnect.sipgate.co.uk`** — a different hostname. Use whatever your own
> account's device page shows under "Registrar". Using the wrong one causes an
> endless "registering" loop.

---

## 3. Files in this bundle

| File | Purpose |
|---|---|
| `SEP_TEMPLATE.cnf.xml` | The phone config. Rename + fill in placeholders. |
| `dialplan.xml` | UK dial plan so numbers dial instantly instead of waiting. |
| `AppDialRules.xml` | Empty rules file; stops a harmless boot-time 404. |
| `README.md` | This guide. |

**Firmware files are NOT included** (Cisco copyright / licensing). If you want to
upgrade firmware as well, see section 8 — you supply the firmware zip yourself
from your Cisco account.

---

## 4. Set up the TFTP server (Raspberry Pi / Debian example)

```bash
sudo apt install tftpd-hpa -y
sudo systemctl enable tftpd-hpa --now
```

Make sure it listens on all interfaces. Edit `/etc/default/tftpd-hpa`:

```
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--secure --verbose"
```

(`--verbose` logs every file request — very handy while setting up. Drop it
later if you want quieter logs.)

```bash
sudo systemctl restart tftpd-hpa
```

The TFTP root is `/srv/tftp`. Files written there need root (use `sudo cp`).

---

## 5. Prepare the config file

1. Find your phone's MAC (label on the base, or **Settings → Phone Information**).
2. Rename the template to `SEP<MAC>.cnf.xml` — **uppercase, no colons**.
   - MAC `00:a3:d1:f3:de:d5` → `SEP00A3D1F3DED5.cnf.xml`
3. Open it and replace every `<<PLACEHOLDER>>`:

   | Placeholder | Replace with | Example |
   |---|---|---|
   | `<<SIP_REGISTRAR>>` | Provider registrar hostname | `sipgate.co.uk` |
   | `<<SIP_USER_ID>>` | SIP user / auth ID | `123456789` |
   | `<<SIP_PASSWORD>>` | SIP password | `s3cr3t` |
   | `<<DISPLAY_NAME>>` | Name shown on screen | `Jane Smith` |
   | `<<LINE_LABEL>>` | Label by the line button | `01234 567890` |

   `<<SIP_REGISTRAR>>` appears in three places and `<<SIP_USER_ID>>` in several —
   replace them all (a global find-and-replace is easiest).

4. Copy the three files to the TFTP root:

```bash
sudo cp SEP<MAC>.cnf.xml dialplan.xml AppDialRules.xml /srv/tftp/
```

---

## 6. Point the phone at the TFTP server (DHCP option 66)

On your router / DHCP server, set **option 66** (TFTP server name) to the IP of
your TFTP box, e.g. `192.168.1.50`. The phone reads this on boot.

(If your DHCP server offers option 150 instead, that also works — it's Cisco's
TFTP-server-address option and takes an IP.)

---

## 7. Boot and verify

Watch the TFTP log, then power-cycle the phone:

```bash
journalctl -u tftpd-hpa -f
```

You should see the phone request, in roughly this order:
`CTLSEP<MAC>.tlv`, `ITLSEP<MAC>.tlv`, `ITLFile.tlv`, **`SEP<MAC>.cnf.xml`**,
some locale files, `dialplan.xml`, `AppDialRules.xml`.

Then check:

- **Phone screen** shows your line/number and is *not* stuck on "Registering".
- **Provider portal** shows the device as **registered / online**.
- **Make a test call.**

If it all works, you're done.

> **Tip:** If you ever change something and the phone seems to ignore it, it may
> be using a cached config. Force a clean re-read with a factory reset: unplug
> the network, hold **#** while plugging back in, then when the buttons flash
> enter **123456789\*0#**.

---

## 8. (Optional) Upgrade the firmware at the same time

This step needs the firmware **zip from your own Cisco account** — it is not
included here.

1. Download the Enterprise SIP firmware zip for the 88xx, e.g.
   `cmterm-88xx.<version>.zip`. Inside it is a `sip88xx.<version>.loads` file
   plus a set of `.sbn` component images.
2. Unzip **all** of it directly into the TFTP root:
   ```bash
   sudo unzip cmterm-88xx.<version>.zip -d /srv/tftp/
   ls /srv/tftp/   # confirm .loads + .sbn files are in the root, not a subfolder
   ```
3. In your `SEP<MAC>.cnf.xml`, **uncomment / add** the load line so it matches the
   `.loads` filename **without** the extension:
   ```xml
   <loadInformation>sip88xx.<version></loadInformation>
   ```
4. Re-copy the config, then reboot the phone. In the TFTP log you'll now see it
   request `sip88xx.<version>.loads` followed by the `.sbn` images — that's the
   flash downloading.

> ⚠️ **Critical:** the load name in `<loadInformation>` must EXACTLY match a
> `.loads` file present on the TFTP server. If it names a file that isn't there,
> the phone boot-loops trying to download firmware it can't find. If you're not
> upgrading, leave the load line out entirely and the phone keeps its current
> firmware.

> ⚠️ **Do not interrupt a firmware flash.** It reboots several times and takes
> ~10–15 minutes over wired ethernet. Pulling power mid-write can brick the phone.

### Firmware & security note
At time of writing, the latest 88xx Enterprise SIP fix for the Oct-2025 web-UI
advisory was **14.3(1)SR2**. Most of the published 88xx vulnerabilities are in
the **web management interface** and require it to be enabled. This config ships
with `<webAccess>0</webAccess>` (disabled), which removes that attack surface.
Always check Cisco's current security advisories for your model and pick the
newest firmware your account offers.

---

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Phone never requests `SEP<MAC>.cnf.xml` | DHCP option 66 not set / wrong IP; phone got its lease before option 66 existed | Set option 66, then factory-reset the phone |
| TFTP log shows `read(ack): Connection refused` | tftpd listening on localhost only | Set `TFTP_ADDRESS="0.0.0.0:69"`, restart |
| `cp: Permission denied` writing to `/srv/tftp` | Not root | Use `sudo cp` |
| Endless boot loop requesting a `.loads` file | `<loadInformation>` names firmware not present on TFTP | Remove the load line, or add the matching firmware files |
| Stuck on "Registering" forever | Wrong registrar hostname or wrong SIP credentials | Verify against your provider's device page (e.g. `sipgate.co.uk` vs `sipconnect.sipgate.co.uk`) |
| Config edits seem ignored | Phone using cached config | Factory reset to force a clean re-read |
| Numbers take ~10s to dial | No dial plan loaded | Ensure `dialplan.xml` is present and referenced in the config |

### Harmless log entries you can ignore
The phone will keep requesting a few files that you don't provide; these all fall
back to safe built-in defaults and don't affect operation:
- `CTLSEP<MAC>.tlv`, `ITLSEP<MAC>.tlv`, `ITLFile.tlv` — security trust lists, only
  used in a secured CUCM deployment.
- `English_United_Kingdom/sl-be-sip.jar`, `United_Kingdom/g3-tones.xml` —
  locale / ringtone overrides that live inside the firmware. Built-in defaults
  apply if absent.
- `defaultheadsetconfig.json` — headset config; harmless when missing.

---

## 10. Security checklist

- Keep `<webAccess>0</webAccess>` unless you actively need the web UI.
- Put the phone on a trusted LAN; don't expose SIP/TFTP to the internet.
- The TFTP server must stay reachable: the phone re-reads config on every boot.

---

## Compatibility

Reported working on Enterprise **SIP** firmware for: 7811, 7821, 7841, 7861,
8811, **8841**, 8851, 8861. Adjust the dial plan digit lengths for your country.
Not for the **SCCP** firmware variant (that one requires CUCM).

## Disclaimer
Provided as-is, no warranty. "Cisco" and model names are trademarks of Cisco
Systems, Inc.; this is an independent community guide and is not affiliated with
or endorsed by Cisco. Firmware files are not redistributed here — obtain them
from your own Cisco account under your own licence.
