```markdown
# gptfakeQE.py README

This script generates **fake Quantum ESPRESSO (QE) input/output files** (`*.in`, `*.sout`) from a **LAMMPS atomic data file** (`*.data`) by:

1. Reading the input LAMMPS structure (`atom_style atomic`).
2. Running two-stage optimization (OPT-1, OPT-2) using an ML calculator (FairChem).
3. Optionally building a supercell to satisfy a minimum cell-length requirement.
4. Running MD (NPT or NVT) to produce a QE-like `*.sout` log.
5. Writing supporting files required by your existing Perl pipeline (e.g. `elements.dat`, and LAMMPS `.data` snapshots).

This is designed to **mimic QE `vc-md` / `scf` output format** so your legacy Perl scripts (e.g. `DFTout2npy_QE.pl` or similar) can parse it.

---

## 1. Output files and where they go

Assume your input file is:

```

/path/to/Al2Co2Cr2Fe2Mo2Nb7Ni7Ta2Ti2W4_fcc_bulk-T300-P0.data

```

The script will create:

### In the SAME directory as the input `.data`:
- `Al2Co2Cr2Fe2Mo2Nb7Ni7Ta2Ti2W4_fcc_bulk-T300-P0.in`
- `Al2Co2Cr2Fe2Mo2Nb7Ni7Ta2Ti2W4_fcc_bulk-T300-P0.sout`
- `elements.dat`

### In a local folder:
- `data_files/000.data ...`
- `data_files/minimized_1.data`, `data_files/minimized_2.data`
- (if supercell is applied) `data_files/supercell.data`, `data_files/minimized_2_supercell.data`

`data_files/000.data` is the **relaxed structure used as the initial structure for MD** and corresponds to the structure whose energy/force/stress are printed at the beginning of the QE `sout` (Iteration=1).

---

## 2. Requirements

### Input file requirements
Your LAMMPS data file must include a `Masses` block with element symbols:

Example:

```

Masses

1 26.9815385  # Al
2 58.933194   # Co
...

```

This is mandatory because the script uses the element symbols to map types → atomic numbers.

### Python environment
- ASE
- fairchem-core
- PyTorch

---

## 3. Command line usage (fixed argument list)

The script requires **exactly 15 arguments** after the script name:

```

python gptfakeQE.py 
<input_data_file> 
<opt1_steps> <opt1_fmax> 
<opt2_steps> <opt2_fmax> 
<npt_steps> <do_supercell 0/1> 
<temp_K> <press_GPa> <timestep_fs>  <cx> <cy> <cz> 
<MODEL_NAME> <TASK_NAME>

````

---

## 4. Meaning of arguments

### (A) Optimization parameters
- `<opt1_steps>`: steps for OPT-1
- `<opt1_fmax>`: fmax threshold for OPT-1 (eV/Å)
- `<opt2_steps>`: steps for OPT-2
- `<opt2_fmax>`: fmax threshold for OPT-2 (eV/Å)

**OPT-1**
- Fixes all atoms and relaxes **cell lengths only** (and angles fixed).
- Controlled by `<cx cy cz>` which selects which lengths are allowed to change.

**OPT-2**
- Relaxes atoms + selected cell lengths, angles fixed.
- Also controlled by `<cx cy cz>`.

### (B) MD parameters
- `<npt_steps>`: number of MD steps
  - If `0` → the fake QE input calculation becomes `scf`, and **no MD is run**.
- `<temp_K>`: temperature in Kelvin (only used if MD runs)
- `<press_GPa>`: target pressure in GPa (used by OPT-1, OPT-2 and NPT; ignored in NVT)
- `<timestep_fs>`: MD timestep in fs
  - The fake QE input uses:
    - `dt = timestep_fs * 20`
    - `nstep = npt_steps`

### (C) Supercell toggle
- `<do_supercell>`: 0 or 1
  - If `1`: after OPT-2, the script checks cell lengths.
  - Any dimension with length `< 12 Å` is repeated until it reaches `≥ 12 Å`.
  - OPT-2 is then run **again** on the expanded supercell.
  - If all dimensions already satisfy `≥ 12 Å`, the second OPT-2 is skipped.

### (D) Cell-length control flags: `<cx cy cz>`
Each is `0` or `1`, selecting which cell lengths are allowed to change in OPT and in MD.

- `cx=1` allows the **a** length to change.
- `cy=1` allows the **b** length to change.
- `cz=1` allows the **c** length to change.

Special behavior:
- If `cx=cy=cz=0` AND `npt_steps>0` → **MD switches from NPT to NVT** (fixed cell).

### (E) Model selection
- `<MODEL_NAME>`: FairChem model name, e.g. `uma-s-1p1`
- `<TASK_NAME>`: task name, e.g. `omat`

---

## 5. Recommended argument settings for common scenarios

### 5.1 Fake QE **SCF only** (no MD)
Use `npt_steps = 0`.

