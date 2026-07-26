# Pangenome-Refgraph-workshop
### A hands-on tutorial: construction and visualization only

## What this tutorial covers

You will take a small set of linear genome sequences, build them into a single
pangenome graph, inspect the graph's structure directly (nodes, edges, and the
"bubbles" where haplotypes disagree), and visualize the result with **sequenceTubeMap**.

This is **construction only**. We are not aligning short reads, calling genotypes,
or producing a VCF against a designated reference. The variants you'll see are simply
the graph's own topology — places where the input haplotypes' sequences diverge and
reconverge.

**Tools used**
| Tool | Role |
|---|---|
| `pggb` (PanGenome Graph Builder) | builds the graph directly from a set of haplotype FASTAs |
| `odgi` | inspect / summarize / visualize the graph (comes bundled with pggb) |
| `vg` | convert the graph into the index formats sequenceTubeMap reads |
| `sequenceTubeMap` | interactive tube-map visualization in the browser |

---
## Step 0 — Install WSL on windows:
Follow these steps to enable WSL and install Ubuntu:

### 1.1 Enable Windows Features and WSL
In the search box, search for `Turn WIndows features on or off`

<img width="1071" height="1044" alt="image" src="https://github.com/user-attachments/assets/faa2a8a0-a58e-4b7a-953c-7be2df3fd9df" />

In the pop-up menu, enable the following:
   * `Virtual Machine Platform`
   * `Windows Subsystem For Linux`

<img width="520" height="464" alt="image" src="https://github.com/user-attachments/assets/6df25474-cb36-41f8-bf4e-ca171c63dd84" />

Click OK, and restart when he asks to restart your PC. After restarting, type in the search box again `command prompt` and open it.
In the command prompt terminal, type `wsl --status`, you will see that WSL has been installed successfully

<img width="1483" height="762" alt="image" src="https://github.com/user-attachments/assets/8e799e1f-b7fc-4740-b19f-6dbad08ea0e9" />

Now you have to update WSL to get the latest version before we install Ubuntu. 

In the **Command Prompt Terminal** run:

```powershell
wsl --update
```

<img width="1483" height="762" alt="image" src="https://github.com/user-attachments/assets/3d7bc981-4f5c-4bb1-990e-425c74d6d3a5" />

After installation, go to `Microsoft Store` and search for `Ubuntu` and Get it!!

<img width="1920" height="1140" alt="image" src="https://github.com/user-attachments/assets/7f88bf3b-f2e9-4dbd-9a3a-524bd7c055e6" />

### 1.2 Launch Ubuntu

Once installed, open it from and set your username and password. Don't worry if you didn't see anything typed during the password step, it's a hidden process.

<img width="1483" height="762" alt="image" src="https://github.com/user-attachments/assets/fcbaf3f6-50cd-41bc-8e66-542cb21a1f47" />

#### Congratulations!!! Now you have Ubuntu Installed as subsystem on Windows!!
you can open Ubuntu Terminal from windows menu

---

## Step 1 — Install Miniconda & Configure Conda Channels

We use **Miniconda** to manage bioinformatics tools.

### 1.1 Download & install Miniconda (source: conda website)

```bash
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh
```
After installing, close and reopen your terminal application or refresh it by running the following command:
```bash
source ~/miniconda3/bin/activate
```
Then, initialize conda on all available shells by running the following command:

```bash
conda init --all
```
### 1.2 Configure Channels (including BioConda)

```bash
conda config --add channels defaults
conda config --add channels conda-forge
conda config --add channels bioconda
conda config --set channel_priority strict
```



## Step 2 — Ceate and activate Conda environemt 

```bash
conda create -n refGraph
conda activate refGraph
```
## Step 3 — Install Required tools and libraries 

### `PGGP`
```bash
conda install bioconda::pggb
```

### `vg`
```bash
conda install bioconda::vg
```

### `samtools`
```bash
conda install bioconda::samtools
```

### `Yarn`
```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list

sudo apt update
sudo apt install yarn
```

### `sequenceTubeMap`

```bash
git clone https://github.com/vgteam/sequenceTubeMap.git
cd sequenceTubeMap
npm install
```
---

## Step 4 — Prepare your input data

`pggb` wants **one multi-FASTA file** containing all the genomes/haplotypes you want in the
graph, each with a unique sequence name.
You can go to NCBI and Install some genomes for any species. and save them in `.fa` format

