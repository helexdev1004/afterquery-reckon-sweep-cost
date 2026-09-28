# Cost a batch-size sweep to a target metric

A harbor-format benchmark task, authored for the AfterQuery Experts programme as data
annotation for AI evaluation. Task slug `afterquery/reckon-sweep-cost` — **ML / Training Analysis**.

An AI agent gets a container, this brief and nothing else. It works alone for
4 hours. Then a sealed verifier it never sees scores the result `0` or `1`.

## What it asks for

Cost to target for the batch-size sweep

The full brief is in [`instruction.md`](instruction.md).

| | |
| --- | --- |
| **Deliverable** | `/app/cost_to_target.csv` |
| **Agent budget** | 4 hours (14400s) |
| **Expert estimate** | 4 hours |
| **Machine** | 1 CPU, 2048 MB RAM, 0 GPU |
| **Verifier budget** | 300s |
| **Tags** | `training-logs`, `batch-size-sweep`, `compute-cost`, `data-cleaning`, `python` |

## Why it is hard

Costing a hyperparameter sweep from its training logs is routine work for a machine learning engineer or a research scientist writing up scaling results: find the step at which each run reached the target loss, and turn steps into tokens and GPU-hours. The data is synthetic and was made truth first. Eight runs were trained on two global batch sizes and four GPU counts, each following its own loss curve, 8150 logged lines in all, and the 8 sealed lines come from each run's own curve.

<details>
<summary>Full note</summary>

The log files and the run sheet were written afterwards. The figures are exact and nothing is computed at marking time. The group's note gives every rule the marking uses. Read literally it gets 2 of 8, because three launchers plainly did not log as it says: one wrote perplexity under its own column name, one wrote the loss in thousandths of a nat as whole numbers, and one wrote the step in thousands. Each is obvious on sight, and each breaks at least one line where all 8 have to be right. What decides the task is the run sheet. It gives each run's log file, and for 4 runs it is wrong: two pairs of logs were copied back from the cluster under each other's names. One of the four also has an odd notation; the other three are logged exactly as the note says. The sheet's reading fails silently - every run still reaches the target at a believable step - and gets 4 of the 8 lines wrong. Nothing first-order shows it. The two runs of each pair share a global batch size and learning rate, so their curves fall at the same pace and differ only as two seeds do; a run's steps to target says nothing about how many GPUs it used. What differs is second order. A log holds the loss on the first device's share of the batch, so the fewer sequences a device sees per step, the more the logged loss jumps from one line to the next: set against the mean of its two neighbours over the flat half of the log, a log's jitter times the square root of its run's sequences per device is the same to within a factor of 1.24 across all eight - once each log is given to the run that really wrote it. Under the sheet's names four logs are out by a factor of nearly three. The target rule averages twenty lines and never looks at how one stands to the next, and the GPU count is a column the GPU-hours need anyway. The note says the run sheet's settings come from the scheduler's accounting records, so when a log's jitter disagrees with its run's GPU count it is the file name that is wrong, not the count. tests/cluster_check.py measures all of this against the sealed figures and refuses to pass otherwise: of 16 readings of the logs the right one gets 8 and the best other 7.

</details>

## How it is verified

The verifier reads /app/cost_to_target.csv with the csv module and compares each run's steps and tokens exactly and its GPU-hours within 0.1 with /tests/sealed.json; all 8 lines must be right. None of the agent's code is imported or run. The sealed figures exist only in the verifier image, /tests is mode 700 there, and the agent's container never mounts it.

<details>
<summary>Full note</summary>

Exact steps are fair because the target rule is stated in full and the sealed step was reckoned from the loss exactly as each log holds it, so a perplexity or whole-number log converts back to the same decision. The two runs of a crossed pair reach the target hundreds of steps apart and use different numbers of GPUs, so the sheet's reading gets none of their four lines right by chance. Nothing in the agent's container attests any figure and nothing has to balance; a table made on the run sheet as written is as self-consistent as the right one. The reward is binary and is written to /logs/verifier/reward.txt on every path; an empty container scores 0 and the reference scores 1. The generator is seeded and shipped.

</details>

<details>
<summary>Reference approach (spoiler)</summary>

solution/sweep_cost.py reads only /app and writes /app/cost_to_target.csv with the standard library. It reads each log, takes the logarithm of a perplexity column, divides thousandths of a nat by a thousand and multiplies thousands of steps by a thousand. For each log it measures the middle distance of a logged loss from the mean of its two neighbours over the second half of the log. Within each global batch size it then gives the jumpiest log to the run with the fewest sequences per device, the next to the next, and so on, whatever the run sheet says. Steps to target is the first logged step at which the mean of the last twenty losses is 3.200 or less; tokens and GPU-hours follow from the run's own batch size, GPU count and measured step time. In the task's own container it gets 8 of 8.

</details>

## Layout

```
instruction.md          the brief - the only thing the agent is told
task.toml               artifacts, timeouts, resources, metadata
environment/            Dockerfile and data for the agent's container
solution/solve.sh       reference solution; scores 1 by construction
tests/                  verifier image, grading logic and ground truth
```

## Running it

With [harbor](https://github.com/laude-institute/harbor) installed and Docker running:

```bash
uv tool install harbor

harbor run -p . -a oracle -e docker   # reference solution - must score 1
harbor run -p . -a nop    -e docker   # nothing happens    - must score 0
```

Those two runs are the calibration the bundle has to satisfy: the oracle run substitutes
`solution/solve.sh` for the agent and must score `1`, the nop run does nothing and must
score `0`. Anything in between means the verifier is measuring the wrong thing.

## Reading this

`tests/` and `solution/` hold the answers. To try the task honestly, read only
`instruction.md` and `environment/`.

---

Part of a set of 38 tasks indexed at
[afterquery-data-annotation](https://github.com/helexdev1004/afterquery-data-annotation).
