---
name: conda-env-register
description: Register an existing local conda or venv installation into the current Claude Science session with manage_environments(mode="register") instead of copying, rebuilding, or reinstalling it. Use when the user asks to connect, expose, reuse, register, or add an existing conda environment or virtual environment such as geocal, xclim, jpylab, a project .venv, or any path with bin/python. Covers finding a suitable interpreter, verifying its absolute path, checking host grants, calling register with source_path and venv_path, and proving the resulting environment really launches that interpreter.
---

# Register an existing Python environment

The goal is to add a **reference** to an environment that already exists on the user's machine. The registration must not copy the environment, create a new venv, or install packages into it.

## When to use

- The user says an existing conda or virtual environment should be registered, connected, reused, exposed, or added.
- The user names an environment such as `geocal`, `xclim`, `jpylab`, `base`, or a project `.venv` and wants it available as `environment=<name>` in code cells.
- The user corrects you that an environment is already available locally and should be used in place.

This skill is not for creating a brand-new managed environment. When the user asks to install dependencies into a fresh isolated environment, use environment creation tools instead.

## Core facts

`manage_environments(mode="register")` accepts an existing interpreter directory. A conda environment is compatible because it also has `bin/python` on macOS/Linux.

A pure reference registration has these parameters:

- `mode="register"`
- `name=<session environment name>`
- `source_path=<parent directory that contains the environment>`
- `venv_path=<absolute path to the environment directory>`
- `create=false` (default; omit or pass explicitly)

The result should report `kind: "path-venv"` and `created_by_daemon: false`. When it does, no environment was copied or created.

## Workflow

1. Confirm the absolute interpreter directory. Common forms are:
   - conda: `/Users/<user>/miniconda3/envs/<env>/bin/python` or `/opt/conda/envs/<env>/bin/python`
   - project venv: `/path/to/project/.venv/bin/python`
   - Windows: `<env>\Scripts\python.exe`

   If the user gives only an environment name, inspect existing conda environments with `conda info --envs` or check the granted host path. Do not assume the path from training data; use a path that was observed in this conversation or provided by the user.

2. Ensure the parent directory is an authorized host grant. Use `list_host_grants()` and verify that `venv_path` lies under a granted path. If it does not, request host access for the parent directory before registration.

3. Register the environment with an explicit absolute path:

   ```
   manage_environments(
     mode="register",
     name="<name>",
     source_path="<parent directory>",
     venv_path="<absolute path to environment directory>",
     create=false
   )
   ```

   Use the environment directory, not the interpreter file, for `venv_path`. For example, `/Users/levin/miniconda3/envs/geocal`, not `.../bin/python`.

4. Verify the registration by launching a code cell with `environment="<name>"` and printing the interpreter path and version:

   ```
   python -c "import sys; print(sys.executable); print(sys.version)"
   ```

   For an R environment, use `Rscript -e 'cat(R.home(), "\n")'`.

5. Report the four concrete facts to the user:
   - registration succeeded
   - session environment name
   - absolute path to the actual interpreter
   - confirmation that it is a reference, not a copy or rebuild

## Sandbox boundary

With the sandbox enabled, a registered path-venv remains read-only to code. Installing packages directly into it will fail. `list` operations still work. If the user needs to add packages, create a managed environment instead or ask the user to install them outside the sandbox.

## Example: registering geocal

Observed setup:

- granted path: `/Users/levin/miniconda3`
- environment directory: `/Users/levin/miniconda3/envs/geocal`
- parent/source path: `/Users/levin/miniconda3/envs`
- session name: `geocal`

Registration call:

```
manage_environments(
  mode="register",
  name="geocal",
  source_path="/Users/levin/miniconda3/envs",
  venv_path="/Users/levin/miniconda3/envs/geocal",
  create=false
)
```

Verification output showed:

```
/Users/levin/miniconda3/envs/geocal/bin/python
3.11.18
```

That output confirmed the registered environment launches the exact existing interpreter.
