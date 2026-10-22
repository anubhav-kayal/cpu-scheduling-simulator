# CPU Scheduling Simulator — Project Explanation and Presentation Guide

This guide explains the project from the problem it solves through its implementation, output, testing, and limitations. It is written to support both understanding the code and presenting the project aloud.

## 1. Project at a glance

The **CPU Scheduling Simulator** is an interactive desktop application for learning how operating systems choose which process gets CPU time. A user enters a set of processes, selects a scheduling algorithm, and sees the resulting execution timeline and performance metrics. A comparison screen runs the same workload through all supported algorithms.

The simulator implements:

1. First Come First Served (FCFS)
2. Shortest Job First (SJF), non-preemptive
3. Shortest Remaining Time First (SRTF), preemptive SJF
4. Priority scheduling, non-preemptive
5. Priority scheduling, preemptive
6. Round Robin (RR), with a configurable time quantum

The application is a **teaching simulation**. It schedules the process data entered by the user; it does not control real processes or change the host operating system's scheduler.

## 2. Problem statement and motivation

CPU scheduling is the operating system's way of deciding which ready process should run next and, for some algorithms, how long it should run before the decision is reconsidered. The algorithm affects how long processes wait, how quickly they first receive CPU time, and how long they take to finish.

These ideas are often taught with hand-drawn Gantt charts and manually calculated times. That can be slow and error-prone, particularly for preemptive algorithms where a process can run, be interrupted, and later resume. It is also difficult to compare several algorithms fairly if the workload has to be recalculated separately for each one.

This project addresses that problem by letting a user:

- Enter one workload once.
- Generate an exact schedule using a selected policy.
- See process execution and CPU idle periods on a timeline.
- Inspect process-level and average timing metrics.
- Compare the same workload under all six scheduling modes.

## 3. Objectives

The project aims to:

- Implement six classic scheduling modes with repeatable, deterministic decisions.
- Validate process inputs before simulation.
- Represent the schedule in a common timeline format that every algorithm and visualization can use.
- Display a proportional Gantt chart, including CPU idle intervals.
- Calculate waiting, turnaround, and response times, plus CPU utilization and throughput.
- Compare algorithm results on the same process workload.
- Test the scheduling core using hand-calculated expected schedules and metrics.

## 4. Inputs and outputs

### Input process fields

Each process has:

| Field | Meaning |
|---|---|
| PID | Unique process identifier. |
| Arrival time | Time when the process becomes available to run. It must be non-negative. |
| Burst time | CPU time the process needs in total. It must be positive. |
| Priority | Optional priority number, required by Priority scheduling. By default, a smaller number means higher priority. |

Round Robin additionally takes a positive integer **time quantum**, the maximum CPU time granted to a process in one turn.

### Schedule output

Each scheduler returns a pair:

```text
(completed_processes, timeline)
```

The timeline is a list of segments of the form:

```text
(pid, start_time, end_time)
```

CPU idle periods are represented explicitly:

```text
("IDLE", start_time, end_time)
```

For example:

```text
[(1, 0, 2), ("IDLE", 2, 5), (2, 5, 8)]
```

means P1 runs from time 0 to 2, the CPU is idle from 2 to 5, and P2 runs from 5 to 8. In a preemptive schedule, one PID can occur in multiple timeline segments.

The shared return contract is defined in [`src/models.py`](../src/models.py). Keeping the same shape across algorithms makes the metrics and GUI independent of the individual scheduling implementation.

## 5. Scheduling concepts used in the project

### Ready queue

The **ready queue** is the set of processes that have arrived and are waiting for CPU time. A scheduler must not select a process before its arrival time.

### Preemption

A **non-preemptive** scheduler lets the selected process continue until its CPU burst finishes. A **preemptive** scheduler may interrupt a running process when a scheduling decision changes—for example, when a shorter job or a higher-priority job arrives.

### Deterministic tie-breaking

When scheduling keys tie, the project uses consistent tie-break rules so that the same workload produces the same output. PID is used to break ties in the relevant scheduling decisions; Priority scheduling also considers arrival order before PID when priorities tie. These rules are important both for predictable demonstrations and for exact tests.

