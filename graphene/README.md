# Hardening GrapheneOS on Pixel: From Zero to Forensic Resistance

Your phone is the weakest link in your security chain. Period. You carry it everywhere, it connects to every network, and it knows more about you than your laptop ever will. So I wiped mine and rebuilt it from scratch.

Here is exactly how I hardened a Pixel 7a running GrapheneOS into a compartmentalized, forensic-resistant mobile setup. Step by step. No hand-waving.

---

## Why GrapheneOS

Leaked Cellebrite documents from early 2025 confirmed that neither Cellebrite Premium, GrayKey, nor MSAB XRY can extract data from an updated GrapheneOS device. Not even in AFU (After First Unlock) state. That is not marketing. That is leaked internal documentation from the companies that sell extraction tools to law enforcement.

The Pixel gives you a Titan M2 security chip with hardware-enforced brute-force throttling. After ~140 failed attempts, it throttles to 24 hours per guess. Your diceware passphrase becomes mathematically untouchable.

---

## The Wipe

Factory reset. `Settings > System > Reset options > Erase all data.`

This is not a reflash. GrapheneOS stays installed, bootloader stays relocked, verified boot stays intact. You are just nuking the user data partition and starting clean.

No restoring from backups. No importing old data. Clean. Slate.

Before you wipe: export your 2FA tokens, confirm your PINs, and verify your password database is synced somewhere that is not the phone. If you lose your 2FA tokens, you are locked out of everything they protect.

---

## First Boot Hardening

### Lock Screen: Password, Not PIN

The very first thing the setup wizard asks you. Choose Password. Generate a 4-word diceware passphrase: 

```
shuf -n4 /usr/share/dict/words | tr '[:upper:]' '[:lower:]'
```

Or use the EFF wordlist. Four random words. All lowercase. Something you can type on a phone keyboard after a week of muscle memory.

Why not a PIN? Defense in depth. The Titan M2 already throttles brute force, but a strong passphrase protects against theoretical future hardware vulnerabilities.

### No Biometrics

Skip fingerprint enrollment entirely. In many jurisdictions, courts have ruled that biometrics can be compelled -- police can force your finger onto the sensor. Passwords are protected as testimony. No fingerprint enrolled means that attack vector does not exist.

### Security Preview Releases

The setup wizard offers to enable alternate release channels for early security patches. Enable it. You get fixes before Google publicly discloses the vulnerabilities. The window between "patchable" and "publicly known" is exactly the window targeted attacks exploit.

---

## Owner Profile Hardening

The Owner profile is special on GrapheneOS. It cannot return to BFU (Before First Unlock) state without a full device reboot. Everything here stays decrypted in memory as long as the phone is on. So we keep it absolutely minimal.

### Duress Credentials

`Settings > Security & privacy > Device unlock`

Set both a duress PIN and a duress password. When entered at any lock screen prompt: instant, irreversible, uninterruptible factory reset. All profiles, all data, all eSIMs. Gone. No confirmation dialog.

Make them plausible but completely different from your real credentials. Not one digit off. Not similar words. An accidental wipe is permanent.

### Auto-Reboot: 8 Hours

`Settings > Security & privacy > Device unlock > Auto reboot`

If the phone sits locked for 8 hours, it reboots to BFU. All encryption keys purged from memory. By morning, your phone is cryptographically equivalent to a powered-off device.

Default is 18 hours. Too generous.

### USB Restrictions

`Settings > Security & privacy > More security & privacy`

Set to "Charging-only when locked except before first unlock."

In BFU: USB is completely off. Not even charging. The port is electrically dead. A forensic tool sees nothing.

In AFU locked: charging only, no data.

Unlocked: full USB access.

### Sensors Default Deny

`Settings > Security & privacy > More security & privacy`

Disable "Allow sensors permission to apps by default." Accelerometer, gyroscope, compass, barometer -- all denied unless explicitly granted. These sensors can fingerprint your device and infer typing patterns.

### Network Hardening

- Private DNS: set to `dns.mullvad.net` (DNS over TLS, encrypted queries)
- Location settings: disable Wi-Fi scanning and Bluetooth scanning
- Enable 2G network protection (blocks IMSI catchers and fake base stations)
- Disable public network notifications (stops probe requests)
- Disable WEP network connections (cryptographically broken since 2001)

### Display and Notifications

- Screen timeout: 30 seconds
- Disable lift to wake, tap to wake
- Notification history: off
- Lock screen notifications: hide content

### Keyboard

Open your keyboard settings and kill everything: autocorrect, suggestions, personalized learning, key sounds, key vibration. The keyboard should be a dumb input device that remembers nothing. Published research (Georgia Tech, 2012) demonstrated keystroke inference from accelerometer data captured through sound. Your keyboard should produce zero observable side-channel output.

### System Sounds

Disable touch sounds, screen lock sounds, charging sounds. A phone sitting on a table should sound identical whether you are typing a password or doing nothing.

---

## The Profile Architecture

