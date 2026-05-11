# Modifications to aiida-trains-pot

## 1. `src/aiida_trains_pot/aiida_trains_pot_workflow/exploration_wc.py`

**Problem:** `KeyError: 'potential'` when running with `md_protocol=None` (no vdw_d2).

**Change:** Added a guard before assigning `pair_coeff_list` so the `"potential"` key is
created if not already present in `input_parameters`.

**Location:** around line 229 (inside `run_md`, after the vdw_d2 protocol block)

**Before:**
```python
input_parameters["potential"]["pair_coeff_list"] = pair_coeffs
```

**After:**
```python
if "potential" not in input_parameters:
    input_parameters["potential"] = {}
input_parameters["potential"]["pair_coeff_list"] = pair_coeffs
```

---

## 2. `src/aiida_trains_pot/evaluation/__init__.py`  ← NEW FILE (empty)

**Problem:** `ModuleNotFoundError: No module named 'aiida_trains_pot.evaluation.parsers'`
when the committee evaluation calculation tries to load the `trains_pot.evaluation` entry point.

**Fix:** Created an empty `__init__.py` in the `evaluation/` directory so Python treats it
as a proper package (previously the directory had no `__init__.py`).

**File path:**
```
src/aiida_trains_pot/evaluation/__init__.py
```

**Content:** empty file

---

## After each modification: reinstall and restart daemon

```bash
pip install -e /home/fioren_a/useful_repos/aiida-trains-pot
verdi daemon restart
```

---

## 3. `local_examples/graphene/no_augm_run_atp.py`

### 3a. Added `"potential"` key to `builder.exploration.parameters`

**Problem:** Without a `"potential"` key in the parameters dict, the workflow crashed
with `KeyError: 'potential'` (related to fix #1 above, belt-and-suspenders).

**Change:**
```python
builder.exploration.parameters = Dict(
    {
        "control": {"timestep": timestep},
        "potential": {
            "potential_style_options": "mace no_domain_decomposition",
        },
    }
)
```

### 3b. Added `prepend_text` to LAMMPS metadata to patch `input.in` for mliap

**Problem:** aiida-lammps generates a LAMMPS input with `pair_style mace no_domain_decomposition`
and `pair_coeff potential.dat C`, which is not compatible with the mliap/Kokkos build.

**Required format:**
```
pair_style mliap unified potential.pt 0
pair_coeff * * C
```

**Fix:** Added a `prepend_text` that patches the input file before LAMMPS runs:
```python
builder.exploration.md.lammps.metadata.options.prepend_text = (
    "cp potential.dat potential.pt\n"
    "sed -i 's|^[[:space:]]*pair_style[[:space:]].*|pair_style mliap unified potential.pt 0|' input.in\n"
    "sed -i -e 's/pair_coeff potential.dat /pair_coeff * * /g' input.in\n"
)
```

- `cp potential.dat potential.pt` — mliap requires a `.pt` file extension
- first `sed` — replaces the entire `pair_style` line
- second `sed` — fixes `pair_coeff` from `potential.dat` form to `* *` wildcard form

