# DailyPlanner — Assignment 7

Yerkebulan Abenov, Nurbergen Assan, Amanalina Tumargul and Khalid Ahmad Ahmadi — **IT-2511**.

Repository: [Assignment-7-of-ITFP](https://github.com/Abenovv/Assignment-7-of-ITFP)

This continues the daily task planner from Assignments 5 and 6. A student can plan all eligible tasks for a date, look up a task, or request a confirmed plan for one task. Assignment 7 makes absence and failure explicit throughout those workflows.

## Build, run and test

Use **.NET SDK 10.0.401**. No database, account or third-party package is needed. Extract the ZIP, then open Terminal or PowerShell in the folder containing `DailyPlanner.sln`. These commands work on Windows and macOS:

```text
dotnet build DailyPlanner.sln --disable-build-servers --maxcpucount:1
dotnet run --project Tests/DailyPlanner.Tests.csproj --no-build
dotnet run --project DailyPlanner.csproj --no-build
```

The second command runs the automated assertions. It prints PASS or FAIL and exits with code 1 if a test fails. Use it instead of `dotnet test`; this project has its own test runner.

The runner contains **68 cumulative tests**: 44 retained tests and 24 tests for this stage. Latest verification, October 25, 2026: build succeeded with zero warnings and errors; **68/68 tests passed**. All 29 CLI success, failure and lookup checks passed. The package is also verified from a fresh extraction.

## Input and the retained pipeline

Input is a JSON array. Each task requires a unique nonblank `id`, nonblank `title`, real `date` in `yyyy-MM-dd` format, `importance` from 1 to 5, `minutes` from 0 to 480, and boolean `completed`. IDs are case-sensitive. Unknown, missing or null fields and invalid values reject the whole input. Zero minutes is a valid reminder; `false` is a real completion value.

The pure stages still perform useful work:

1. `SelectTasks` keeps unfinished tasks on the requested date whose importance meets the threshold.
2. `ScoreTasks` calls the supplied `Func<TaskItem, int>` policy.
3. `BuildPlan` sorts by descending score, breaks ties by ordinal ID and assigns positions starting at 1.

Importance first scores `importance * 10`. Short tasks first scores `importance * 10 + max(0, 60 - minutes)`. Stages return copied read-only collections and do not read files, print, inspect the clock or mutate input.

The default run uses `samples/tasks.json`, the fixed date `2026-10-13`, both policies, minimum importance 3 and 180 available minutes. Expected orders are **T1, T2, T7, T3** and **T3, T7, T2, T1**. Each plan has four tasks, totals 120 minutes and confirms with 60 minutes left.

```text
dotnet run --project DailyPlanner.csproj --no-build -- samples/tasks.json 2026-10-13 short 3 180
```

Arguments are file, date, policy (`both`, `importance` or `short`), minimum importance and available minutes. Omitted trailing arguments use the defaults. File paths are relative to the current working directory.

## Success and domain failures

A single-task request uses the same pipeline and the safe preparation/confirmation workflow:

```text
dotnet run --project DailyPlanner.csproj --no-build -- --task T3 samples/tasks.json 2026-10-13 short 120
```

Expected: T3 receives score 80, the plan totals 10 minutes, and `PLAN-20261013` confirms one task with 110 minutes left. Single-task requests use minimum importance 1 because the student explicitly chose the task. Available time must be 1–480 minutes, and the chosen task must fit it.

Each command below exits 1 with a useful Result failure. No business failure requires throwing and catching an exception.

| Command arguments after `--` | Expected failure |
| --- | --- |
| `--task UNKNOWN` | `Task 'UNKNOWN' was not found.` |
| `--task T5` | `Task 'T5' is already completed.` |
| `--task T6` | `Task 'T6' belongs to 2026-10-14, not 2026-10-13.` |
| `--task T1 samples/tasks.json 2026-10-13 importance 59` | `Preparation rejected: The selected tasks exceed the available time.` |
| `--task T1 samples/tasks.json 2026-10-13 importance 0` | Preparation rejects available time outside 1–480. |

A missing prerequisite and repetition remain explicit in the Assignment 6 state workflow. `PlanningWorkflow.Confirm(Draft)` rejects confirmation before preparation. Preparing or confirming a confirmed state is rejected. The demonstration shows those cases:

```text
dotnet run --project DailyPlanner.csproj --no-build -- --error-demo
```

`--workflow-demo` is an alias. This demonstration deliberately prints several successes and failures and exits 0 when the sample loads. Run individual failure commands to observe exit code 1.

## Missing values: optional versus required

`TaskLookup.Find` returns `Option<TaskItem>.Some` or `.None`; it never fabricates a task. `Some` requires a non-null payload. The lookup trims the requested ID and compares it with ordinal case-sensitive equality.

```text
dotnet run --project DailyPlanner.csproj --no-build -- --lookup T3
dotnet run --project DailyPlanner.csproj --no-build -- --lookup UNKNOWN
dotnet run --project DailyPlanner.csproj --no-build -- --lookup R0 samples/reminders.json
dotnet run --project DailyPlanner.csproj --no-build -- --task R0 samples/reminders.json
```

The first command finds T3. The second reports that no task exists and exits 0: this is an optional lookup, and no plan was requested. The reminder lookup prints `minutes=0; completed=false`; both values are present. The last command confirms the zero-minute reminder.

`TaskLookup.Required` converts `None` to a Result failure for `--task`, because a request cannot continue without the chosen task. Tests also distinguish `Some(0)`, `Some(false)` and `Some("")` from `None`. Empty titles and IDs remain invalid under task validation; the generic Option model itself does not mistake empty text for absence.

## Result, Map and Bind

`Result<T>` has closed `Success` and `Failure` cases. Success carries a required value; failure carries a nonblank error. Construction rejects a null success payload, including one produced by Map. These guards detect incorrect API use; ordinary request rejection is represented by Result failures.

The real request chain is:

```csharp
TaskLookup.Required(tasks, id)
    .Bind(task => CheckEligibility(task, date))
    .Map(task => TaskPipeline.CreatePlan([task], date, 1, scorePolicy))
    .Bind(plan => PrepareAndConfirm(PlanningState.Start(plan, availableMinutes)));
```

`Map` accepts `T -> U`: it transforms a successful payload with a pure calculation and wraps the new value. `Bind` accepts `T -> Result<U>`: it continues with an operation that may itself fail and returns that Result directly. Both preserve the original error and skip their callback on failure.

The application adds `TaskFile.Load(path).Bind(...)` before this chain and a final Map to create display text. `Main` matches the final success/failure once and performs console output. A missing task therefore skips eligibility, scoring and confirmation; a completed or wrong-day task skips scoring and confirmation. Tests use counters to prove callbacks stay uncalled.

A Result describes the outcome of an operation. `PlanningCreation` describes factory validation. `PlanningState` describes the lifecycle of a plan: Draft, Prepared or Confirmed. These meanings are separate. `PrepareAndConfirm` converts rejected transitions into Result failures while using the existing validated factory and safe transitions.

## Exception boundaries and consistent state

File access is in `Input/TaskFile.cs`. `Read` catches `FileNotFoundException`, `DirectoryNotFoundException`, `UnauthorizedAccessException`, other `IOException`, and invalid-path `ArgumentException`/`NotSupportedException`. It returns a failure identifying the file and operation. `TaskParser` catches `JsonException` during deserialization; `TaskFile.Load` adds the file path to parser and validation failures.

```text
dotnet run --project DailyPlanner.csproj --no-build -- samples/not-found.json
dotnet run --project DailyPlanner.csproj --no-build -- samples/broken.json
dotnet run --project DailyPlanner.csproj --no-build -- samples/malformed.json
dotnet run --project DailyPlanner.csproj --no-build -- samples/invalid.json
```

`not-found.json` is intentionally absent. `broken.json` contains incomplete JSON, `malformed.json` has an incorrect field type, and `invalid.json` violates a task rule. Each command reports the path and reason and exits 1. There is no fallback to an empty successful plan.

Parsing publishes the task collection only after every record passes validation. Domain operations create new immutable values. The application writes no task or plan files; it prints successful output only after the requested chain completes. Failure cannot overwrite an earlier confirmation or partially save data. Tests assert unchanged source text, task values and an existing confirmed-state reference after missing-file and later-record failures.

`[]` is valid parsed data, but a confirmed plan must contain at least one task, so `samples/empty.json` is rejected by preparation. Unexpected programming errors are not relabeled as successful business outcomes.

## Requirement and test map

| Evidence | Files and assertions |
| --- | --- |
| Guarded success/failure and Map/Bind | `Domain/Result.cs`; five `OutcomeTests.cs` tests check mapping, binding, returned Result identity, preserved errors, skipped callbacks and invalid construction. |
| Real present/missing lookup | `Domain/Option.cs`, `TaskLookup.cs`; four `LookupTests.cs` tests cover present/absent, required absence, case-sensitive IDs, zero, false and empty text. |
| Three or more domain failures in a real chain | `Domain/TaskRequestWorkflow.cs`; seven `RequestTests.cs` tests check correct confirmation, missing/completed/wrong-day tasks, capacity, preservation and reminders. |
| File and parsing boundaries | `Input/TaskFile.cs`, `TaskParser.cs`; eight `FileBoundaryTests.cs` tests cover valid input, missing file/directory, malformed JSON, wrong types, later invalid record, empty file and invalid path. |
| Controlled file tests | Each file test creates its own temporary directory and cleans it in `finally`; tests do not depend on a laptop-specific file. |
| Earlier functionality | The 44 retained tests cover input rules, both policies, ranking, immutability, validated factories, closed cases, state transitions and presentation. |
| Actual application use | `Program.cs` routes the default planner, `--lookup`, `--task` and the demonstration through these modules. |

## Team contribution and defense

The following is the prepared Assignment 7 work split. Each member must review, adapt, test and defend their own part. Inherited code and a prepared file assignment do not establish personal authorship.

| Member, IT-2511 | Logic for this stage | Tests and explanation |
| --- | --- | --- |
| Yerkebulan Abenov | CLI integration, final outcome handling and file boundary | `FileBoundaryTests.cs`; file/parsing diagnostics and no partial update. |
| Nurbergen Assan | Option and optional/required task lookup | `LookupTests.cs`; Some/None and legitimate zero/false values. |
| Amanalina Tumargul | Result payload guards, Map and Bind semantics | `OutcomeTests.cs`; transformation, continuation and failure propagation. |
| Khalid Ahmad Ahmadi | Task request validation and Result adapter for safe planning transitions | `RequestTests.cs`; success, domain rejection and skipped scoring. |

Each member explains the relevant README section and their assigned assertions. A useful live modification is adding a request rule that rejects tasks longer than 90 minutes, writing one failure test and one boundary-success test, then rebuilding and running the tests. Keep the parser's separate 480-minute limit unchanged unless intentionally changing that rule too.