## 6. The six algorithms

### 6.1 FCFS — First Come First Served

FCFS runs processes in order of arrival. If arrival times are equal, the lower PID is selected first. Once a process starts, it runs until its burst is complete.

**Strength:** Simple and easy to explain.

**Trade-off:** A long process at the front can make shorter processes wait. This is commonly called the convoy effect.

**What to point out in the app:** The timeline follows arrival order. If no process has arrived yet, an `IDLE` block appears.

### 6.2 SJF — non-preemptive Shortest Job First

At each time the CPU becomes available, SJF looks at processes that have already arrived and selects the one with the shortest total burst time. It then lets that process finish.

**Strength:** Can reduce average waiting time for a known workload.

**Trade-off:** Long processes can wait a long time if short jobs continue to be selected. In a real system, the exact future burst length may not be known.

**Important distinction:** A short process arriving after the current process has started does not interrupt it in this mode.

### 6.3 SRTF — Shortest Remaining Time First

SRTF is the preemptive counterpart to SJF. It selects the ready process with the smallest remaining CPU time and reconsiders the decision when a new process arrives or the running process completes. If a newly arrived process has less remaining work, it can preempt the current process.

**Strength:** Gives short jobs a chance to finish quickly and often reduces average waiting time.

**Trade-off:** Long processes may be repeatedly delayed. It also causes more context switches in a real operating system; this simulator models scheduling order and time, not the cost of switching.

**What to point out in the app:** A process can appear in more than one Gantt block if it is interrupted and later resumes.

### 6.4 Priority scheduling — non-preemptive

Among the processes that have arrived, this mode selects the best priority and lets it finish. By default, **a lower numeric priority value means higher priority**. If priorities tie, arrival time and then PID provide deterministic ordering.

**Strength:** Represents workloads where some processes should be favored over others.

**Trade-off:** A low-priority process can be delayed for a long time if higher-priority processes keep arriving.

### 6.5 Priority scheduling — preemptive

This mode uses the same priority convention but checks again when a process arrives. If a newly ready process has a higher priority than the current process, it preempts it. The interrupted process can resume later.

**What to point out in the app:** Compare this schedule with the non-preemptive version using the same input. A high-priority process that arrives during another process's run can start immediately in the preemptive version, but must wait in the non-preemptive version.

**Limitation:** Priority aging is not implemented. Aging gradually raises the priority of processes that have waited for a long time and is a common way to reduce starvation.

### 6.6 Round Robin

Round Robin uses a FIFO ready queue. Each ready process receives up to one time quantum. If it still has work left when its quantum expires, it goes to the back of the queue. Newly arrived processes during a time slice enter the ready queue before the just-preempted process is appended again.

**Strength:** Gives ready processes regular turns and is useful for illustrating interactive time sharing.

**Trade-off:** The quantum affects behavior. A very small quantum creates many short turns; a very large quantum reduces preemption and can behave more like FCFS for a particular workload. The simulator does not charge time for real context switches.

## 7. How the metrics are calculated

For each completed process:

### Turnaround time

```text
turnaround_time = completion_time - arrival_time
```

This is the total elapsed time from process arrival until completion.

### Waiting time

```text
waiting_time = turnaround_time - burst_time
```

This is the process's total time waiting rather than executing on the CPU.

### Response time

```text
response_time = first_start_time - arrival_time
```

This measures how long the process waits before it first gets CPU time. For a preemptive process, it uses the **first** start, not the start of its final segment.

For the workload:

### CPU utilization

With a timeline, the calculation is:

```text
CPU utilization = busy timeline time / total timeline span × 100%
```

`IDLE` segments count as non-busy. This makes idle gaps visible in the utilization result.

### Throughput

```text
throughput = completed process count / total timeline span
```

### Averages and ranking

Average waiting, turnaround, and response times are computed across the completed processes. The comparison view displays average waiting and turnaround values and ranks algorithms by **ascending average waiting time** for the current workload.

