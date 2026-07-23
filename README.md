# HyperExecute JMeter End-to-End Automation

A Python script ([`hyperexecute_automation.py`](hyperexecute_automation.py)) that automates the LambdaTest HyperExecute JMeter flow through **two commands**:

- **`create`** – run **once** per project. Creates the project, uploads **all** files (the ones you pass to `--files`), and **prints the generated project ID**. No test run.
- **`trigger`** – run **as often as you like**. Executes a test on an existing `--project-id` (**no re-upload**) and holds all the run parameters you tune between runs — regions, duration, ramp-up, max vusers, etc.

Full pipeline covered: create project → upload files → (then) trigger job → monitor job → monitor artifacts → download results zip. Every parameter is supplied at runtime via CLI flags or environment variables.

---

## Requirements

- Python 3.7+
- The `requests` library:

```bash
pip3 install -r requirements.txt
# or
pip3 install requests
```

---

## Credentials

Get these from your LambdaTest **Account Settings > Password & Security**:

- **userName** – your LambdaTest username
- **accessKey** – your LambdaTest access key

Pass them via flags (`--userName`, `--accessKey`) or environment variables:

```bash
export LAMBDATEST_USERNAME=your_username
export LAMBDATEST_ACCESS_KEY=your_access_key
```

---

## Step 1 — `create` (create project + upload files) · run once

Run from the project folder. This creates the project and uploads the **jmx + jar + properties files** in one request, then prints the generated project ID. It does **not** run a test:

```bash
python3 hyperexecute_automation.py create \
  --userName YOUR_USERNAME \
  --accessKey YOUR_ACCESS_KEY \
  --project-name "<YOUR_PROJECT_NAME>" \
  --files <YOUR_JMX_FILE> [<extra files...>]
```

For example:

```bash
python3 hyperexecute_automation.py create \
  --userName YOUR_USERNAME \
  --accessKey YOUR_ACCESS_KEY \
  --project-name "AutomatedSample" \
  --files loadtesting.jmx jmeter-parallel-0.12.jar system.properties user.properties
```

`create` takes the **project name you choose** and the **files to upload**. Replace `<YOUR_PROJECT_NAME>` with any name (must be unique in your account) and `<YOUR_JMX_FILE>` with your own `.jmx`. List every file after `--files`, separated by spaces — they are all uploaded together. You can include `.jmx`, `.jar` (plugins), `.properties`, `.csv` data files, etc.

It prints the generated project ID on its own line (easy to copy) with `#`-commented instructions and a ready-to-run `trigger` command:

```
######################################################################
#  PROJECT CREATED - COPY YOUR PROJECT ID BELOW
#  Provide this value to the 'trigger' command via --project-id
#  (or export it: export HYPEREXECUTE_PROJECT_ID=<the id below>)
#---------------------------------------------------------------------
01KY23M7T3YZAC3W0R433ZSSQV
#---------------------------------------------------------------------
#  Ready-to-run example (no re-upload needed) - edit run params freely:
#
#  python3 hyperexecute_automation.py trigger \
#    --userName myuser --accessKey <ACCESS_KEY> \
#    --project-id 01KY23M7T3YZAC3W0R433ZSSQV \
#    --jmx-path loadtesting.jmx \
#    --regions eastus --duration 60 --rampup 60 \
#    --max-vusers-per-vm 5 --global-timeout 90 \
#    --job-label "AutomatedJmeterSample"
######################################################################
```

> Project **names must be unique** in your account. Re-running `create` with a name that already exists will not create a duplicate — it stops and tells you to use `trigger` with the existing project ID instead.

---

## Step 2 — `trigger` (run the test) · run as often as you like

With the project created and files uploaded, run tests against it with `--project-id`. This is where **all the run parameters** live (regions, duration, ramp-up, max vusers, timeout, label). No re-upload happens:

The only required inputs are the project ID and the jmx name. Everything else is optional — **if you don't pass a run parameter, the value from the uploaded `.jmx` is used**:

