# Process Mining & KRR: Group 2

Bottleneck analysis, job-shop scheduling, and live conformance checking over XES event logs from a simulated production line.

**Team:** Octaviana Cheteles, Paola Joza, Vlad Alexe

## Deliverables

| File | Contents |
|---|---|
| `Presentation_KRR_PM.pptx` | Project presentation |
| `Project_Code.ipynb` | All analysis and implementation code |

---

## 1. Viewing the presentation

**Open the presentation in Slide Show mode (F5, or Slide Show > From Beginning).**

The slides rely on animations and layered elements to build up their content step by step. In normal editing view these layers sit on top of each other, which makes most slides unreadable. Reading view (Alt+F5) works too if you prefer a windowed version.

---

## 2. Running the code

### Dependencies
pandas
matplotlib
networkx
pm4py
z3-solver


### Before the first run

The log directory is hard-coded near the top of the first cell:

```python
directory_path = '/home/jovyan/Project/FilteredFiles'
```

Change this to wherever your `.xes` files are. Only files ending in `.xes` are picked up; everything else in the folder is ignored.

### Cell order

Run the cells top to bottom. They are not independent:

1. **Bottleneck analysis.** Computes global and local average durations per event across all logs, builds the process-flow graph, finds the critical path, and selects the log with the shortest overall time. Produces the process flow diagram, the local-vs-global duration comparison chart, and the bottleneck identification.
2. **Z3 initialisation.** A sanity check that the solver is available.
3. **Job-shop scheduling.** Schedules four jobs across four workstations using the durations computed in cell 1. Depends on `local_average_durations` existing, so cell 1 must have run.
4. **Gantt chart.** Visualises the schedule from cell 3.
5. **Live conformance checking.** Interactive. See below.

---

## 3. Live conformance checking

This cell is a **tool, not a validator**. It assumes well-formed input and performs no error handling, so a typo will either crash the cell or silently corrupt the running prediction. It is not defensive by design; the intent was to demonstrate the conformance logic, not to build a robust interface.

To test it properly, follow the prompts exactly as written in the output.

### Input rules

**Start with `Arrival`, using `00:00:00` for both the start and end time.**

The prediction is anchored on the arrival timestamp, so entering anything else first produces meaningless end-time estimates for the rest of the run.

After that:

- **Activity names are case-sensitive and must match exactly.** Valid values: `Arrival`, `Welding`, `Painting`, `Drilling`, `Sawing`, `Drain`. A misspelling or lowercase entry will raise a `ValueError` and stop the loop.
- **Enter each activity only once.** Activities are removed from the pending list as they are entered, so a repeat raises the same error.
- **Times must be `hh:mm:ss`,** zero-padded, 24-hour. For example `12:04:30`, not `12:4:30`.
- **Enter `Drain` last.** It ends the loop and prints the closing message. Without it the cell keeps prompting indefinitely.

### Expected sequence
Arrival -> Drilling -> Sawing -> Welding -> Painting -> Drain


Transport steps between stations are not entered manually. A fixed 60-second transport duration is applied automatically between consecutive activities.

### What the output tells you

After each entry the tool reports how your actual duration compared against the average for that activity, lists the activities still outstanding, and updates the predicted end time of the whole process.

---

## 4. Method notes

**Bottleneck identification** compares each activity's local average duration (within one log) against its global average (across all logs). The activity with the largest positive difference is flagged as the most likely bottleneck. `Transport` is explicitly excluded from this comparison, since its duration is dominated by AGV availability rather than by the workstation itself.

**The critical path** is computed over a directed graph in which `Arrival` and `Drain` act as fixed source and sink, and all four processing activities must be traversed in between. The scheduling search then selects the log whose critical path duration is shortest.

**Temporal constraints** are expressed using Allen's Interval Algebra. The intended process is a chain of `Meet` relations, with each activity meeting the transport step that follows it:
Meet(Arrival, Transport1) -> Meet(Transport1, Drilling) -> Meet(Drilling, Transport2)
-> Meet(Transport2, Sawing) -> Meet(Sawing, Transport3) -> Meet(Transport3, Welding)
-> Meet(Welding, Transport4) -> Meet(Transport4, Painting) -> Meet(Painting, Transport5)
-> Meet(Transport5, Drain)
