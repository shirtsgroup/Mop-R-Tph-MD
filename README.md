# Mop-R-Tph-MD

Input files for the molecular dynamics (MD) and temperature replica-exchange MD (T-REMD) simulations reported in:

> **"Unimolecular Folding in Oligomers of a New ortho-Terphenyl Amino Acid"**

This repository contains the GROMACS input files used to study the conformational folding of MOP-terphenyl oligomers (a tetramer and an octamer) in explicit solvent. Two simulation sets are provided:

- **`tetramer-tremd/`** — T-REMD of a MOP terphenyl **tetramer** (64 replicas, 250–450 K).
- **`octamer-helix-md/`** — conventional MD of a MOP terphenyl **octamer** in a helical conformation (300 K, 100 ns).
- **`oct_helix_273K.mp4`** — animation of the octamer helix trajectory at 273 K (see [Animation](#animation) below).

---

## Repository layout

```
mop-R-Tph-MD/
├── README.md
├── oct_helix_273K.mp4
├── tetramer-tremd/
│   ├── remd_parameters.yml          # T-REMD setup (64 replicas, T range, mdps)
│   ├── em.mdp                       # Energy minimization (template)
│   ├── berendsen_nvt.mdp            # Berendsen NVT equilibration (template, TEMP placeholder)
│   ├── berendsen_npt.mdp            # Berendsen NPT equilibration (template, TEMP placeholder)
│   ├── production_npt.mdp           # Production NPT (template, TEMP placeholder)
│   ├── submit.berendsen_nvt.slurm   # Slurm: EM -> NVT equilibration
│   ├── submit.berendsen_npt.slurm   # Slurm: NVT -> NPT equilibration
│   ├── submit_remd.production.slurm # Slurm: T-REMD production
│   ├── submit_remd.continue.slurm   # Slurm: extend production run
│   ├── submit_remd.finish.slurm     # Slurm: finish production run
│   └── sim0/ ... sim63/             # One directory per replica (64 total)
│       ├── em_system.gro
│       ├── berendsen_nvt.{gro,mdp,tpr}
│       ├── berendsen_npt.{gro,mdp,tpr}
│       ├── production_npt.{gro,mdp,tpr}
│       ├── system_openff_hmr.top     # Topology (OpenFF + HMR)
│       └── mdout.mdp
└── octamer-helix-md/
    ├── em.mdp
    ├── em_system.gro
    ├── berendsen_nvt.{gro,mdp}
    ├── berendsen_npt.{gro,mdp}
    ├── npt_new.mdp                  # Production NPT (100 ns)
    ├── mop_octamer_system_openff.top        # Topology (OpenFF)
    ├── mop_octamer_system_openff_hmr.top    # Topology (OpenFF + HMR)
    └── mop_octamer_system_openff_hmr_rest.top
```

---

## Simulation details

### Tetramer T-REMD (`tetramer-tremd/`)

| Parameter | Value |
|---|---|
| System | MOP terphenyl tetramer in explicit solvent |
| Force field | OpenFF-2.0 (foldamer) + OpenFF-1.0 (TCM)  |
| Water model | TIP3P (explicit solvent) |
| Hydrogen mass repartitioning | Yes (`hmr_ratio = 3`, via ParmEd) |
| Integrator | `md` (leap-frog) |
| Time step | 4 fs (0.004 ps) |
| Thermostat (production) | v-rescale, τ = 2 ps |
| Barostat (production) | c-rescale, isotropic, 1 bar, τ = 5 ps |
| Constraints | H-bonds (LINCS, iter 2, order 8) |
| Electrostatics | PME, rcoulomb = 1.0 nm |
| van der Waals | cutoff + Potential-shift-Verlet, rvdw = 1.0 nm |
| Dispersion correction | EnerPres |
| Replicas | 64, temperatures 250–450 K |
| Equilibration | Energy minimization → Berendsen NVT (1.25 M steps ≈ 5 ns) → Berendsen NPT (1.25 M steps ≈ 5 ns) |
| Production | 25 M steps ≈ 100 ns per replica |
| Exchange | Every 100 steps (0.4 ps); `nex = 262144` |
| Output frequency | Every 50 000 steps (200 ps) |

Each replica directory (`sim0` … `sim63`) holds the pre-built starting structure, topology, and run input for one temperature. The top-level `*.mdp` files are templates in which the placeholder `TEMP` is replaced by the replica-specific temperature (see `remd_parameters.yml`).

### Octamer MD (`octamer-helix-md/`)

| Parameter | Value |
|---|---|
| System | MOP terphenyl octamer in helical conformation, explicit solvent |
| Force field | OpenFF |
| Temperature | 300 K (animation at 273 K in `oct_helix_273K.mp4`) |
| Integrator | `md` |
| Time step | 4 fs |
| Thermostat | v-rescale (production), Berendsen (equilibration) |
| Barostat | c-rescale (production), Berendsen (equilibration), 1 bar |
| Production length | 25 M steps ≈ 100 ns (`npt_new.mdp`) |
| Equilibration | Energy minimization → Berendsen NVT (5 ns) → Berendsen NPT (5 ns) |

---

## Animation

The octamer helix trajectory at 273 K is shown below.

<video src="oct_helix_273K.mp4" controls width="640"></video>

> If the video does not render inline, download it directly: [oct_helix_273K.mp4](oct_helix_273K.mp4)

---

## Software requirements

- **GROMACS** ≥ 2023 (tested with `gmx_mpi` from GROMACS 2023.5)
- **MPI** (OpenMPI 4.1.5) for the multi-replica runs
- **ParmEd** (for HMR topology generation)
- **OpenFF** force-field parameters (foldamer parametrization)
- A Slurm-based HPC cluster (scripts use `#SBATCH` directives)

---

## Running the simulations

### Tetramer T-REMD

From within `tetramer-tremd/`:

```bash
# 1. Energy minimization + Berendsen NVT equilibration (all 64 replicas)
sbatch submit.berendsen_nvt.slurm

# 2. Berendsen NPT equilibration
sbatch submit.berendsen_npt.slurm

# 3. T-REMD production
sbatch submit_remd.production.slurm

# 4. (Optional) extend the production run by ~4 ns per replica
sbatch submit_remd.continue.slurm

# 5. (Optional) finish / final segment
sbatch submit_remd.finish.slurm
```

The production command runs all replicas concurrently with replica exchange:

```bash
mpirun -np 128 gmx_mpi mdrun -v -multidir sim{0..63} \
  -deffnm production_npt -ntomp 1 -nex 262144 -replex 100
```

The SLURM headers for these bash scripts will need to be adjusted to your local cluster.

### Octamer MD

From within `octamer-helix-md/`:

```bash
# Energy minimization
gmx grompp -f em.mdp -c em_system.gro -p mop_octamer_system_openff_hmr_rest.top -o em_system -r em_system.gro
gmx mdrun -deffnm em

# NVT equilibration
gmx grompp -f berendsen_nvt.mdp -c em.gro -p mop_octamer_system_openff_hmr_rest.top -o berendsen_nvt -r em_system.gro
gmx mdrun -deffnm berendsen_nvt

# NPT equilibration
gmx grompp -f berendsen_npt.mdp -c berendsen_nvt.gro -p mop_octamer_system_openff_hmr_rest.top -o berendsen_npt -r em_system.gro
gmx mdrun -deffnm berendsen_npt

# Production (100 ns)
gmx grompp -f npt_new.mdp -c berendsen_npt.gro -p mop_octamer_system_openff_hmr.top -o npt_new
gmx mdrun -deffnm npt_new
```
Energy minimization and Berendsen NPT and NVT simulations are run with restraints on the helix structure with the inclusion of `-r em_system.gro` and `-p mop_octamer_system_openff_hmr_rest.top`. Production simulations remove these restraints so the helix can move freely and interact with solvent. To remove the restraints for these simulations remove the `-r` flag and change the topology file to `mop_octamer_system_openff_hmr.top` which does not include `posre.itp` in the topology file.

---

## Notes

- The `*.tpr` files are pre-generated GROMACS run input files; regenerate them with `gmx grompp` if you modify an `*.mdp` or topology.
- Temperatures in `sim*/` are spaced exponentially between 250 K and 450 K (see `remd_parameters.yml`).
- Topologies are self-contained (generated by ParmEd) and include HMR for a 4 fs time step.

## License

Input files provided for reproducibility of the published study. Please cite the associated paper when using these files.