```bash
python3 hyperexecute_automation.py trigger \
  --userName YOUR_USERNAME \
  --accessKey YOUR_ACCESS_KEY \
  --project-id <YOUR_PROJECT_ID> \
  --jmx-path <YOUR_JMX_FILE> \
  --regions eastus
```

Provide any run parameters you want to override at runtime:

```bash
python3 hyperexecute_automation.py trigger \
  --userName YOUR_USERNAME \
  --accessKey YOUR_ACCESS_KEY \
  --project-id <YOUR_PROJECT_ID> \
  --jmx-path <YOUR_JMX_FILE> \
  --regions westus eastus centralindia southeastasia \
  --duration 60 --rampup 60 \
  --max-vusers-per-vm 5 --global-timeout 90 \
  --job-label "<YOUR_JOB_LABEL>"
```

- **`--project-id` and `--jmx-path` are supplied at runtime** — replace `<YOUR_PROJECT_ID>` with the ID printed by `create`, and `<YOUR_JMX_FILE>` with your uploaded jmx name (e.g. `loadtesting.jmx`). You can change them on every run.
- Instead of the flags, you can export them once: `export HYPEREXECUTE_PROJECT_ID=...` and `export HYPEREXECUTE_JMX_PATH=...`.
- **`--duration`, `--rampup`, `--max-vusers-per-vm`, `--global-timeout` are optional** — omit any of them and the value baked into your `.jmx` (or the HyperExecute default) is used. Only the parameters you pass are sent.
- **`--job-label` is optional and customer-provided** — pass any label you like; omit it and no label is sent.
- Pass multiple regions with `--regions` (space-separated) — each becomes a separate run entry.

To change the uploaded files later, run `create` again with a new `--project-name`, or update them from the HyperExecute dashboard.

---

## Runtime values you can provide (`trigger`)

What each runtime flag accepts:

| Flag                  | Accepts                                  | Example                                | Notes |
| --------------------- | ---------------------------------------- | -------------------------------------- | ----- |
| `--project-id`      | the project ID string                    | `01KY24Y2RZ7AT69S720NXTWQ1N`         | Printed by `create`; or `HYPEREXECUTE_PROJECT_ID` |
| `--jmx-path`        | the uploaded jmx filename                | `loadtesting.jmx`                    | The name of the `.jmx` you uploaded; or `HYPEREXECUTE_JMX_PATH` |
| `--regions`         | one or more region codes (space-sep)     | `--regions westus2 eastus`           | Each region = one run entry (see list below) |
| `--duration`        | integer seconds                          | `--duration 300`                     | Omit → use the value in the `.jmx` |
| `--rampup`          | integer seconds                          | `--rampup 60`                        | Omit → use the value in the `.jmx` |
| `--max-vusers-per-vm` | integer                                | `--max-vusers-per-vm 5`              | Omit → HyperExecute default |
| `--global-timeout`  | integer minutes                          | `--global-timeout 90`                | Omit → HyperExecute default |
| `--concurrency`     | integer                                  | `--concurrency 2`                    | Default `1` |
| `--splitcsv`        | (flag, no value)                         | `--splitcsv`                         | Split CSV data across nodes |
| `--job-label`       | any text                                 | `--job-label "Nightly load"`         | Optional dashboard label |
| `--report-dir`      | directory name                           | `--report-dir report`                | HTML report output dir (default `report`) |

### Regions

Pass region codes space-separated to `--regions`. Supported regions:

| Region (dashboard display)              | Code for `--regions` |
| --------------------------------------- | ---------------------- |
| West US 2 (Moses Lake, Washington)      | `westus2`            |
| East US (Richmond, Virginia)            | `eastus`             |
| Central India (Pune, Maharashtra)       | `centralindia`       |
| Southeast Asia (Singapore)              | `southeastasia`      |
| Brazil South (São Paulo State, Brazil)  | `brazilsouth`        |
| Mexico Central (Querétaro State, Mexico)| `mexicocentral`      |

