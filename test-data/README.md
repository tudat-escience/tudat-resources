**Declaration:** this analysis and conversion was developed using an
LLM (Qwen3.8-Flash-Next).  The final results and intermediate steps
were manually verified on Linux by the committer.

**NOTE:** recipes are assuming you are in the top-level directory of
this repository.

# Subsetting/trimming the NOE & INPOP19A SPICE kernels

The full `NOE-*.bsp` kernels (~880 MB) are slow to download in CI. We shrink
them to ~155 MB with two tools: `spacit` drops whole segments by NAIF ID
(NOE-5/6), and `spkmerge` re-fits segments to a smaller time window (NOE-4).

## Tools

You can find them [here](https://naif.jpl.nasa.gov/naif/utilities_PC_Linux_64bit.html).
- [`spacit`](https://naif.jpl.nasa.gov/pub/naif/utilities/PC_Linux_64bit/spacit)
  binary is used to drop unused segments by converting to transfer
  files and back.
- [`spkmerge`](https://naif.jpl.nasa.gov/pub/naif/utilities/PC_Linux_64bit/spkmerge)
  binary is used to re-fit to smaller time window used in the test
  suite.

## Required IDs / windows (from the test suite)

| Kernel                   | Method                         | Kept                       | Result           |
|--------------------------|--------------------------------|----------------------------|------------------|
| `NOE-4-2020.bsp`         | `spkmerge` time-clip 1995–2040 | 499, 401, 402 (all bodies) | 233MB → **55MB** |
| `NOE-5-2021.bsp`         | `spacit` ID subset             | 599, 501, 502, 503, 504    | 435MB → **76MB** |
| `NOE-6-2018-MAIN-v2.bsp` | `spacit` ID subset             | 699, 606                   | 212MB → **24MB** |

NOE-4's three bodies are all needed, so it is shrunk by **time** instead of by
ID (see the spkmerge section). The test epochs span ~1999–2034; the 1995–2040
window leaves margin.

The minor moons (Amalthea, Thebe, Metis, Adrastea, Mimas, Enceladus, Tethys,
Dione, Rhea, Hyperion, Iapetus) are not referenced by any test.

> Note: NOE-4's segments are `499 rel 4`, `401 rel 4`, `402 rel 4` (offsets
> relative to the planet). SPICE chains them with `inpop19a`'s `planet rel SSB`
> segments. The subsets only work when `inpop19a_TDB_m100_p100_spice.bsp` is
> also loaded — verify with `spkezr(..., "SOLAR SYSTEM BARYCENTER")` after
> furnshing both files, not the subset alone.

## `NOE-{5,6}-*.bsp`: subset using `spacit`

`spacit` converts between binary SPK and a hex-encoded text "transfer file" (DAFETF).
Subsetting = convert to transfer, drop unwanted `BEGIN_ARRAY` blocks, convert back.

### 1. Binary → transfer file

`spacit` is interactive; drive it by piping `B`, input path, output path, `Q`uit:

```bash
mkdir -p /tmp/tudat/
printf 'B\n~/.tudat/resource/spice_kernels/NOE-5-2021.bsp\n/tmp/tudat/noe5.tf\nQ\n' | ./spacit
printf 'B\n~/.tudat/resource/spice_kernels/NOE-6-2018-MAIN-v2.bsp\n/tmp/tudat/noe6.tf\nQ\n' | ./spacit
```

### 2. Filter the transfer file

Each segment block starts with `BEGIN_ARRAY <n> <size>` followed by a line
`'SEG-<id>-...'`. Keep the 5-line header, the blocks whose ID is wanted, and the
trailing comment area after the last `END_ARRAY`:

```python
# filter_tf.py  —  usage: filter_tf.py <in.tf> <out.tf> <id> [<id> ...]
import sys, re
inp, outp, wanted = sys.argv[1], sys.argv[2], set(sys.argv[3:])
lines = open(inp).readlines()
last_end = max(i for i, l in enumerate(lines) if l.startswith("END_ARRAY"))
in_block = False
with open(outp, "w") as out:
    for i, l in enumerate(lines):
        if l.startswith("BEGIN_ARRAY"):
            m = re.match(r"'SEG-(\d+)-", lines[i + 1])
            in_block = bool(m and m.group(1) in wanted)
        if i < 5 or in_block or i > last_end:
            out.write(l)
```

```bash
python3 filter_tf.py /tmp/tudat/noe5.tf /tmp/tudat/noe5_sub.tf 599 501 502 503 504
python3 filter_tf.py /tmp/tudat/noe6.tf /tmp/tudat/noe6_sub.tf 699 606
```

### 3. Transfer file → binary

```bash
rm -rf test-data/spice_kernels/NOE-*.bsp
printf 'T\n/tmp/tudat/noe5_sub.tf\ntest-data/spice_kernels/NOE-5-2021.bsp\nQ\n' | ./spacit
printf 'T\n/tmp/tudat/noe6_sub.tf\ntest-data/spice_kernels/NOE-6-2018-MAIN-v2.bsp\nQ\n' | ./spacit
```

`spacit` will **not** overwrite an existing output file — `rm` it first if
regenerating.

### 4. Verify

```bash
python3 - <<'EOF'
import spiceypy
K = "test-data/spice_kernels/"
spiceypy.furnsh(K + "inpop19a_TDB_m100_p100_spice.bsp")   # needed for SSB chaining
for f, ids in [("NOE-5-2021.bsp", [599, 501, 502, 503, 504]),
               ("NOE-6-2018-MAIN-v2.bsp", [699, 606])]:
    spiceypy.furnsh(K + f)
    for i in ids:
        assert spiceypy.spkcov(K + f, i), f"missing {i} in {f}"
for b in ["Mars", "Jupiter", "Io", "Saturn", "Titan"]:
    spiceypy.spkezr(b, 0.0, "J2000", "NONE", "SOLAR SYSTEM BARYCENTER")
print("OK")
EOF
```

## `NOE-4-*.bsp`: time-clipping with `spkmerge`

`spacit` can only drop whole segments, so it cannot shrink NOE-4 (all three of
its bodies are needed). `spkmerge` instead **re-fits the
Chebyshev segments to a time window** — exact, no accuracy loss — which is what
shrinks NOE-4.

`spkmerge` is interactive: run `./spkmerge`, it prompts for a command file. The
command file is plain `keyword = value` lines (no `\begindata`, no quotes around
paths). Key keywords: `LEAPSECONDS_KERNEL`, `SPK_KERNEL` (output),
`SOURCE_SPK_KERNEL` (input), `BODIES`, `BEGIN_TIME`, `END_TIME`.

```
# clip_noe4.cmd
LEAPSECONDS_KERNEL = test-data/spice-kernels/naif0012.tls
SPK_KERNEL = test-data/spice_kernels/NOE-4-2020.bsp
SOURCE_SPK_KERNEL = /home/user/.tudat/resource/spice_kernels/NOE-4-2020.bsp
BODIES = ( 499 401 402 )
BEGIN_TIME = 1995 JAN 01 00:00:00
END_TIME = 2040 JAN 01 00:00:00
```

```bash
printf '/tmp/tudat/clip_noe4.cmd\n' | ./spkmerge
```

Verify the clipped file matches the original (chained with inpop19a) at sampled
epochs — the difference should be 0.0 m.

## `INPOP19a_*.asc`: time-clipping the `_asc` directories

**NOTE:** All files are not necessary, required set is recorded in
`tudatpy/tests/resource-catalog.txt`.

`inpop19a_TCB_m100_p100_asc/` and `inpop19a_TDB_m100_p100_asc/` cover
~1897–2103 (±100 yr).  The tests that read them
(`unitTestInpopFileReading`, `unitTestRelativisticTimeConversion`)
only query J2000 ± ~733 d (relativity: ±2 yr + 6 h buffer; reader: 0
and 1e7 s), except `pos_TCG.asc`, which is also queried at the
TAI-sync epoch (JD 2443144.5, 1977).

Each file is one Chebyshev segment per line: `start_JD end_JD coeffs...`
(segment length varies per file: 4 d for most bodies, 16 d for EMB/Sun/Ven,
etc.). Subsetting = keep the lines whose start JD falls in the window and fix
the header's segment count / start / end (tokens 8, 12, 14 of line 2; the
reader only uses the Chebyshev order at token 6, but keep the header
self-consistent).

```python
# clip_asc.py  —  usage: clip_asc.py <in.asc> <out.asc> <lo_jd> <hi_jd>
import sys
inp, outp, lo, hi = sys.argv[1], sys.argv[2], float(sys.argv[3]), float(sys.argv[4])
lines = open(inp).readlines()
head = lines[1].split()
data = [l for l in lines[2:] if len(l.split()) > 1]
kept = [l for l in data if lo <= float(l.split()[0]) <= hi]
head[8] = str(len(kept))
head[12] = "%.2f" % float(kept[0].split()[0])
head[14] = "%.2f" % float(kept[-1].split()[1])
with open(outp, "w") as out:
    out.write(lines[0]); out.write(" ".join(head) + "\n"); out.writelines(kept)
```

```bash
K=~/.tudat/resource/spice_kernels
T=test-data/spice-kernels/
mkdir -p /tmp/tudat/inpop_asc_backup
for d in inpop19a_TCB_m100_p100_asc inpop19a_TDB_m100_p100_asc; do
  cp -r $K/$d $T
  for f in $T/$d/*.asc; do
    case "$f" in *_header.asc) continue;; esac
    if [[ "$f" == *pos_TCG.asc ]]; then LO=2443140.0; else LO=2450445.0; fi
    python3 clip_asc.py "$f" "$f.tmp" $LO 2452645.0 && mv "$f.tmp" "$f"
  done
done
```

(J2000 = JD 2451545; window = J2000 ± 1100 d, ~3× the queried range.)

Result: 360 MB → 11.4 MB (−97%).

### Verify (if only verifying this step)

```bash
ninja -C build test_io_InpopFileReading test_relativity_RelativisticTimeConversion
./build/tests/test_io_InpopFileReading && ./build/tests/test_relativity_RelativisticTimeConversion
```


# Trimming `MCDMeanAtmosphereTimeAverage` atmospheric tables

Trimmed `atmosphere_tables/MCDMeanAtmosphereTimeAverage/` from 376 MB
to 164 MB (−56%).

## Reasoning

Each file is a 3-D grid: 150 longitude × 75 latitude × 500 altitude (50 km – 10,000 km), stored as
500 blocks of 150 lines (one per longitude) of 75 tab-separated values (latitude), with 2 blank
lines between blocks. Header: line 1 = `3` (ndim), lines 2–3 blank, lines 4–6 = lon/lat/alt axes,
lines 7–8 blank, data starts at line 9.

`TabulatedAtmosphere` uses `MultiLinearInterpolator` for the 3-D case, so removing grid nodes only
changes values *above* the cut. The test suite never queries in-range altitudes above ~400 km:

| Consumer                                                                     | In-range altitudes queried |
|------------------------------------------------------------------------------|----------------------------|
| `unitTestTabulatedAtmosphere` (corner + interpolation)                       | 5e4, 2.37e5 m              |
| `unitTestEstimationFromSpiceData` / `FromDsnData` (MGS, Nov 2005 – Feb 2006) | ~370 km                    |

Out-of-range queries must stay out-of-range (default/boundary extrapolation tests) or in-range as
before:

- 5e7 and 5e10 m → still above the cut → default values unchanged.
- 5e5 m is queried with lon/lat out-of-range and must remain **in-range** so the default is
  triggered by lon/lat only. Hence the cut keeps the first node ≥ 5e5 (node 217 = 5.0074696e5).
- 0.0 m → still below the 5e4 minimum.

Cut: keep altitude nodes 0..217 (50 km – 500.7 km), i.e. the first 218 of 500 blocks.
All interpolations below 500.7 km are bit-identical to the full table.

## Trim

```bash
cp ~/.tudat/resource/atmosphere_tables/MCDMeanAtmosphereTimeAverage/*.dat \
    test-data/atmosphere_tables/MCDMeanAtmosphereTimeAverage/
pushd test-data/atmosphere_tables/MCDMeanAtmosphereTimeAverage/
for f in density pressure temperature specificHeatRatio gasConstant; do
  python3 - "$f.dat" <<'EOF'
import sys
f = sys.argv[1]
with open(f) as fh:
    lines = fh.readlines()
axes = lines[5].split()
assert len(axes) == 500 and float(axes[0]) == 5e4, f
kept = 218                                   # alt 50 km .. 5.0074696e+05 m
out = lines[:5] + ["\t".join(axes[:kept]) + "\n"] + lines[6:33142]
with open(f, "w") as fh:
    fh.writelines(out)
EOF
done
```

`33142` = last line of kept block 217 (block *i* spans lines `9 + 152*i` to `158 + 152*i`).

## Verify (if only verifying this step)

```bash
ninja -C build test_aerodynamics_TabulatedAtmosphere
./build/tests/test_aerodynamics_TabulatedAtmosphere   # 10/10 pass
```

The two OD tests are not in this build config; they only propagate MGS at ~370 km, below the cut.


# Build & test everything

- Set `$TUDATPY_RESOURCE_DIR` to dir with trimmed spice kernels (`test-data/`).
- Then `cmake --build build -j4` and `(cd build; ctest -j4)`
— all 345 tests should pass.
