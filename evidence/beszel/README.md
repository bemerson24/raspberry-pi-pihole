# Beszel monitoring evidence — November 16, 2026

These images derive from four genuine screenshots supplied from the `home-dns` Beszel dashboard. They show system health and time-series panels in the displayed ~11:40 AM–12:40 PM Pacific window. They do not prove long-term monitoring, alert delivery or security incident detection.

The first three images retain their content (PNG normalized to RGB). Image 04 was cropped to exclude an incomplete container table exposing a LAN address and had the Wi-Fi SSID obscured with an opaque mask. Graph lines, axis labels and other technical measurements were not redrawn. The underlying private identifier is not included in the public copies. Full source images should stay private.

**Evidence scope:** Reports dashboard presentation at capture time, not independently exported telemetry. For the cause and independent Docker stats corroboration of the cgroup repair, see `case-studies/docker-uptime-kuma-beszel-monitoring-2026-11-15.md` in the Pi-hole repository.

This set does not include a Uptime Kuma dashboard screenshot. That evidence was inspected in the prior investigation but is not part of this export.

## SHA-256 evidence provenance

- `01-beszel-host-and-docker-cpu-2026-11-16.png`: `bd5de32a9f12ee7e097cb4c4eef25a537c77beb524d32a16cc19da77b78d9538` (source: `image(20261116-204025).png`; source SHA-256 `247654b816bbf65474606b33d284590c61e3e265d0aa83f54b49a683edeee269`)
- `02-beszel-docker-memory-and-storage-2026-11-16.png`: `a5610e7bb4e6545c4a562c0922bb3572981a058b773e3acf9439e0871fb224e8` (source: `image(20261116-204041).png`; source SHA-256 `a2982d8a38a5f0b2d170e5a54cfe2100510134dd9e7334d5c964723c4e1c746e`)
- `03-beszel-network-and-load-2026-11-16.png`: `353755b7aacb0246991cce0c6ffab92d8ca48ece5941a0cc6421d25d24add153` (source: `image(20261116-204057).png`; source SHA-256 `94262da61a5f35b4f49ad8358aa4ea90c48944ac6b384383a4750c879c20c57e`)
- `04-beszel-temperature-wifi-2026-11-16.png`: `d7197723ab8fd83b1d74c16d84908336788efa2c0906bd25fbd1396c15eb1e20` (source: `image(20261116-204111).png`; source SHA-256 `5e80b4e00a2a8458b55f6677faab56dde1477ead30e41296adce9fbc7c432756`)