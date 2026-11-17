# pvr.sublime

The Kodi TV client of [service.tvlink](https://github.com/patkino2022/service.tvlink):
live TV, archive (catchup) and guide. It is AlexELEC's
[pvr.tvlink](https://github.com/AlexELEC/pvr.tvlink) 22.0.6 (GPL-2.0-or-later)
with its own id and build, so the boxes
no longer depend on the pvr.tvlink that ships inside the AlexELEC image and can
move to official CoreELEC.

What differs from upstream:

* id `pvr.sublime`, data in `addon_data/pvr.sublime`, server 127.0.0.1 by default;
* startup: when the first playlist load fails (Kodi starts PVR clients ~1.2 s
  after start, python services ~2.7 s), the client retries every 2 s for a
  minute, then every 30 s, instead of waiting for its next refresh hour.

## Build

GitHub Actions (`.github/workflows/build.yml`) cross-compiles for aarch64 against
the header-only kodi-dev-kit of the exact Kodi the boxes run (xbmc 4665ec22a4,
22.0 RC1, `kodi.binary.instance.pvr` 9.2.0), with pugixml and a static zlib;
libstdc++/libgcc are linked in. The artifact `pvr.sublime-<ver>-aarch64.zip`
installs like any add-on zip.

When the boxes' Kodi changes its PVR API version, change `XBMC_REF` and rebuild:
`addon.xml` takes its `<import>` versions from that `versions.h`.
