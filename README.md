# sync-COD —— COD / PCOD / TCOD source data root and data-side tools

> [Chinese (Simplified)](README-CN.md) · English

**The shared source-data directory for the two library-building repositories**, plus tools for operating on this data (acceptance + syncing with the official source).

This repository **contains only code and documentation**: 13 scripts + docs in `tools/`.
The data itself (about 120 GB / 2 million files) **is not committed**; users obtain it themselves and place it at the data root
(default `D:/Project/XRD/sync-COD`), then point the environment variable `XRD_DATA_ROOT` at it.

On 2026-10-23 the data was moved from `PolyXRD-DataBase/` to this directory (same-volume rename, done in seconds).
Reason for the move: this data is neither a product of polyDB nor of xinDB, but the **shared input** of both;
keeping it in either repository would make the other one "reference someone else's data across repositories".

```bash
git clone https://github.com/PolyXRD/sync-COD.git
export XRD_DATA_ROOT=/path/to/data-root     # point this at where COD/PCOD/TCOD live
python tools/verify_data_root.py            # verify the data is complete first
```

## Layout

```
<data-root>/                     ← recommended name: sync-COD
├── tools/                       ← tracked: 3 acceptance + 10 sync scripts (sync disabled)
├── README.md  .gitignore        ← tracked
├── .gitattributes  LICENSE*     ← tracked (LF normalization + dual license)
│
├── COD/                      535,166 CIFs   3-level layout 1/00/00/1000003.cif
├── PCOD/                   1,061,962       2-level layout (GRINSP predicted, all classified inorganic)
├── TCOD/                     362,293       3-level layout (theoretical structures)
├── inorg_cif/                                 ← copy of the data, not tracked
│   ├── COD/                   79,198       CIFs classified as inorganic
│   ├── PCOD/               1,061,962
│   └── TCOD/                  46,943
├── inorg_manifest.sqlite     289,300,480 B    per-entry manifest 1,959,421 rows (inorganic 1,188,103)
└── sync_logs/                                 sync logs and manifest archives (not tracked)
```

About 120 GB / 2 million files. `inorg_cif/` is a copy; the original archive is kept intact (read-only reference, never deleted or modified) —
so it takes up a second copy of the same data. This is deliberate redundancy, so xinDB only needs to scan the inorganic subset.

The cloned `tools/` can be dropped straight into the data root; the scripts locate the data root as "their own parent directory".
It can also live anywhere, as long as `XRD_DATA_ROOT` is set.

## Who reads it

| Consumer | Purpose |
|---|---|
| `PolyXRD-DataBase/tools/build_polydb.py` | Reads `COD/` + `inorg_manifest.sqlite`, produces polyDB |
| `PolyXRD-DataBase/tools/scan_inorg.py` | Generates `inorg_cif/` and `inorg_manifest.sqlite` |
| `XinMatch-DataBase/tools/build_cod_inorg_xindb.py` | Reads `inorg_cif/{COD,PCOD,TCOD}`, produces xinDB |
| `XinMatch-DataBase/tools/scan_inorg.py` | Same as above, generates the manifest and copies |
| `PolyXRD` project (`tools/build_polydb.py`, `scripts/audit_cod_source.py`) | Library building and source-data reconciliation |
| `XinMatch` project (`xinmatch/core/db/cif_scan.py`, `tools/_env_check.py`) | Library building and self-check |

All of them locate it via the environment variable **`XRD_DATA_ROOT`** (default: this directory); drive letters are no longer hard-coded.

## Consistency baseline (for cross-checking)

| Item | Value |
|---|---|
| Total manifest rows | 1,959,421 |
| Inorganic manifest rows | 1,188,103 (60.6%) |
| COD | 535,166 entries, of which 79,198 inorganic |
| PCOD | 1,061,962 entries, all classified inorganic |
| TCOD | 362,293 entries, of which 46,943 inorganic |
| `inorg_manifest.sqlite` size in bytes | 289,300,480 |

Running `python tools/verify_data_root.py` compares these numbers item by item and samples 1,500 files to compare byte sizes.

## Criterion (do not change)

Inorganic = **contains no C, or contains C but no H** (carbonates / carbides / CO₂ count as inorganic).
This criterion is implemented by `cif_common.is_inorganic` and directly determines the 1,188,103 figure above;
change it and the new manifest will no longer line up with the published polyDB / xinDB, making item-by-item comparison impossible.

## Notes

- The data directories (`COD/ PCOD/ TCOD/ inorg_cif/ inorg_manifest.sqlite sync_logs/`) are all in
  `.gitignore`, so they **cannot be committed by accident**. Before adding data, run `git status --short` for a look.
- When moving a large data directory, note: voidtools **Everything** and Windows Search hold directory handles,
  causing a whole-directory `mv` to fail with Permission denied (moving the child files works instead). See the workaround in
  `tools/README.md` and the READMEs of the two library-building repos.
- On this machine, `rm -rf` on large files actually **moves them to the Recycle Bin**, so free space does not increase; to truly reclaim it you must empty the Recycle Bin,
  and WorkBuddy's safe-delete blocks a second deletion inside the Recycle Bin. Don't count on `rm` to free space.

## Related repositories

| Repository | Input | Output |
|---|---|---|
| [PolyXRD-DataBase](https://github.com/PolyXRD/PolyXRD-DataBase) | `COD/` + `inorg_manifest.sqlite` | `*.polyDB` (directory-based, mmap) |
| [XinMatch-DataBase](https://github.com/PolyXRD/XinMatch-DataBase) | `inorg_cif/{COD,PCOD,TCOD}` | `*.xinDB` (single-file SQLite) |

Neither contains data; both build their libraries from the source data in this repository.
