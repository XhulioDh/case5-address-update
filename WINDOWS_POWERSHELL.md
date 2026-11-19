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

## Live runs (these cost money)

Follow PROTOCOL.md. Limits built into `live_run.py`: $1.00 per run, 5
attempts per model (attempts where the model replied, completed or not) and
$4.00 per batch.

**1. Practice first, for free** (no key, no model, no cost):

```powershell
python live_run.py --fake
```

At each agent message, choose `1` or `2` (the frozen replies R1 and R2),
`u` (unscripted question: the run stops), `c` (completion) or `s` (stop), as
shown on screen. For the practice run: press `1`, then `c`.

**2. Install the Anthropic library (once):**

```powershell
python -m pip install anthropic
```

**3. Give PowerShell your API key for this window only.** Run these two
lines. The first one asks for the key and hides what you paste:

```powershell
$k = Read-Host "Paste your API key" -AsSecureString
```

```powershell
$env:ANTHROPIC_API_KEY = [System.Net.NetworkCredential]::new("", $k).Password
```

Never paste the key into a file in this folder, into a chat, or directly
into a command such as `$env:ANTHROPIC_API_KEY = "sk-..."`, because
PowerShell saves typed commands in its history. When you close the window,
the key is forgotten.

**3b. Free check that your key can use a model** (no tokens, no cost):

```powershell
python live_run.py --check-model --model claude-fable-5-2
```

It should print `OK: this key can see claude-fable-5-2`.

**4. Start a real run (follow-up batch `followup1`).** The frozen schedule
chooses the model and the portal condition; just run:

```powershell
python live_run.py
```

It shows `Start REAL attempt N of 30 with <model>?`. Type `yes` to confirm.
You play the citizen using only the frozen rules in PROTOCOL.md section 3;
press `c` only when the completion rule (section 4) fits. The script does not
show the condition. After the first two attempts, check the real cost in the
Anthropic Console before continuing (PROTOCOL_FOLLOWUP1.md section 4).

The finished baseline batch can no longer be run:
`python live_run.py --batch baseline1 --model claude-opus-5-5` refuses.

**5. Afterwards:** fill in a copy of `REVIEW_SHEET.md` for the run, and
check your real spend in the Anthropic Console (Billing).

To forget the key before closing the window:

```powershell
Remove-Item Env:ANTHROPIC_API_KEY
```
