# Parallel PRANK

**Parallel PRANK** is a lightweight Python wrapper for running independent [PRANK](https://ariloytynoja.github.io/prank-msa/) multiple-sequence alignments concurrently. Rather than supplying one FASTA file with PRANK's `-d` option, you supply **one or more folders containing FASTA files**. Each file is processed by a separate PRANK process, and `-j` limits how many alignments run concurrently. On Linux, each running alignment is pinned to a distinct logical CPU by default, including subprocesses launched by PRANK.

Useful for gene-by-gene alignments, including chloroplast and mitochondrial CDS datasets, without having to run hundreds of PRANK commands manually.

## Features

- **Folder-based inputs:** align every supported FASTA file in an input directory.
- **Multiple input/output pairs:** match comma-separated lists by position (`-i cpDNA,mtDNA -o cpDNA_prank,mtDNA_prank`).
- **Bounded parallel execution:** choose the maximum number of simultaneous PRANK alignments using `-j` / `--jobs`.
- **Exclusive logical-CPU slots (Linux):** use `taskset` to pin each alignment and its subprocesses to one available logical CPU; slots are released when an alignment completes. Single-threaded library settings limit nested parallelism. Use `--no-pin` to opt out.
- **Native PRANK flags:** forward normal PRANK options (e.g. `-codon`, `-F`, `-once`, `-iterate=3`, `-showtree`) to each alignment.
- **Optional previous behavior:** `--default` adds `-codon -F`; these flags are **not** added without `--default` unless specified manually.
- **Automatic codon-length filtering:** whenever `-codon` or `--default` is present, check the ungapped length of every FASTA sequence and exclude records whose length is not divisible by three before alignment. Original input files are never modified.
- **Filtering audit trail:** list every excluded sequence (ID, ungapped length, remainder) at the beginning of its `.prank.log` file, before PRANK output.
- **Per-file logs:** collect each alignment's PRANK stdout and stderr in `<name>.prank.log`.
- **Combined help:** `-h` prints wrapper options followed by the installed PRANK's help, excluding PRANK input/output controls managed by the wrapper.

## Requirements

- Python **3.8+** (standard library only).
- PRANK installed, with its executable named `prank` and accessible in your `PATH`.
- On Linux (default affinity mode), `taskset` from `util-linux` must be available in `PATH`. Other platforms run without pinning.
- FASTA files with sequences to align, typically one file per gene.

Check the installation with:

```bash
prank -help
python3 parallel_prank.py -h
```

## Installation

Download or clone this repository, then run the script with Python:

```bash
python3 parallel_prank.py -i genes -o aligned_genes -j 8
```

To call it as `parallel_prank` from anywhere on Linux, make it executable and add a symlink from a folder on your `PATH` (for example, `~/.local/bin`):

```bash
chmod +x parallel_prank.py
mkdir -p ~/.local/bin
ln -s "$(realpath parallel_prank.py)" ~/.local/bin/parallel_prank
# Ensure ~/.local/bin is on your PATH.
```

## Usage

```text
parallel_prank -i INPUT_DIR[,INPUT_DIR...] -o OUTPUT_DIR[,OUTPUT_DIR...] -j N [--default] [--no-pin] [PRANK options...]
```

| Wrapper parameter | Purpose | Required? |
| --- | --- | --- |
| `-i`, `--input` | One or more comma-separated folders containing FASTA files | Yes |
| `-o`, `--output` | Comma-separated destination folders, paired with inputs in the same order | Yes |
| `-j`, `--jobs` | Maximum number of **simultaneous PRANK processes** (positive integer) | Yes |
| `--default` | Automatically use `-codon -F` on every PRANK alignment | No |
| `--no-pin` | Disable Linux per-alignment CPU pinning (permits `-j` greater than accessible CPU count) | No |
| `-h`, `--help` | Show wrapper help followed by help from installed PRANK | No |

**All three arguments `-i`, `-o`, and `-j` are required.** There is no default job count or predefined input folder. When `--default` is absent, PRANK receives only the input/output locations and the PRANK options you explicitly supply. Note that supplying `-codon` manually enables the same FASTA length validation and filtering as `--default`.

### Example 1: One folder with original script's alignment options

```bash
parallel_prank -i cpDNA_genes -o aligned_cpDNA_prank -j 8 --default
```

This runs up to eight alignments simultaneously. Each FASTA is first checked for coding sequence lengths divisible by three. For a FASTA containing only valid sequences, the underlying PRANK invocation is equivalent to:

```bash
prank -d=cpDNA_genes/rbcL.fasta -o=aligned_cpDNA_prank/rbcL -codon -F
```

### Example 2: Multiple folders simultaneously

```bash
parallel_prank \
    -i mtDNA_genes,cpDNA_genes \
    -o aligned_mtDNA_prank,aligned_cpDNA_prank \
    -j 12 --default
```

Folder pairs are matched **by position**, not by name:

- `mtDNA_genes` → `aligned_mtDNA_prank`
- `cpDNA_genes` → `aligned_cpDNA_prank`

A single, shared pool of **up to 12** PRANK alignments is used across *both* datasets (not 12 per directory). On Linux, these are assigned 12 distinct available logical CPUs.

If the numbers of input and output directories differ, the script **aborts before starting any alignments**. For example, `-i mtDNA_genes,cpDNA_genes -o aligned_genes` is invalid.

### Example 3: PRANK's own options, no preset

```bash
parallel_prank -i genes -o aligned -j 6 -codon -F -once -showtree
```

Or use PRANK's non-codon alignment behavior without any additional flags:

```bash
parallel_prank -i genes -o aligned -j 6
```

PRANK flags are passed unchanged to every alignment. As with PRANK itself, options with values should use its `-option=value` syntax, such as `-iterate=3` or `-f=phylipi`. With `-codon`, the wrapper automatically filters records whose ungapped lengths are not multiples of three; without `-codon` or `--default`, **no length check or filtering** occurs.

### `-j` controls alignments, with exclusive CPU slots on Linux

For example, `-j 24` means **up to twenty four independent PRANK alignments** running at the same time. The wrapper uses Python threads to supervise these processes; it does not parallelize an individual PRANK alignment.

```bash
parallel_prank -i cpDNA_genes -o aligned_cpDNA_prank -j 24 --default
```

With 24 eligible logical CPUs, up to 24 PRANK alignments run concurrently, with one CPU slot reserved per alignment.

## Automatic codon-length validation

When the PRANK `-codon` flag is explicitly provided **or** `--default` is used (which adds `-codon -F`), Parallel PRANK checks **each sequence within each FASTA file** before starting that alignment:

1. Calculate each sequence's **ungapped nucleotide length**, ignoring `-` characters and whitespace, as in `cds_3mult_cleaner.py`.
2. Keep sequences when `length % 3 == 0` and exclude others from the corresponding alignment.
3. When exclusions are necessary, write only the retained records to a **temporary FASTA** and use that file for PRANK. Temporary files are deleted after the job; original input FASTAs remain unchanged.
4. Write the validation summary and all excluded sequence IDs, lengths, and remainders to the **top of the per-alignment `.prank.log`**. PRANK's own output follows this section.

For example, when running:

```bash
parallel_prank -i cpDNA_genes -o aligned_cpDNA_prank -j 8 --default
```

A log might begin:

```text
# Parallel PRANK: codon-length precheck
# Input FASTA: cpDNA_genes/rbcL.fasta
# Length rule: ignore '-' gaps and whitespace
# Records: 5 | Kept: 3 | Removed: 2
# Excluded sequences (length not divisible by 3):
# REMOVE  Taxon_A  length=1429  remainder=1
# REMOVE  Taxon_C  length=1430  remainder=2
# PRANK will use a temporary filtered FASTA; the original file is unchanged.
# --- PRANK output ---
... output from PRANK ...
```

Files with **no excluded sequences** use the original FASTA directly. If fewer than **two sequences remain** after checking, that alignment is **skipped** (PRANK is not run), the reason appears in its log, and the wrapper reports a failure/returns a nonzero status. Other alignments continue running.

If neither `-codon` nor `--default` is provided, this precheck is **completely disabled** and every FASTA is passed directly to PRANK, even when individual sequence lengths are not multiples of three. There is no new command-line option to enable validation separately; it is tied to the existing codon mode.

The divisibility check catches non-triplet lengths; it **does not verify biological reading frames**, stop codons, codon annotation, or nucleotide identities.

## Input and output layout

Supported extensions (case-insensitive): **`.fa`, `.fasta`, `.fas`, `.fna`, `.ffn`**. Only files immediately inside each input directory are scanned; subdirectories are not scanned.

```text
project/
├── parallel_prank.py
├── cpDNA_genes/
│   ├── rbcL.fasta
│   └── matK.fasta
└── mtDNA_genes/
    ├── cox1.fasta
    └── nad1.fasta
```

After running Example 2:

```text
project/
├── aligned_cpDNA_prank/
│   ├── rbcL.prank.log
│   ├── matK.prank.log
│   └── ... PRANK alignment outputs ...
└── aligned_mtDNA_prank/
    ├── cox1.prank.log
    ├── nad1.prank.log
    └── ... PRANK alignment outputs ...
```

Each output prefix uses the FASTA filename without its last extension (e.g. `rbcL.fasta` → `rbcL`). PRANK determines alignment-output suffixes and formats. Output directories are created automatically; logs contain both PRANK's standard output and errors. Reusing output directories can overwrite existing files.

Files producing the same output prefix are rejected before execution—for example, `gene.fa` and `gene.fasta` in the same input directory, or identical gene basenames from two input folders that share one output folder.

## Wrapper-owned vs. PRANK parameters

Regular PRANK accepts one file at a time via `-d=...` and one output prefix via `-o=...`. Parallel PRANK handles these internally for each FASTA, so:

- `-i` / `--input` **replaces** PRANK's input-file selection (`-d`).
- `-o` / `--output` supplies **destination directories**, not an individual PRANK output filename.
- PRANK's `-d`, `-d1`, `-d2` and `-o` controls cannot be passed through as additional PRANK arguments. PRANK's `-o` is reserved too because supplying a shared output prefix would cause parallel files to overwrite each other.
- Other PRANK options can be appended normally. For instance, `-once`, `-iterate=3`, and `-showtree` are preserved, even though the first two start with the same letters as wrapper options.

To inspect available PRANK parameters (which can vary with the installed version), use:

```bash
parallel_prank -h
```

This combines help from the wrapper and `prank -help`; the PRANK input/output controls managed by the wrapper are omitted. If PRANK is not installed, the wrapper prints its own help and explains why PRANK's help is unavailable.

## Errors and notes

- The script refuses empty folder names, mismatched input/output counts, missing input folders, unavailable PRANK, and `-j` values below 1.
- On Linux, it also rejects `-j` greater than the number of CPUs available under process affinity (unless `--no-pin` is set), and requires `taskset` for pinning.
- If there are no supported FASTA files, no alignment is started.
- Individual PRANK failures do not stop other jobs. Failed jobs are reported with their log file paths.
- In codon mode, every excluded record is reported before PRANK output, and the console prints a total number of excluded sequences across all jobs.
- If a codon-mode FASTA has fewer than two sequences remaining, that job is skipped and counted as failed; other jobs still run. Malformed FASTA files are reported in their logs.
- Exit status is **0** when all alignments succeed; it is nonzero for validation errors, missing inputs, no FASTA files, or alignment failures.
- For `--default` / `-codon`, supply coding DNA sequences appropriate for codon-aware alignment; the new length validation cannot detect internal frameshifts or incorrect CDS boundaries.

## Citation

If you publish alignments generated with the wrapper, please cite **PRANK** and its relevant methodological publication. Parallel PRANK is a convenience wrapper that schedules independent PRANK alignments; it does not change PRANK's alignment algorithm.

PRANK github: https://github.com/ariloytynoja/prank-msa
