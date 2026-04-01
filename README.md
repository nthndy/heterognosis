# heterognosis

A continuation of the [macrohet](https://github.com/nthndy/macrohet) project. **heterognosis** aims to understand the molecular underpinnings of Mtb-infected macrophage heterogeneity by employing 4i — iterative indirect immunofluorescence imaging — in combination with live-cell timelapse tracking. Single cells are tracked across a live-cell imaging window, then linked to multiplexed protein abundance measurements from the same cells in fixed tissue, enabling single-cell resolution of infection outcome alongside proteomic state.

---

## Repository structure

```
heterognosis/
│
├── data_organisation/          # Data transfer and zarr preparation
│   ├── 00_transfer_nemo_to_local.ipynb
│   ├── 01_zarr_reorganisation.ipynb
│   └── 02_adding_metadata_to_zarr.ipynb
│
├── notebooks/                  # Core analysis pipeline
│   ├── 01_tiff_to_zarr_chunked.ipynb
│   ├── 02_live_cell_segmentation.ipynb
│   ├── 03_fixed_cell_segmentation.ipynb
│   ├── 04_multiplex_stacking.ipynb
│   ├── 05_linking.ipynb
│   └── 06_zarr_io_qc.ipynb
│
├── LICENSE
└── README.md
```

---

## Environment

All notebooks run under the `macrohet` conda environment. To install:

```bash
mamba env create -f environment.yml
conda activate macrohet
```

Key dependencies: `cellpose`, `trackastra`, `zarr`, `dask`, `napari`, `macrohet`, `scikit-image`, `tqdm`, `natsort`, `numcodecs`.

---

## Data organisation notebooks

### `00_transfer_nemo_to_local`
Transfers raw 4i acquisition data from the Crick SMB share (`data2.thecrick.org`) to local storage (`/mnt/DATA3/BPP0050`). Uses a recursive `os.walk` sync that skips files already present and identical via `filecmp`, with all errors logged to `file_transfer_log.txt`. A second section handles a flat copy of the Live-2 `Images/` directory that was transferred separately via a direct OPERA mount.

### `01_zarr_reorganisation`
Reorganises the multiplexed fixed-cycle zarr stores produced by `04_multiplex_stacking` into an OME-NGFF compliant structure. Moves each `*_plexed.zarr` into a unified `multiplexed.zarr` with per-FOV subdirectories (`<row>_<col>/images/`), then reads Harmony acquisition metadata and assay layout XML to write OME-NGFF `.zattrs` per FOV containing physical pixel spacing, per-channel metadata (wavelengths, exposure), and experimental conditions (infection status, antibiotic treatment).

### `02_adding_metadata_to_zarr`
Constructs and writes `omero` and `multiscales` OME-NGFF metadata blocks for the live-cell zarr acquisitions (Live-1). Metadata includes: acquisition framerate derived from absolute timestamps, per-channel emission/excitation wavelengths, objective magnification and numerical aperture, pixel resolution (X, Y), and Z-plane spacing estimated from the modal inter-plane distance. Metadata is written to a top-level `OME.zarr` group and the assay layout is appended to the zarr root `.zattrs`.

---

## Analysis pipeline notebooks

Notebooks are numbered in execution order. The pipeline runs: tile → segment (live) → segment (fixed) → stack multiplexed → link.

### `01_tiff_to_zarr_chunked`
Reads Opera Phenix `.tiff` output and tiles multi-position acquisitions into OME-Zarr mosaics. A feathered alpha mask (`create_alpha_mask`) is applied at tile boundaries to blend overlapping tiles without hard edges. Iterates over T/C/Z for each position, assembling a (T, C, Z, Y, X) array. DPC (Phase Contrast) images are distributed uniformly across Z slices. OMERO and multiscales metadata are written to the zarr root after tiling. A chunked variant (section 3) handles the Live-2 acquisition, which was too large for in-memory batch processing — it pre-allocates the full zarr array and writes one timepoint at a time directly to disk.

### `02_live_cell_segmentation`
Segments macrophage cytoplasm from live-cell zarr acquisitions (Live-1, Live-2) using **Cellpose**, with BF and DPC channels as input. Tracks cells across time with **Trackastra** (CTC pretrained model). Segmentation labels are written back to each position's zarr store under `labels/trackastra_labels/`. Track tables (with per-object properties) are exported as CSV per position for downstream analysis.

### `03_fixed_cell_segmentation`
Segments and tracks cells across all fixed-cycle zarr acquisitions (Cy1–Cy10). The Cy1 cycle is processed first as the registration anchor. All other cycles follow the same pipeline: BF/DPC Z-max projection → Cellpose segmentation → Trackastra tracking → write labels to zarr. An image dimensionality audit runs first to confirm shape consistency across all positions and cycles before segmentation begins.

### `04_multiplex_stacking`
Collects all Fixed Cy2–Cy10 zarr image paths grouped by position, stacks them along a new staining-round axis to produce `(S, T, C, Z, Y, X)` arrays, and writes each position out as `<pos>_plexed.zarr`. Cross-cycle registration is then computed for a QC position: an Otsu-masked DAPI channel (Cy1 reference) is centre-cropped and registered against each cycle via phase cross-correlation, producing per-cycle (dy, dx) shifts. Aligned cycles are visualised in napari with a scale bar, physical pixel scaling, and a cycle counter overlay.

### `05_linking`
The most complex notebook — integrates all data modalities into a single unified spatiotemporal array. Loads Live-1, Live-2, the Cy1 intermediate fixed frame, and the multiplexed fixed images; concatenates along the time axis; Z-max projects; and pads the live channels to match the fixed channel count. Segmentation is extended across the fixed frames by propagating the final live mask. Drift correction is applied in two stages: live-to-live transitions are corrected via manual fiducial annotation in napari, and fixed-cycle frames are corrected via Cellpose-guided DAPI phase cross-correlation. A unified shift array is applied to images, segmentation labels, and track coordinates. Shift magnitudes are plotted per frame for QC.

> **Known issue:** a single large jump ~3 frames into the fixed alignment needs reconciling with the live-cell alignment.

### `06_zarr_io_qc`
Quick-look notebook for spot-checking a single zarr position. Loads images, segmentation, and the Trackastra track table, and launches a combined napari viewer with images, labels, and tracks overlaid. Configurable via `zarr_path`, `pos`, and `N_PREVIEW` at the top of the notebook. Use after any pipeline step to verify output integrity before proceeding.

---

## Data

Raw data are stored at `/mnt/DATA3/BPP0050` and on the Crick SMB share. Data are not included in this repository. Experiment identifier: **BPP0050**.

---

## Related

- [macrohet](https://github.com/nthndy/macrohet) — upstream project: live-cell segmentation, tracking, and heterogeneity quantification in iPSDMs