These metrics describe different goals. An algorithm with the lowest average waiting time for one workload is not necessarily the best choice for every workload or every system.

## 8. Architecture and module responsibilities

The project separates scheduling logic from presentation:

```text
User
  ↓
InputForm (Tkinter)
  ↓
Input validation
  ↓
scheduler.py (algorithm dispatch)
  ↓
algorithms/* (completed processes + timeline)
  ↓
metrics.py
  ↓
Gantt chart + process results + comparison chart
```

### Main modules

| File or folder | Responsibility |
|---|---|
| [`src/main.py`](../src/main.py) | Starts the application and gives a platform-specific hint if Tkinter is unavailable. |
| [`src/models.py`](../src/models.py) | Defines and validates `Process`; defines the shared timeline and schedule-result types. |
| [`src/scheduler.py`](../src/scheduler.py) | Maps the algorithm name selected in the UI to its scheduling function. |
| [`src/algorithms/common.py`](../src/algorithms/common.py) | Shared helpers for copying processes, building timelines, recording first start, and finalizing results. |
| [`src/algorithms/fcfs.py`](../src/algorithms/fcfs.py) | FCFS implementation. |
| [`src/algorithms/sjf.py`](../src/algorithms/sjf.py) | Non-preemptive SJF and preemptive SRTF implementations. |
| [`src/algorithms/priority.py`](../src/algorithms/priority.py) | Non-preemptive and preemptive Priority implementations. |
| [`src/algorithms/round_robin.py`](../src/algorithms/round_robin.py) | Round Robin implementation using a ready queue and time quantum. |
| [`src/metrics.py`](../src/metrics.py) | Per-process averages, utilization, throughput, summaries, and algorithm comparison. |
| [`src/gui/input_form.py`](../src/gui/input_form.py) | Process entry, algorithm selection, conditional fields, validation, and user messages. |
| [`src/gui/gantt_chart.py`](../src/gui/gantt_chart.py) | Draws the timeline using Matplotlib embedded in Tkinter. |
| [`src/gui/comparison_view.py`](../src/gui/comparison_view.py) | Displays per-process results and the cross-algorithm comparison chart. |
| [`src/gui/app.py`](../src/gui/app.py) | Builds the main window and connects the input, scheduler, metrics, and views. |

### Why algorithms use copies

Each scheduling run works with copies of the input processes. The algorithm updates fields such as remaining time and completion time on those copies, leaving the user's form data unchanged. This prevents one simulation from contaminating a later run or a comparison.

### Why there is a shared timeline

Every algorithm writes segments in the same format. The GUI therefore does not need separate charting code for each algorithm. The timeline also preserves interruptions and explicit idle time, making it the source for the Gantt chart and timeline-based utilization calculations.

## 9. End-to-end explanation of a run

When the user clicks **Run Simulation**, the application follows this sequence:

1. The input form checks that the required fields are present and valid.
2. It creates `Process` objects for the entered rows.
3. For Round Robin, it reads and validates the time quantum. For Priority scheduling, it requires priorities.
4. The application sends the selected algorithm name, processes, and optional quantum to `run_algorithm`.
5. `scheduler.py` dispatches the request to the corresponding algorithm function.
6. The algorithm copies the processes, applies its scheduling rule, and creates timeline segments. It adds `IDLE` segments whenever the CPU has no ready process.
7. As a process first receives CPU time, the algorithm records its start. When it completes, the model derives turnaround, waiting, and response times.
8. The metrics module calculates the run summary.
9. The GUI refreshes the Gantt chart and the per-process result table and updates its status text.

When the user clicks **Run Comparison**, the comparison view obtains the current workload, runs each supported algorithm on it, collects each schedule and metrics summary, draws grouped bars for average waiting and turnaround times, and reports the lowest average waiting time.

## 10. Input validation and edge cases

The application and model validate the important scheduling inputs:

- PIDs must be integer identifiers and unique within a workload.
- Arrival times must be integers greater than or equal to zero.
- Burst times must be positive integers.
- Priority values must be integers when Priority scheduling is selected.
- Round Robin quantum must be a positive integer.
- Metrics require completed process results.

