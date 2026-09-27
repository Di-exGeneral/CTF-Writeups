# Alternating

**Category:** Misc / File Forensics
**Difficulty:** Medium
**Status:** Solved
**Flag:** `ctf{7ce5567830a2f9f8ce8a7e39856adfe5208242f6bce01ca9af1a230637d65a2d}`
**Tools:** unrar, 7z, binwalk, ntfs-3g, Wine + WinRAR

## Summary

A `.rar` archive that listed only one empty file (`Flag.txt.txt`) under Linux tooling, with a taunt in the challenge description: *"Make sure to extract using WinRAR. Windows is your friend."* The real content was hidden in an NTFS Alternate Data Stream (ADS) attached to that file, a feature that only exists on NTFS and is invisible to standard Linux archive tools and filesystems.

## Recon

```
file Flag.rar
xxd Flag.rar | head -20
```

RAR5 archive. A hex dump revealed two embedded filenames inside the archive structure: `Flag.txt.txt` and `real_flag.txt`, plus an `STM` marker, RAR5's service-record indicator for a stream attached to a file entry.

```
unrar l Flag.rar      # lists only Flag.txt.txt, size 0
7z l -slt Flag.rar     # same, one visible entry
binwalk Flag.rar       # no embedded/appended file structures found
```

## Exploitation

The `STM` + `real_flag.txt` signature confirmed this wasn't a corrupted archive, it was an NTFS ADS: `Flag.txt.txt:real_flag.txt`. ADS is a real filesystem feature (a file can carry hidden secondary streams), which is why WinRAR on real Windows/NTFS surfaces it, and why Linux tools extracting onto ext4 silently drop it, the target filesystem doesn't support the concept at all.

Attempts to work around this on Linux:

- `unrar x` / `7z x` onto ext4, stream dropped or errored (`7z` actually created a `file:stream`-named file but failed to decompress it with "Unsupported Method")
- Mounting a loopback NTFS image and extracting onto it, got further (7z could see the stream), but the kernel's built-in `ntfs3` driver doesn't support the `streams_interface=windows` mount option that exposes ADS as accessible paths, and `ntfs-3g` alone still didn't materialise a readable stream

None of the Linux-native paths fully worked. Given the challenge explicitly pointed at Windows tooling, the reliable fix was to stop fighting the platform mismatch:

```
sudo apt install wine -y
winecfg
wget https://www.rarlab.com/rar/winrar-x64-701.exe
wine winrar-x64-701.exe
```

Extracting through actual WinRAR (via Wine) correctly surfaced and extracted the ADS, revealing the real flag content.

## Lessons

- NTFS Alternate Data Streams are a genuine, if obscure, way to hide data, invisible in a normal directory listing (`ls`, `dir` without switches), invisible to filesystems that don't support the NTFS stream model, and easy to miss if you only trust your archive tool's top-level file list.
- A challenge hint like "use WinRAR, Windows is your friend" is worth taking literally rather than treating as flavour text, it was a direct pointer to a platform-specific feature, not just a suggestion of convenience.
- When Linux-native tools disagree with each other (`unrar` silent, `7z` errors on decompression), that inconsistency itself is a signal worth digging into rather than picking whichever tool is quietest.