> Codes follow Azure's region naming. If a run rejects a region code, copy the exact value from the **HyperExecute dashboard → configure a JMeter job → Region dropdown** (some accounts use `westus` for West US 2).

Examples:

```bash
# single region
--regions eastus

# multiple regions (each runs the test and produces its own run entry)
--regions westus2 eastus centralindia southeastasia
```

---

## Reporting & artifacts

The trigger payload always includes `-e -o <report-dir>` to generate the JMeter HTML dashboard. Results are collected as **two separate, non-overlapping artifacts**:

| Artifact             | Source                                           | Contents                                                                        | Downloaded as           |
| -------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------- | ----------------------- |
| **`JMeter`** | auto-generated by HyperExecute                   | `errors.xml`, `jmeter.log`, `results.jtl`                                 | `<job-id>_JMeter.zip` |
| **`report`** | defined in the trigger payload (`report/**/*`) | the HTML dashboard only (`index.html`, `content/`, `statistics.json`, …) | `<job-id>_report.zip` |

HyperExecute already emits the `JMeter` (logs) artifact on its own, so the script only adds the `report` artifact — defining a second `JMeter` would create a duplicate same-named artifact. Both are downloaded automatically at the end of a `trigger` run (unless `--no-download`). Use `--output NAME` to change the filename prefix (`NAME_JMeter.zip` / `NAME_report.zip`).

| Flag                    | Meaning                                                                |
| ----------------------- | ---------------------------------------------------------------------- |
| `--report-dir`        | HTML report output directory, used with`-e -o` (default: `report`) |
| `--extra-jmeter-args` | Extra raw JMeter args appended after`-e -o <report-dir>`             |

> **Important:** The HyperExecute trigger API only accepts a limited set of JMeter command args (essentially `-e -o <dir>`). It **rejects** `-J<property>=value` overrides. So report tuning - **report title, percentiles, granularity, Apdex thresholds** - must be set in `user.properties` (which is uploaded with the job), e.g.:
>
> ```properties
> jmeter.reportgenerator.report_title=Wipro E2E Load Test
> aggregate_rpt_pct1=90
> aggregate_rpt_pct2=95
> aggregate_rpt_pct3=99
> ```

---

## All options

**`create` command** (create project + upload files):

| Flag               | Description                                                 | Default    |
| ------------------ | ----------------------------------------------------------- | ---------- |
| `--project-name` | Name for the new project (**required**)               | –         |
| `--files`        | All file(s) to upload, space-separated (**required**) | –         |
| `--project-type` | Project type                                                | `jmeter` |

**`trigger` command** (run a test — holds all run parameters):

| Flag                         | Description                                                           | Default    |
| ---------------------------- | --------------------------------------------------------------------- | ---------- |
| `--project-id`             | Existing project ID — runtime (or`HYPEREXECUTE_PROJECT_ID`)        | –         |
| `--jmx-path`               | JMX name, e.g.`loadtesting.jmx` — runtime (or `HYPEREXECUTE_JMX_PATH`) | –         |
| `--regions`                | One or more regions, space-separated (per-region run)                 | `eastus` |
| `--duration`               | Test duration in seconds (per region)                                 | from `.jmx` |
| `--rampup`                 | Ramp-up period in seconds (per region)                                | from `.jmx` |
| `--max-vusers-per-vm`      | Max virtual users packed on one VM                                    | (omitted)  |
| `--global-timeout`         | Overall job timeout in minutes                                        | (omitted)  |
| `--concurrency`            | Job concurrency level                                                 | `1`      |
| `--splitcsv`               | Split CSV data across nodes                                           | off        |
| `--job-label`              | **Optional** dashboard label; empty if omitted                  | (none)     |
| `--report-dir`             | HTML report output directory                                          | `report` |
| `--extra-jmeter-args`      | Extra raw JMeter args after`-e -o <dir>`                            | –         |
| `--job-poll-interval`      | Job status poll interval (s)                                          | `10`     |
| `--artifact-poll-interval` | Artifact status poll interval (s)                                     | `10`     |
| `--output`                 | Output zip filename                                                   | auto       |
| `--no-download`            | Skip download (just run + confirm)                                    | off        |