The scheduling core also handles cases documented in the tests, including empty input, a single process, late arrivals that produce idle time, duplicate arrival times, equal bursts, and deterministic ties. Empty input returns an empty schedule from the algorithm functions; the GUI prevents a normal run with no processes.

## 11. Testing and validation

The test suite is written with pytest. It compares algorithm output to known schedules and hand-calculated values, rather than only checking that the code runs. Tests cover:

- FCFS order and idle gaps.
- SJF selection and SRTF preemption.
- Priority ordering, priority direction, and preemption.
- Round Robin queue order and different quantum sizes.
- Waiting, turnaround, response, utilization, throughput, and comparison ranking.
- Edge cases and deterministic tie-breaking.

The [pre-submission checklist](checklist.md) records **39 tests passed**. This is the status recorded in that file; it has not been re-run specifically while preparing this guide.

Run the suite from the repository root:

```bash
python3 -m pytest tests/ -v
```

The sample workloads with expected Gantt orders and metrics are in [`sample_test_cases.md`](sample_test_cases.md).

## 12. How to install and run

The project requires Python 3.11 or newer, Tkinter for the desktop GUI, and Matplotlib. Pytest is used for tests. Dependencies are listed in [`requirements.txt`](../requirements.txt).

```bash
python3 -m pip install -r requirements.txt
python3 src/main.py
```

On macOS, some Python installations do not include Tkinter. The repository includes [`run.sh`](../run.sh), which searches for an interpreter with Tkinter and then launches the GUI:

```bash
chmod +x run.sh
./run.sh
```

## 13. Suggested live presentation sequence

The following sequence demonstrates different behaviors without requiring a large workload.

### Demo 1: FCFS with an idle period

Enter:

| PID | Arrival | Burst |
|---:|---:|---:|
| 1 | 0 | 2 |
| 2 | 5 | 3 |

Choose FCFS and run the simulation. The expected timeline is:

```text
P1 [0–2] → IDLE [2–5] → P2 [5–8]
```

**Say:** “The CPU has no ready process between times 2 and 5, so the simulator represents that interval explicitly as IDLE. The Gantt chart and CPU utilization both reflect this gap.”

### Demo 2: SRTF preemption

Use **TC-SRTF-1** from [`sample_test_cases.md`](sample_test_cases.md). P1 begins first, then P2 arrives with less remaining work and takes over.

**Say:** “SRTF makes a new decision when a process arrives. Because P2 has less remaining time, it preempts P1. P1 is not lost; it resumes later.”

### Demo 3: Preemptive versus non-preemptive Priority

Use the matching late high-priority arrival examples **TC-PRIO-NP-3** and **TC-PRIO-P-1** from [`sample_test_cases.md`](sample_test_cases.md).

**Say:** “The inputs are comparable, but the policies handle a higher-priority arrival differently. Non-preemptive Priority waits for the current process to finish; preemptive Priority switches to the newly arrived higher-priority process.”

### Demo 4: Round Robin quantum

Use **TC-RR-1** with `q=1`, then optionally use **TC-RR-3** with a larger quantum.

**Say:** “The quantum controls how long a process can run in one turn. With a small quantum, the Gantt chart shows frequent rotation. With a large quantum, fewer processes are interrupted during their burst.”

### Demo 5: Compare all algorithms

Keep a workload in the input form, open **Compare All Algorithms**, and select **Run Comparison**.

**Say:** “Every algorithm receives the same workload. The bars show average waiting and turnaround time, and the summary identifies the lowest average waiting time for this input. The result is workload-specific, not a universal ranking.”

### Demo 6: Testing

If there is time, run:

```bash
python3 -m pytest tests/ -v
```

**Say:** “The core algorithms and metrics are checked against known timelines and hand-calculated results, including idle gaps, preemption, quantum behavior, and edge cases.”

## 14. Suggested spoken explanation

