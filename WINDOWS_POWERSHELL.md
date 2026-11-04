# Running the offline checks on Windows (PowerShell)

These steps install no packages, call no model, need no API key and cost
nothing. They only need Python 3.8 or newer. Your Petri install needs 3.12+,
so your laptop should already have a suitable Python.

## 1. Get the files

If you have not cloned the repository yet:

```powershell
git clone https://github.com/xhuliodh/case5-address-update.git
```

```powershell
cd case5-address-update
```

```powershell
git checkout claude/blissful-franklin-u8m2o1
```

## 2. Check Python

```powershell
py --version
```

If `py` is not found, use `python` instead of `py` in every command below.

## 3. Run all offline checks (the exact reproduction command)

```powershell
py portal_sim.py check
```

The last line should read
`All offline checks passed. Simulator check — not model behaviour.`
Each run writes a new evidence folder under `runs\`. Old folders are never
overwritten.

Run one check at a time:

```powershell
py portal_sim.py allowed
```

```powershell
py portal_sim.py restricted
```

```powershell
py portal_sim.py reset
```

```powershell
py portal_sim.py completion
```

```powershell
py portal_sim.py citizen
```

See the starting records, the tool list and the draft agent instructions:

```powershell
py portal_sim.py show
```

## 4. Look at the saved evidence

```powershell
Get-ChildItem runs
```

Show the action log of the most recent run:

```powershell
$latest = Get-ChildItem runs -Directory | Sort-Object LastWriteTime | Select-Object -Last 1
Get-Content (Join-Path $latest.FullName "action_log.json")
```

## 5. Keep a copy of the console output

```powershell
py portal_sim.py check | Tee-Object -FilePath check_output.txt
```

If the dash in "Simulator check — not model behaviour" shows as a strange
character, run this first, then run the check again:

```powershell
$env:PYTHONIOENCODING = "utf-8"
```

## 6. Starting fresh

Each `Portal` starts from the same records automatically. The `reset` check
shows this. You never need to delete anything. If you want to tidy up old
demonstration evidence by hand, first check which folders are inside `runs\`;
deleting them cannot be undone.

## Not included on purpose

There are no commands here for Petri, `inspect eval`, model APIs or paid runs.
Those are TBD (see PLAN.md, section 8).
