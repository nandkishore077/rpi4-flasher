# Raspberry Pi 4 Flasher

A free, browser-based tool that writes a compiled bare-metal `kernel8.img` straight onto a Raspberry Pi 4's SD card — no separate flashing app, no manual `dd`, no hand-editing `config.txt`.

**🔗 Live tool:** https://nandkishore077.github.io/rpi4-flasher/

No install, no signup, no cost. Open the link in Chrome or Edge and use it.

---

## What it does

1. **Mount** — pick your SD card's boot partition through a normal file picker.
2. **Upload** — drag in your compiled binary. It's written as `kernel8.img` regardless of its local filename.
3. **Flash** — one click:
   - Checks for `bootcode.bin`, `start4.elf`, `fixup4.dat` — fetches any missing ones from the official [raspberrypi/firmware](https://github.com/raspberrypi/firmware) repo
   - Checks `config.txt` — writes sensible defaults if missing, or **non-destructively** patches just the lines needed to boot (never wipes out existing settings)
   - Writes the image, then reads it back and verifies with a SHA-256 checksum

## Requirements

- **Google Chrome or Microsoft Edge, on desktop.** Not Safari, not Firefox, not mobile — they don't support the File System Access API this relies on.
- An SD card that's **already had Raspberry Pi OS imaged onto it once** (via Raspberry Pi Imager or Balena Etcher). This tool writes into an existing boot partition — it can't partition a blank card from scratch.

## Why

The usual bare-metal test loop — eject card, mount on laptop, copy file, sometimes hand-fix `config.txt`, reinsert — is slow and easy to get subtly wrong (a stale `kernel=` override or a missing firmware file causes a boot failure with no obvious error). This collapses that into: drag file, click button.

## Privacy & safety

- Nothing is uploaded anywhere. Your kernel image never leaves your computer — the only network request this tool ever makes is fetching *official* Raspberry Pi firmware files from GitHub, and only if your SD card is missing them.
- Every write is verified with a checksum read-back, and every config.txt change is logged in the on-page console — nothing happens silently.
- No accounts, no tracking, no ads.

## License

MIT — free to use, modify, and share.