> “Our project is a desktop CPU Scheduling Simulator designed to make classic operating-system scheduling algorithms easier to understand. The user enters processes with arrival and burst times, and can provide priorities or a Round Robin time quantum when needed. The application validates that input and runs one of six scheduling modes: FCFS, non-preemptive SJF, SRTF, preemptive and non-preemptive Priority, or Round Robin.
>
> Each scheduler returns completed process data and a timeline. The timeline records which process ran during each interval and includes explicit IDLE intervals when nothing was ready. The GUI displays that timeline as a Gantt chart and shows start, completion, waiting, turnaround, and response times for each process. It also calculates workload metrics such as CPU utilization and throughput.
>
> The comparison screen runs the same workload through all six modes and charts their average waiting and turnaround times. Internally, the project separates the data model, scheduling algorithms, metrics, and Tkinter interface. Algorithms work on copies of the input so that one run does not affect another. We test the scheduler using hand-calculated examples and edge cases. The result is an educational simulator—it models scheduling decisions but does not schedule real operating-system processes.”

## 15. Likely questions and concise answers

### Does this program schedule real processes on the computer?

No. It simulates scheduling for the process records entered by the user. It does not interact with the operating system's real process scheduler.

### What is the difference between waiting time and response time?

Waiting time is all time a process spends waiting during its lifetime. Response time is only the time from arrival until its first CPU allocation.

### Why can waiting time be calculated as turnaround time minus burst time?

Turnaround includes both CPU execution and time waiting. Subtracting the total CPU burst leaves the waiting time, assuming the simulated process uses one CPU burst.

### Why does a preemptive process appear more than once in the Gantt chart?

It can run for a while, be interrupted by another process, and later resume. Each uninterrupted interval becomes a separate timeline segment.

### Why do you show IDLE?

It records intervals where no process has arrived and is ready to execute. Hiding idle time would make the execution timeline and CPU utilization misleading.

### Which algorithm is best?

There is no universally best algorithm. Different policies optimize different goals, and their measured performance depends on the workload. The comparison screen reports results for the workload entered.

### What is the starvation issue in Priority scheduling?

If higher-priority processes keep arriving, a low-priority process may wait for a very long time. Priority aging is a common mitigation, but it is outside this implementation.

### What does the Round Robin quantum change?

It sets the maximum time one process can run in a turn. A small quantum creates more frequent rotation; a large quantum reduces the number of interruptions for a given workload.

### Why use a common output format?

It allows the chart, metrics, and comparison features to consume schedules from any algorithm without duplicating algorithm-specific presentation logic.

## 16. Scope and limitations

The project focuses on classic scheduling concepts and their visible timing consequences. It intentionally does not model:

- Actual operating-system processes, multiple CPU cores, or concurrent execution.
- Context-switch overhead.
- I/O bursts or processes that alternate between CPU and I/O.
- Priority aging.
- Multilevel queue or multilevel feedback queue scheduling.
- Real-time scheduling deadlines.

These are reasonable future enhancements, but they are not required to demonstrate the implemented algorithms.

## 17. Project status and supporting material

The build plan records phases 0–12 as complete. The [README](../README.md) explains the user-facing product and setup. The [report template](report_template.md) can be used to assemble a formal written report, while the [pre-submission checklist](checklist.md) records remaining manual presentation and submission items.

In particular, verify screenshot links and report content before submission: the repository has screenshot assets under [`docs/screenshots/`](screenshots/), while some README and report image references are placeholders. Add any missing team details and capture or link the screenshots you intend to show.

## 18. One-minute summary

> “This project is an interactive desktop simulator for six CPU scheduling modes. It takes a user-defined process workload, validates it, generates a deterministic schedule, and visualizes the result as a Gantt chart with idle time shown. It reports per-process waiting, turnaround, and response times, as well as CPU utilization and throughput. The comparison screen runs the same workload through every mode to help explain their trade-offs. The code is separated into models, algorithms, metrics, and GUI modules, and the scheduling behavior is covered by pytest tests. It is intended for learning and demonstration, not for controlling real operating-system scheduling.”
