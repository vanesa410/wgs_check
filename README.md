# wgs_check

Audit sequencing deliveries stored on an rclone remote (ownCloud / WebDAV) and
report, per sample, whether the delivery looks complete.

Expects the layout `<STUDY>/<ID>/<SUBFOLDER>/`, e.g.

```
STUDY/
  SAMPLE01/
    WGS/
      SAMPLE01_R1.fastq.gz
      SAMPLE01_R2.fastq.gz
      SAMPLE01_checksums.txt
  SAMPLE02/
    WGS/
      ...
```

Only directory metadata is read. File contents are never downloaded, so the
audit is cheap and nothing is staged on the cluster.

## Requirements

- Python 3.6+
- `rclone`, configured with a remote pointing at the storage
- `openpyxl` (only for `--xlsx`): `pip install openpyxl`

## Usage

```bash
python3 wgs_check.py remote:STUDY_NAME -o study_wgs.csv --xlsx study_wgs.xlsx
```

Other studies and folder layers follow the same shape:

```bash
python3 wgs_check.py remote:OTHER_STUDY -o other.csv
python3 wgs_check.py remote:STUDY --subfolder WES
```

The scan takes roughly two requests per ID and prints nothing until it
finishes, so for a study with ~1000 IDs expect several minutes of silence.
Running it in the background keeps the terminal usable:

```bash
nohup python3 wgs_check.py remote:STUDY_NAME \
    -o study_wgs.csv --xlsx study_wgs.xlsx \
    --rclone /usr/bin/rclone > wgs_scan.log 2>&1 &
```

It is a light, network-bound job, so it runs fine on a login node without a
PBS queue.

### Options

| Option | Default | Meaning |
| --- | --- | --- |
| `-o`, `--out` | `wgs_report.csv` | output CSV |
| `--xlsx FILE` | off | also write a formatted Excel copy |
| `--subfolder` | `WGS` | per-ID data folder |
| `--checksum-glob` | `*checksum*.txt` | checksum filename pattern, case-insensitive |
| `--rclone` | `$RCLONE`, else `rclone` | rclone binary to use |

## Output

One row per fastq file. Per-sample values repeat across that sample's rows.
Samples with no fastq files still get a row, so nothing is silently dropped —
an ID with an empty or missing WGS folder is exactly what the audit is for.

| Column | Meaning |
| --- | --- |
| `sample_id` | ID folder name |
| `file_name` | fastq file |
| `size_gb` | file size (GiB, 1024³) |
| `read` | `1`, `2`, or `?` if unparseable |
| `has_checksum` | TRUE/FALSE |
| `checksum_file` | which file matched, so the check is auditable |
| `fastq_gz_count` | fastq files in the folder |
| `r1_count`, `r2_count` | reads per side |
| `size_gb_diff` | \|total R1 − total R2\|, blank if unpaired |
| `complete` | `yes` / `no` |
| `note` | why a sample is incomplete |

The Excel copy shades alternate samples so each sample's R1/R2 rows read as one
block, and adds a frozen header row with filters.

### Completeness

`complete = yes` requires **a checksum file** and **R1 files balancing R2
files**.

Balance rather than "exactly two files" is deliberate: a sample sequenced across
two lanes has four fastq files and is perfectly healthy, while three files can
never pair. Read numbers are parsed from `_R1`/`_R2`, `_1`/`_2` and `_R1_001`
style names, anchored on a separator so digits inside sample names (`L002`,
`00027`) don't false-match.

`size_gb_diff` is informational. R1 and R2 of a genuine pair hold the same read
count, so their sizes normally track within a few percent — a large gap suggests
one side was truncated in transfer.

## A note on outputs

The CSV and XLSX contain sample IDs and filenames. They are patient-linked
metadata, so they are git-ignored here and should not be committed.
