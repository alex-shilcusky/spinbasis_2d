# 2D Spin Basis

Variational Monte Carlo for the 2D J1-J2 Heisenberg model on the periodic
square lattice, in the plain sigma^z spin basis, with a choice of wavefunction:
Jastrow, RBM (optionally translation- or space-group-symmetric), or the Vision
Transformer from the NetKet tutorial.

Companion to `../2d_eigenbasis`: the same config layout, driver (overdispersed
`VMC_NG` with optional adaptive alpha), Slurm wrappers, checkpoint/resubmit
sentinels and output file formats, so `plot_energy_vs_iteration.ipynb` can
read sweeps from both projects. Energies are logged in S = sigma/2 units in both.

## Layout

```
spinbasis_2d/
├── run_2D_spinbasis.py    entry point: builds H, model, driver; runs VMC
├── config.toml            base configuration + parameter sweeps
├── submit.sh              login node: validate, expand sweep, sbatch
├── run.sh                 per-task Slurm script + auto-resubmit
└── helpers/
    ├── wavefunctions.py   [model] config -> Flax module (jastrow | rbm | vit)
    └── vit.py             the tutorial's spin-basis ViT
```

Run output goes to `spinbasis_data_2d/`, a **sister** of this repo
(`master/spinbasis_data_2d/`), so data never touches git.

Each submission writes `spinbasis_data_2d/<timestamp>G<commit>P<pid>/run_XXXX/`.
The folder is created on first submit. Override with
`RUN_ROOT=/some/existing/path ./submit.sh config.toml`.

The plotting notebook looks for a folder named `2D_eigenbasis_data`, so to plot these
runs set its `DATA_ROOT` to `master/spinbasis_data_2d` instead.

## Setup (once, on the cluster)

`submit.sh` requires a clean git repository and `run.sh` requires `.venv`:

```bash
cd spinbasis_2d
git init && git add -A && git commit -m "spin-basis VMC"
uv sync                 # uv.lock pins the same versions as 2d_eigenbasis
./submit.sh config.toml
```

## Choosing a wavefunction

Set `[model] type` and edit that type's table (`[model.jastrow]`,
`[model.rbm]`, `[model.vit]`). All three tables stay in the file, so a sweep
over `model.type` needs no other edits. See the sweep examples at the bottom
of `config.toml`.

Adding a wavefunction is one branch in `helpers/wavefunctions.py:build_model`
plus a `[model.<name>]` table.

## Differences from 2d_eigenbasis worth knowing

- **Checkpoints**: `run.mpack` and `run.progress` are written together, atomically,
  every `SAVE_EVERY` steps. A hard kill or `scancel` loses at most `SAVE_EVERY`
  steps, and resubmitting the array task resumes correctly with no manual repair.
- **Divergence** (`InvalidLossStopping`) never overwrites the checkpoint.
- **Log files** are `run step 000000.log`, `run step 004400.log`, ...: zero-padded,
  so sorting by name sorts by time.