**Both commands:** `--userName`, `--accessKey` (or env vars), `--debug`.

See everything with:

```bash
python3 hyperexecute_automation.py --help
python3 hyperexecute_automation.py create --help
python3 hyperexecute_automation.py trigger --help
```

---

## What the output looks like

Clean, scannable output (no emoji). `create`:

```
==============================================================
  HyperExecute  ·  Create Project + Upload Files
==============================================================
  User          rishabhsinghtestmuai
  Project       AutomatedSample  (jmeter)
  Files         loadtesting.jmx, jmeter-parallel-0.12.jar, system.properties, user.properties
--------------------------------------------------------------
  -> Creating project 'AutomatedSample' (jmeter) ...
  [OK] Project created  (ID: 01KY24Y2RZ7AT69S720NXTWQ1N)
  -> Uploading 4 file(s):
       - loadtesting.jmx
       - jmeter-parallel-0.12.jar
       - system.properties
       - user.properties
  [OK] Files uploaded

##############################################################
#  PROJECT CREATED  -  copy your Project ID below
...
```

`trigger`:

```
==============================================================
  HyperExecute  ·  Trigger Test Run
==============================================================
  User          rishabhsinghtestmuai
  Project ID    01KY24Y2RZ7AT69S720NXTWQ1N
  JMX           loadtesting.jmx
  Regions       westus, eastus, centralindia, southeastasia
  Load          duration 60s / rampup 60s / 5 vusers-per-vm
  Timeout       90 min
  Job Label     AutomatedJmeterSample
--------------------------------------------------------------
  -> Triggering job in 4 region(s): westus, eastus, centralindia, southeastasia
  [OK] Job triggered  (Job ID: d456e691-...)
  -> Waiting for job to finish (polling every 10s) ...
       [   1s] initiated
       [  74s] running
       [ 210s] completed
  [OK] Job completed

  Job Number    7
  Tasks         4 completed / 0 failed / 4 total
  Exec Time     58s
  -> Waiting for artifacts (polling every 10s) ...
  [OK] Artifacts ready
  JMeter        204800 bytes
  report        4404019 bytes
  -> Downloading artifact 'JMeter' ...
  [OK] Downloaded d456e691-..._JMeter.zip (200.0 KB)
  -> Downloading artifact 'report' ...
  [OK] Downloaded d456e691-..._report.zip (4.2 MB)

==============================================================
  Done
==============================================================
  Job ID        d456e691-...
  Project ID    01KY24Y2RZ7AT69S720NXTWQ1N
  Dashboard     https://hyperexecute.lambdatest.com/hyperexecute/jobs/d456e691-...
==============================================================
```

Add `--debug` to any command to also print the raw HTTP URLs, request payloads, and response bodies.

---

## Exit codes

- `0` – workflow completed successfully
- `1` – a step failed (create, upload, trigger, job, artifacts, or download); a concise reason is printed. Add `--debug` for the raw API request/response details.

---

## Troubleshooting

- **"A project named X already exists"** – you've already created it. Switch to the `trigger` command with that project's ID (shown when it was first created, or on the HyperExecute Projects dashboard). No need to create or re-upload.
- **Project ID not found after create** – run with `--debug` to see the raw response JSON; the script looks for `id` / `projectId` / `projectID` / `data.id` / `data.projectId`.
- **Job triggers but nothing runs** – confirm `--jmx-path` matches the uploaded jmx name (default is `hyperexecute-jmeter-/<your.jmx>`).
- **401 / auth errors** – re-check `--userName` and `--accessKey`.
