# Parallel PRANK

**Parallel PRANK** is a lightweight Python wrapper for running independent [PRANK](https://ariloytynoja.github.io/prank-msa/) multiple-sequence alignments concurrently. Rather than supplying one FASTA file with PRANK's `-d` option, you supply **one or more folders containing FASTA files**. Each file is processed by a separate PRANK process, and `-j` limits how many processes run at once.

Useful for gene-by-gene alignments, including chloroplast and mitochondrial CDS datasets, without having to run hundreds of PRANK commands manually.

## Features

- **Folder-based inputs:** align every supported FASTA file in an input directory.
- **Multiple input/output pairs:** match comma-separated lists by position (`-i cpDNA,mtDNA -o cpDNA_prank,mtDNA_prank`).
- **Parallel execution:** choose the maximum number of simultaneous PRANK processes using `-j` / `--jobs`.
- **Native PRANK flags:** forward normal PRANK options (e.g. `-codon`, `-F`, `-once`, `-iterate=3`, `-showtree`) to each alignment.
- **Optional previous behavior:** `--default` adds `-codon -F`; these flags are **not** added without `--default` unless specified manually.
- **Per-file logs:** collect each alignment's PRANK stdout and stderr in `<name>.prank.log`.
- **Combined help:** `-h` prints wrapper options followed by the installed PRANK's help, excluding PRANK input/output controls managed by the wrapper.

## Requirements

- Python **3.8+** (standard library only).
- PRANK installed, with its executable named `prank` and accessible in your `PATH`.
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
parallel_prank -i INPUT_DIR[,INPUT_DIR...] -o OUTPUT_DIR[,OUTPUT_DIR...] -j N [--default] [PRANK options...]
```

| Wrapper parameter | Purpose | Required? |
| --- | --- | --- |
| `-i`, `--input` | One or more comma-separated folders containing FASTA files | Yes |
| `-o`, `--output` | Comma-separated destination folders, paired with inputs in the same order | Yes |
| `-j`, `--jobs` | Maximum number of **simultaneous PRANK processes** (positive integer) | Yes |
| `--default` | Automatically use `-codon -F` on every PRANK alignment | No |
| `-h`, `--help` | Show wrapper help followed by help from installed PRANK | No |

**All three arguments `-i`, `-o`, and `-j` are required.** There is no default job count or predefined input folder. When `--default` is absent, PRANK receives only the input/output locations and the PRANK options you explicitly supply.

### Example 1: One folder with original script's alignment options

```bash
parallel_prank -i cpDNA_genes -o aligned_cpDNA_prank -j 8 --default
```

This runs up to eight alignments simultaneously. For each FASTA, the underlying PRANK invocation is equivalent to:

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

A single, shared pool of **up to 12** PRANK processes is used across *both* datasets (not 12 per directory).

If the numbers of input and output directories differ, the script **aborts before starting any alignments**. For example, `-i mtDNA_genes,cpDNA_genes -o aligned_genes` is invalid.

### Example 3: PRANK's own options, no preset

```bash
parallel_prank -i genes -o aligned -j 6 -codon -F -once -showtree
```

Or use PRANK's non-codon alignment behavior without any additional flags:

```bash
parallel_prank -i genes -o aligned -j 6
```

PRANK flags are passed unchanged to every alignment. As with PRANK itself, options with values should use its `-option=value` syntax, such as `-iterate=3` or `-f=phylipi`.

### `-j` controls separate jobs, not threads within PRANK

For example, `-j 8` means **up to eight independent PRANK processes** running at the same time. The wrapper uses Python threads to start and supervise these processes; it does not parallelize an individual PRANK alignment. Pick a limit appropriate for your available CPU and RAM.

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
- If there are no supported FASTA files, no alignment is started.
- Individual PRANK failures do not stop other jobs. Failed jobs are reported with their log file paths.
- Exit status is **0** when all alignments succeed; it is nonzero for validation errors, missing inputs, no FASTA files, or alignment failures.
- For `--default` / `-codon`, supply coding DNA sequences appropriate for codon-aware alignment; improper reading frames may cause PRANK to fail.

## Citation

If you publish alignments generated with the wrapper, please cite **PRANK** and its relevant methodological publication. Parallel PRANK is a convenience wrapper that schedules independent PRANK alignments; it does not change PRANK's alignment algorithm.

PRANK documentation: https://ariloytynoja.github.io/prank-msa/