In this seesion, we we'll work on MTB:
https://www.ncbi.nlm.nih.gov/datasets/genome/?taxon=1773


you can download 10 references from the link, Combine them in `haplotypes.fa` file, and index them:

```bash
bgzip haplotypes.fa
samtools faidx haplotypes.fa.gz
```

---

## Step 5 — Build the graph with pggb

The three parameters you must set:

| Flag | Meaning |
|---|---|
| `-n` | number of genomes/haplotypes (skip if you used PanSN-spec naming) |
| `-s` | segment length used for the initial all-vs-all mapping |
| `-p` | minimum percent identity accepted between segments |

A minimal run for a small, closely related set of sequences:

```bash
pggb -i haplotypes.fa.gz -n 4 -s 5000 -p 90 -t 8 -o out -M
```

- `-t` = number of threads
- `-M` = also emit a multiple sequence alignment (MAF) alongside the graph — handy
  for comparing directly against the graph output while teaching
- `-o` = output directory

What happens under the hood:

1. **wfmash** aligns every haplotype/genome against every other haplotype/genome (all-vs-all —
   there is no reference).
2. **seqwish** induces a variation graph from those pairwise alignments.
3. **smoothxg** locally re-aligns and "smooths" the graph so that regions which are
   linear in reality stay linear in the graph, rather than being fragmented into
   lots of tiny nodes.
4. **odgi** and **gfaffix** normalize and finalize the graph.

The main output is `out/*.gfa` — your pangenome graph.

---

## Step 6 — Inspect the graph structure directly

Open the GFA in a text editor — it's just three kinds of lines, which map exactly
onto the three signals we discussed earlier:

| GFA line | Graph concept | Example |
|---|---|---|
| `S` (segment) | **Node** — a piece of shared or private sequence | `S 14 ACGTACGT` |
| `L` (link) | **Edge** — a connection between two nodes | `L 14 + 15 + 0M` |
| `P` (path) | **Path** — one haplotype's full route through the nodes | `P sampleA#1#region 14+,15+,17+ *` |

A quick sanity check with `odgi`:

```bash
odgi build -g out/*.gfa -o graph.og
odgi stats -i graph.og -S
```

This reports the number of nodes, edges, and paths — a fast way to confirm the
graph has one path per haplotype and to see how much the sequence collapsed.

For a quick static "bandage-style" picture:

```bash
odgi viz -i graph.og -o graph.png -x 1500 -y 500
```

This draws each path as a colored horizontal band across the nodes — exactly the
same visual grammar as the tube map, just non-interactive. It's a good first look
before moving to the interactive version.

---

## Step 7 — Convert the graph for sequenceTubeMap

sequenceTubeMap reads `vg`'s own index formats: an **XG** index (the graph itself)
and a **GBWT** index (the haplotype paths, so it can draw a colored track per
haplotype rather than one flat consensus line).

```bash
# Graph -> XG
vg convert -x -g out/*.gfa > graph.xg

# Haplotype paths -> GBWT (built from the paths already embedded in the graph)
vg gbwt -E -x graph.xg -o graph.gbwt
```

- `-E` tells `vg gbwt` to build the index from the paths that are already embedded
  in the graph (our haplotypes from Step 1) — again, no VCF or read alignment
  involved.

---

## Step 8 — Visualize

**Option A — smallest friction (recommended for teaching):**
Go to the sequenceTubeMap project's online demo and use the **"Use Custom Data"**
tab to upload your `graph.xg` / `graph.gbwt` (or a small `.vg`/JSON export) directly
in the browser. No server setup required — good for toy graphs built in class.

**Option B — local server (for bigger graphs):**

```bash
cd sequenceTubeMap
mkdir -p exampleData/mygraph
cp /path/to/graph.xg /path/to/graph.gbwt exampleData/mygraph/
```

Edit `src/config.json` so `dataPath` points at that folder:
Just add this section inside the `"DATA_SOURCES"` in the `src/config.json` 
```
{
      "name": "MTB pangenome",
      "tracks": [
        {"trackFile": "exampleData/mygraph/graph.xg", "trackType": "graph"}
      ],
      "dataType": "built-in",
      "region": "NC_000962.3:1-5000"
    }
```

then:

```bash
npm start
```

Open `http://localhost:3000`, select your dataset from the dropdown, search the
region, and render.

---
