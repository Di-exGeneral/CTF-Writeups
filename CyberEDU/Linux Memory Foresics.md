# Linux Memory Foresics

**Category:** Forensics \
**Difficulty:** Easy \
**Status:** Solved \
**Flag:** `SSS{m3m0ry_1s_3v3rywh3r3}` 


## Summary

A Linux memory dump (`memory.dump`) provided with no other context. Initital assumption it was a Windows image (based on habit) was wrong, `file` identified it as an ELF core dump. Getting Volatility 3 to work against this exact kernel build turned into a rabbit hole, but the flag was ultimately found via a direct binary string scan rather than structured memory analysis


## Recon

```sh
file memory.dump
```

Output

`ELF 64-bit LSB core file, x86-64` - a linux image, not Windows

```sh
vol -f memory.dump linux.vmcoreinfo.VMCoreInfo
vol -f memory.dump banners.Banners
```

Identified the exact kernel: `5.15.0-185-generic` (Ubuntu 22.04 "jammy", build `#195-Ubuntu`)


## The symbol table rabbit hole
 
Volatility 3 needs a matching ISF symbol table to walk Linux kernel structures, and unlike Windows (which it auto-fetches from Microsoft's symbol server), Linux symbol tables have to be supplied manually, usually built from the distro's debug kernel package via `dwarf2json`.
 
This build wasn't available:
- Not present in the live `ddebs.ubuntu.com` pool (older point releases get pruned)
- Ubuntu's `ddebs` repo GPG key was rejected outright by modern `apt`/`sqv` (SHA1-signing policy deprecation)
- Not found on Launchpad's build archive for this specific point release either
At this point, structured process/memory analysis via Volatility was blocked pending a symbol table that couldn't be sourced for this exact build.


## Pivot: raw string scanning

Rather than continuing chasing kernel symbols, went back to scanning raw dump for embedded plaintext:

```sh
strings memory.dump | frep -iE "flag|secret|password"
```

This surfaced a `passwd` log entry (`password for 'ctf' changed by 'root'`) confirming a `ctf` user account existed, a reasonable but ultimately unrelated lead. A naive search for `VCD{` (based on a coincidental byte sequence spotted near that log entry) turned out to be a false positive: matches were sitting inside systemd journal binary headers, not real flag text.

The actual approach that worked, a broad regex scan across the whole dump for *any* printable `tag{contents}` pattern, not assuming a specific flag prefix:

```python
import re
data = open('memory.dump', 'rb').read()
text = data.decode('latin-1')
matches = re.findall(r'[A-Za-z0-9_]{2,15}\{[A-Za-z0-9_\-\.]{10,80}\}', text)
```

Buried among hundreds of `udev`/`env{...}` rule matches (false positives from device management rules) was one genuine outlier: `SSS{m3m0ry_1s_3v3rywh3r3}`.


## Lessons

- **Check the file type before assuming the platform:** Defaulting to Windows-oriented Volatility plugins wasted several steps before `file` settled it in one command

- **Symbol-table availability is a real constraint, not a formality.** Older or less common kernel builds can be genuinely unobtainable through normal channels (pruned repos, deprecated signing keys, archives that don't mirror every point release). Worth checking availability *before* committing to a structured-analysis approach.

- **A dumb, broad string scan is a legitimate first (or fallback) move in memory forensics.** Many CTF-style memory challenges plant the flag as plaintext in the image, discoverable without ever needing to parse kernel structures, worth trying early, since it's nearly free, rather than only as a last resort.

- **Don't assume a flag's prefix.** Searching for an assumed format (`CTF{`, `flag{`) can miss a differently-branded flag entirely; a generic `tag{...}` pattern scan is more robust when the format isn't specified upfront.
