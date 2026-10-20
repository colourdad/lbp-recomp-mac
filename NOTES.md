# Notes

Running log for the port: what was tried, what broke, exact errors, guest addresses, fixes.

## 2026-10-20: Phase 1 (reconnaissance)

- Read-only recon from a Linux cloud container; nothing built. Findings in RECON.md.
- benjidotsh/LittleBigRecomp already runs LBP1 (BCES00141 v02.00) on macOS arm64 to the Introduction level.
  Its ps3recomp submodule pin `8a6a0cb` is not public; public `benjidotsh/ps3recomp` `lbp` is `d15591b` (v0.15.0 + 7).
- GitHub REST API is blocked from the cloud container; git over https and github.com pages work.
