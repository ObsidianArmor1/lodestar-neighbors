# Lodestar global neighbour pack

Precomputed visual neighbours for the Lodestar corpus: **999,693 Street View
panoramas**, each with its **top-300 most visually similar panoramas drawn from
the whole corpus**.

Static files only. A client fetches one ~0.5 MB chunk per lookup — no server, no
GPU, no inference in the browser.

## Why global neighbours

Earlier packs were built per map, because the corpus *was* the map: a 49k map
ranked each panorama against its own 49k. This pack ranks every panorama against
all 999,693, so a location's references come from anywhere in the corpus.

The consequence is that **there are no per-map artifacts**. Any map cut from
Lodestar works against this one dataset, including maps that do not exist yet.

## Layout

```
directory.bin.gz          999,693 x 22-byte ascii panorama id
                          then 999,693 x (int32 lat*1e6, int32 lng*1e6)
neighbors/NNNNN.bin.gz    512 rows: [512 x 300 int32 indices][512 x 300 float16]
manifest.json             sizes and sha256 for every file
```

Row `r` lives in chunk `r // 512` at row offset `r % 512`. The directory doubles
as the routing table: a panorama's position in it **is** its row, and neighbour
indices are positions in that same directory. Coordinates are quantised to 1e-6
degrees (~11 cm).

## Provenance

Vectors are C-RADIOv4-H (`nvidia/C-RADIOv4-H`, revision `0057b339`), four views
per panorama at 448x256, `thumbfov=90`, offsets 0/90/180/270 from each
panorama's own spawn heading. Per panorama: normalised four-view summary mean
concatenated with the normalised four-view coarse-spatial mean, renormalised —
3,840 dimensions.

Neighbours come from an exact top-500 table: float16 shortlist of 2,000 per row
rescored in float32, verified against float32 brute force on sampled rows
(500/500 overlap, top-1 agreement on every row checked). This pack ships the
first 300, which preserves the rank-288 boundary scan the trainer performs.

307 panoramas of the frozen 1,000,000 are absent: Google serves a black frame
for them, and the extraction's quality gate refused to embed a black frame.