This is where GrapheneOS becomes a weapon. Each user profile gets its own encryption keys, its own data directory, its own isolated environment. When you "end session" on a profile, its encryption keys are purged from memory. That profile returns to BFU. Cryptographically unreachable.

### Profile 0: Owner (Minimal)

- Phone app (calls, because SIM lives here)
- Messages (SMS for OTP codes)
- Settings
- GrapheneOS App Store
- Nothing else

### Profile 1: Daily Driver

- Mullvad VPN (kill switch, DNS filtering)
- Aegis (2FA tokens)
- Signal (E2E messaging)
- SimpleX Chat (anonymous messaging)
- Vanadium (hardened Chromium browser, incognito default)
- Organic Maps (offline navigation)
- Tor Browser (anonymity layer)
- Calendar (offline, no sync)
- MuPDF (minimal PDF reader)
- FairEmail (standard email accounts)
- Obtainium (app updates from developer repos)

No Google infrastructure touches this profile.

### Profile 2: Google Sandbox

- Sandboxed Google Play Services
- Mullvad VPN (fixed server in your country, different from Profile 1)
- WhatsApp
- Telegram
- Banking apps
- Vanadium (persistent Twitter/X session only)

This profile accepts the tradeoff. Google, Meta, and your bank know who you are here. The compartmentalization means their telemetry is contained and cannot see Profile 1.

---

## App Installation Strategy

Priority order for sources:

1. GrapheneOS App Store (system apps, Play Services)
2. Obtainium (pulls APKs directly from developer GitHub repos -- Signal, Mullvad, Aegis, Organic Maps, MuPDF, Tor Browser)
3. Play Store (last resort -- WhatsApp, banking apps, anything requiring Google)

Avoid F-Droid for security-critical apps. F-Droid re-signs apps with their own keys (adding a trusted third party) and has significant update delays.

Install Obtainium first in each profile. After installing it from Vanadium, immediately revoke Vanadium's "install unknown apps" permission.

---

## VPN Configuration

### Profile 1: Mullvad, Rotating Servers

Install before anything else touches the network. Always-on VPN with kill switch at the Android system level. DNS filtering enabled (blocks ads, trackers, malware). WireGuard protocol.

### Profile 2: Mullvad, Fixed Server

Same kill switch and DNS filtering, but pinned to a specific server in your country. Why? This profile has banking apps and Google. A fixed local IP reduces attestation failures, CAPTCHA walls, and security alerts. Anonymity is not the goal here -- the identity is already exposed. Stability and encryption in transit are.

Different server from Profile 1. Different WireGuard key. No correlation between profiles.

### VPN goes in each profile independently

The Owner profile gets no VPN. There is almost no IP traffic there -- calls go over cellular voice, SMS over carrier signaling. Private DNS covers the only meaningful leak vector.

---

## App Hardening Highlights

### Signal

- Registration Lock: enabled
- Screen Security: enabled (blocks screenshots and app switcher preview)
- Always Relay Calls: enabled (hides your IP from call recipients)
- Disappearing messages default: 1 week
- Signal PIN: set and memorized

### WhatsApp

- Last seen: Nobody
- Profile photo: My Contacts
- Read receipts: off
- Two-step verification: enabled with unique PIN
- Chat backup: OFF (no Google Drive backups, ever)
- Auto-download media: disabled on all networks

### Telegram

- Phone number visibility: Nobody
- Find by number: My Contacts only
- Last seen: Nobody
- Peer-to-peer calls: Nobody (prevents IP leakage)
- Two-step verification: enabled
- Passcode lock: enabled

Remember: regular Telegram chats are NOT end-to-end encrypted. Only Secret Chats are. Treat regular Telegram as if their servers read everything -- because they can.

### Twitter/X

Skip the app. Use the mobile website in Vanadium. The native app fingerprints your device, reads installed apps, and runs background tracking. The website is sandboxed by the browser engine. Add `x.com` as a home screen shortcut.

---

## Operational Discipline

The setup is nothing without discipline.

- End sessions on profiles when done. Do not just switch to Owner. Explicitly end it. Keys purged, profile returns to BFU.
- The 8-hour auto-reboot is your safety net, not your strategy.
- Check for GrapheneOS system updates weekly.
- Audit app permissions monthly. Apps request new permissions after updates.
- The phone is not a computer. SSH, infrastructure management, sensitive research -- all of that stays on your computer.
- Use the phone less. Every minute the screen is on is a minute the profile sits in AFU with keys in memory.

---

## The Bottom Line

The most secure phone is the one you use less. But when you do use it, make sure it is built right.

Three profiles. Three separate encryption domains. Google contained in a sandbox. Daily communication isolated from real-identity apps. Owner profile so minimal it barely exists. Auto-reboot purging keys while you sleep. USB dead in BFU. Duress credentials ready.

Your phone went from the weakest link to a hardened node.

Now go touch grass. Preferably with the phone locked in your pocket, slowly counting down to BFU.

---

#GrapheneOS #Privacy #InfoSec #MobileSecurity #OPSEC #Hardening
