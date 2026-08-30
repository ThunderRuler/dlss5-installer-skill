# Known hashes — `nvngx_dlssnr.dll` v310.8.0

The neural runtime must pass Authenticode verification. A `HashMismatch` copy produces
`STANDBY/FAILED` and `feature 18 create failed with 0xBAD00002`, and it circulates widely
— library installers have been observed deploying it by default, and it has been found
pre-seeded into half a dozen games on one machine.

Check both, always:

```powershell
Get-AuthenticodeSignature "path\to\nvngx_dlssnr.dll" | Select-Object Status
Get-FileHash "path\to\nvngx_dlssnr.dll" -Algorithm SHA256
```

| SHA256 (full) | Status | Verdict |
|---|---|---|
| `E16BCF15E16E13F527491CDF7845B2FE6521A738D8F7C9C721866A8496E1FC8E` | Valid | **known-good** — the clean signed build |
| `4B8D19BC3EFF58A084F5ECA7489C921501C203450169FB82FF4F649A4482BA05` | HashMismatch | **known-bad** — modified after signing; causes STANDBY/FAILED |

Notes:

- The RenoDX DLSS5 addon prints `Runtime sha256: E16BCF15... (reference match)` in its
  overlay when the runtime is the expected build — that line is stronger evidence than an
  Authenticode check alone.
- A `.original` backup sitting next to a bad copy has been observed to carry the **same
  bad hash** — never assume a backup is clean; hash it.
- RHI caches a Valid copy at `%LOCALAPPDATA%\RHI\DLSS-NR\310.8.0\nvngx_dlssnr.dll`.
- Sweep a whole library for copies (good and bad) in one pass:

```powershell
Get-ChildItem "D:\SteamLibrary\steamapps\common","C:\Program Files (x86)\Steam\steamapps\common" `
  -Recurse -Filter "nvngx_dlssnr.dll" -Depth 4 -ErrorAction SilentlyContinue |
  ForEach-Object {
    $h=(Get-FileHash $_.FullName -Algorithm SHA256).Hash
    "{0,-12} {1,-14} {2}" -f $h.Substring(0,12),(Get-AuthenticodeSignature $_.FullName).Status,$_.FullName
  }
```

If a new runtime version appears (≠ 310.8.0), these hashes do not apply — verify
Authenticode `Valid` status and update this file.

## Other components — verify by hash, not size

Feeder releases have shipped **byte-identical in size** across versions (v0.2.0 and
v0.3.0 `dlss5-feed.addon64` are both exactly 83,968 bytes), and the runtime log banner
can keep printing the old version string after an upgrade. `Get-FileHash` is the only
reliable identity check. Record the hash of whatever you deploy so a later "did the
upgrade actually land?" question is answerable.
