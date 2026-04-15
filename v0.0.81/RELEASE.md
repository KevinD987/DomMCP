# DomMCP Public Release v0.0.81

## Artifacts
- `linux-x86_64/dommcp_addin`
- `linux-x86_64/dommcp_addin-linux-x86_64-glibc2.34`
- `linux-x86_64/SHA256SUMS`

## Version marker
- Server version string: `0.0.81`
- Build lineage: `ba87b26` branch state at release preparation time

## Integrity
Validate after download/copy:

```bash
cd linux-x86_64
shasum -a 256 -c SHA256SUMS
```

Expected checksum:

```text
1e44b81d44e0fa94e516b18020b74bf8025ca46a0763a5c8c964a235930b7e4e  dommcp_addin
71216cd3e21de0cc3f6ea7edc25afb6f5ea3f4f6aa5da3f16e15ef74dfc759a1  dommcp_addin-linux-x86_64-glibc2.34
```

## Install (minimal)
```bash
# 1) copy binary to Domino host
cp dommcp_addin /opt/hcl/domino/notes/latest/linux/dommcp_addin
chmod +x /opt/hcl/domino/notes/latest/linux/dommcp_addin

# 2) restart add-in with config
domino cmd "tell dommcp quit"
domino cmd "load dommcp_addin /local/notesdata/dommcp-default-config.json"
```

## Platform requirement
- Linux x86_64
- **HCL Domino 14.5 required**

## License behavior
- Starter: curated baseline runtime; without license the server runs in `starter_default_unlicensed`.
- Professional/Enterprise: enabled through signed entitlements.
- Missing/invalid/expired/incompatible license: Professional/Enterprise upgrade paths remain blocked; Starter baseline remains available.