Example:
```bash
python gptfakeQE.py INPUT.data \
  5 0.2  5 0.1 \
  0 0 \
  300.0 0.0 1.0 \
  1 1 1 \
  uma-s-1p1 omat
````

Notes:

* `do_supercell` still required (set to 0 or 1).
* `temp_K`, `press_GPa`, `timestep_fs` are still required but are not used for MD because MD does not run.
* Output:

  * `*.in` will have `calculation="scf"` and will not include `nstep/dt`.
  * `*.sout` prints one “Iteration=1” block and ends.

---

### 5.2 Bulk 3D **NPT (isotropic)** (a,b,c all allowed)

Use `cx cy cz = 1 1 1`.

Example:

```bash
python gptfakeQE.py INPUT.data \
  200 0.2  200 0.1 \
  50 1 \
  300.0 1.0 1.0 \
  1 1 1 \
  uma-s-1p1 omat
```

* `press_GPa` is the external target pressure.
* MD runs NPT for `npt_steps=50`.
* QE input:

  * `calculation="vc-md"`
  * `nstep=50`
  * `dt=timestep_fs*20`

---

### 5.3 2D slab with vacuum in **z**: relax x,y only (keep vacuum fixed)

Typical for structures with vacuum along `c` (z).

Use:

* `cx cy cz = 1 1 0`

Example:

```bash
python gptfakeQE.py INPUT.data \
  200 0.2  200 0.1 \
  50 1 \
  300.0 0.0 1.0 \
  1 1 0 \
  uma-s-1p1 omat
```

Interpretation:

* a and b can change; c stays fixed.
* This avoids shrinking or expanding the vacuum thickness.
* This applies to both optimization stages and NPT MD (barostat only in x/y).

Recommendation:

* For 2D slabs, often use `press_GPa=0.0` (or a small target pressure).

---

### 5.4 **NVT** MD (fixed cell, only atoms move)

Two ways:

#### Option A (recommended): `cx cy cz = 0 0 0`

This forces MD mode to NVT automatically.

```bash
python gptfakeQE.py INPUT.data \
  0 0.0  200 0.05 \
  100 0 \
  300.0 0.0 1.0 \
  0 0 0 \
  uma-s-1p1 omat
```

Notes:

* `press_GPa` is ignored in NVT.
* Cell is constant.
* OPT-1 is skipped automatically because no cell DOF exists.

#### Option B: keep some cell DOF but choose `npt_steps=0`

That is not NVT; it becomes SCF only.
If you want MD with fixed cell, use Option A.

---

## 6. How to choose `opt1/opt2` values

Suggested starting points:

### Fast / rough

* opt1_steps = 20, opt1_fmax = 0.2
* opt2_steps = 50, opt2_fmax = 0.1

### More stable

* opt1_steps = 200, opt1_fmax = 0.1
* opt2_steps = 200, opt2_fmax = 0.05

Important:

* If your aim is to reduce pressure quickly, opt-1 (cell-only) is effective when atoms are fixed.
* opt-2 then reduces forces and (for allowed directions) further relaxes cell lengths.

---

## 7. QE-like output behavior (important for your Perl parser)

* The `.sout` prints an initial energy/force/stress block at **Iteration=1** for the starting structure (`000.data`).
* It prints `! total energy` blocks exactly **nstep times** (step 0 to step nstep-1).
* The **final structure** at step `nstep` is printed with `CELL_PARAMETERS` and `ATOMIC_POSITIONS`, but **no** `! total energy` block is printed for that last structure.

This matches the rule:

* `nstep=50` → exactly 50 energy blocks
* `050.data` exists and prints geometry, but no SCF properties for it

---

## 8. Typical mistakes and how to fix them

### Error: “Masses block missing valid '# Element'”

Fix your LAMMPS data file `Masses` block to include element symbols after `#`.

### Too slow or too few cores used

If running under Slurm:

* use `--exclusive` and set `OMP_NUM_THREADS=$SLURM_CPUS_ON_NODE`
* your script already tries to sync PyTorch threads to Slurm/OMP variables

---

## 9. Quick reference table: which mode do I want?

| Goal                  | npt_steps | cx cy cz | Behavior                        |
| --------------------- | --------: | -------- | ------------------------------- |
| SCF only              |         0 | any      | calculation=scf, no MD          |
| 3D NPT                |        >0 | 1 1 1    | NPT in x,y,z                    |
| 2D slab + vacuum in z |        >0 | 1 1 0    | NPT/relax in x,y only           |
| NVT (fixed cell)      |        >0 | 0 0 0    | NVT (cell fixed), press ignored |

---

## 10. Example commands (copy/paste)

### SCF

```bash
python gptfakeQE.py A.data 5 0.2 5 0.1 0 0 300 0.0 1.0 1 1 1 uma-s-1p1 omat
```

### 2D slab NPT in x,y only

```bash
python gptfakeQE.py slab.data 200 0.2 200 0.1 50 1 300 0.0 1.0 1 1 0 uma-s-1p1 omat
```

### NVT

```bash
python gptfakeQE.py bulk.data 0 0.0 200 0.05 100 0 300 0.0 1.0 0 0 0 uma-s-1p1 omat
```

---
