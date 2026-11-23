# Crash storm guard

A design for a guard in HandleError against a crash storm: the same error occurring again and again. It is switched on with a new parameter `stormGuard`, which defaults to `0`. With `0`, HandleError behaves exactly as it does today.

The examples come from the Tatin server, where a recurring fault brought the problem up.

## The problem

When the same error repeats, `HandleError` writes a full set of artefacts and runs the custom function on every occurrence, with no limit of any kind. A single recurring fault therefore fills a disk at whatever rate it occurs, and a mailbox too when a custom function sends a notification. The one notification that mattered gets buried, and a mail provider may throttle the sender until none of them arrive.

## What happens today

This describes HandleError 5.1.0. `HandleError` catches the error through the `⎕TRAP` installed by `SetTrap`, which calls `Process`. Everything below happens inside `:Hold 'HandleError.Process'`, so crashes are serialised:

| # | Step | Artefact | Switch |
|---|------|----------|--------|
| 1 | `CheckErrorFolder` | creates the error folder when missing | `checkErrorFolder` |
| 2 | `CreateCrashNamespace` | — | — |
| 3 | `CreateFilename` | — | — |
| 4 | kill all threads but its own and the main one; not in development unless `enforceOff` is 1 | — | `killThreads` |
| 5 | `WriteToLogFile` | the log entry | `logFunction` |
| 6 | `⎕DL 0.2` | — | — |
| 7 | `SaveErrorWorkspace` | the error workspace | `saveErrorWS` |
| 8 | `SaveCrash` | the `crash` component file | `saveCrash`, `saveVars` |
| 9 | `WriteToWindowsEvents` | a Windows event | `windowsEventSource` |
| 10 | `ExecuteCustomFns` | whatever the application does | `customFns` |
| 11 | `WriteHtmlFile` | the HTML report | `createHTML` |

Steps 1 to 11 run under `:Trap parms.trapInternalErrors/0 1000`. After that trap, `Process` signals `parms.signal` when it is not `0`, otherwise it either `⎕OFF`s in runtime or prints the error to the session.

`ExecuteCustomFns` traps whatever the custom function throws and adds "customFns failed" and the error to `addToMsg`. The HTML report comes last so that it shows this and anything else a custom function added to `addToMsg`.

The log entry, the workspace and the component file are written before step 10, so a guard in a custom function could hold back a notification, and with some effort the HTML report, but never those three. The guard therefore has to live in `HandleError`.

### Workspaces

In general the error workspace is by far the largest artefact, and also the best evidence there is. A threaded application cannot save a workspace at all, so for one like the Tatin server the question does not arise. Such an application should set `saveErrorWS` to `0`; its artefacts per crash are then the HTML report, the component file and the log entry.

## The design

### The signature

Everything is keyed on a signature that identifies the error: `EN` and the name and line of the function where it occurred, `Foo[6]`, taken from the second line of `⎕DMX.DM`. Both are known when `Process` starts, so the signature costs nothing to compute.

`EM` is deliberately left out. A message passed to `⎕SIGNAL` often carries data such as a filename or an ID, which would give every occurrence of one fault a signature of its own, and the backoff would never kick in.

A signature is only an approximation. The same fault reported from two different lines counts twice, and two different faults with the same error number on one line count once. Both are acceptable: the first costs a few extra artefacts, and the second is rare.

### The rules

With `stormGuard` set to 1, every crash increments the count for its signature. A crash then gets the *full treatment*, all eleven steps as they are today, only when both of these rules allow it:

1. Exponential backoff per signature: the count is at least twice `lastFull`, the count at which this signature last got the full treatment, which starts at 0. Normally that means occurrences 1, 2, 4, 8, 16 and so on, so a thousand occurrences cost ten sets of artefacts. The backoff never holds a new error back, because a new error has its own signature and its own first occurrence.
2. A ceiling across all signatures: fewer than `stormCeiling` crashes got the full treatment within the last hour. This catches a storm of *distinct* signatures, which the backoff cannot see and which a corrupted workspace or an exhausted resource produces: one cause, many different errors. During such a storm the ceiling can hold back a new error too. That is deliberate.

Together they keep a single recurring fault down to a handful of artefacts, leave a new fault visible outside a storm and bound the total whatever the mix of signatures.

Comparing with `lastFull` rather than testing for a power of two means that an occurrence the ceiling blocks is not lost. When the ceiling blocks occurrence 8, `lastFull` stays at 4, so occurrence 9 qualifies again, and so does every later one until the ceiling lets one through. That one becomes the new `lastFull`, and the doubling continues from there.

A signature that has not occurred for `stormResetAfter` hours has its count and `lastFull` set back to 0, so a fault that comes back after a quiet day is reported again.

### What a suppressed crash still does

A crash that does not get the full treatment still runs steps 1 to 4 and does everything that follows step 11 (signal, `⎕OFF`, printing to the session) as before.

It also always writes the log entry, step 5, with the signature and its count and a note that the storm guard held the rest back. The log is the record that the error happened at all, and it is what explains a silence afterwards. The Windows event, step 9, is treated the same way.

It skips the HTML report, the delay before the workspace is saved, the error workspace, the component file and the custom functions.

### What the custom functions get

When `stormGuard` is 1, `Process` adds two variables to `parms` before it calls the custom functions: `stormSignature` and `stormCount`. A notification can then say "this one has occurred 4096 times".

### Where the counts live

In a small text file in the error folder: one line per signature with its count, its `lastFull` and the times of its first and last occurrence, followed by the times of the most recent full treatments, for the ceiling.

All times are UTC. Local time jumps when daylight saving time starts or ends, which would stretch or shrink "the last hour" and the `stormResetAfter` period.

Whenever the guard writes the file, it drops the signatures that have not occurred for `stormResetAfter` hours and the full-treatment times older than an hour. Neither is needed any more, and the file stays small however long the application runs.

A file survives a restart. A crash storm often involves restarts, so counts held in the workspace would reset exactly when they are needed. A human trying to work out what happened can read it, and it needs no configuration, since the error folder already belongs to the handler.

### One process per error folder

The guard requires that only one process writes crash files to a given error folder. `:Hold` serialises crashes within one process, but not across processes: two processes sharing an error folder would read and write the counts file at the same time and lose updates. HandleError's own documentation must state this restriction.

### Fail open

The guard has a trap of its own. When it cannot read or write its file, the crash gets the full treatment and the log entry says that the guard failed. Without that trap, the outer trap around steps 1 to 11 would swallow the guard's failure and silence crash reporting, with nothing anywhere to say so.

The file is small and is read and written once per crash, so the guard does not lengthen the hold measurably.

### Parameters

Three new parameters in `CreateParms`:

| Parameter | Default | Meaning |
|---|---|---|
| `stormGuard` | 0 | Boolean. 1 switches the guard on. With 0, `Process` behaves as it does today and the other two are ignored. |
| `stormCeiling` | 10 | The maximum number of crashes within one hour that get the full treatment, across all signatures. |
| `stormResetAfter` | 24 | Hours after which a signature that has not occurred starts again at 1. |

The application decides where the values come from. The Tatin server, for example, will read `StormGuard` from the `[TRAP]` section of its INI file and set `#.ErrorParms.stormGuard` accordingly.

## Alternatives considered

| Option | Why not |
|---|---|
| Leave it as it is | It remains the default, `stormGuard←0`, but a recurring fault has already shown what it costs. |
| A rate limit on time alone: at most one full treatment per *n* minutes | It suppresses the wrong crash. A storm is one error, and a *new* error during a storm is news, which this option holds back for up to *n* minutes. |
| The ceiling on its own | A rate limit with a bigger bucket. It drops crashes that nothing else would have dropped. |
| A digest: one summary per interval | Needs a scheduled sender, which only runs while the application is alive, and a bad crash is exactly what stops it. |
| A guard in the application's custom function | Runs after the log entry, the workspace and the component file are written, so it cannot hold those back. |
| Pruning old artefacts after the fact | Writes files in order to delete them, and the expensive part is paid in full before the pruning happens. |
| Switching the error workspace off | Throws away the best evidence for a rare crash. It is still the right setting for a threaded application, which cannot save one anyway. |
| Counts held in the workspace | Lost on every restart. |
| Counts derived from the application log | Reads and parses a growing file inside the hold. |

## Related, but separate

`parms.signal` lets the handler ask for a restart instead of carrying on, and the Tatin server relies on it. When one signature fires fifty times in ten minutes, the application is broken, and a restart may be a better answer than another report. Combining the guard with a restart policy is a separate decision. It would make the thresholds less critical, since nothing would get the chance to fire a thousand times.

## Open

- The defaults for `stormCeiling` and `stormResetAfter` are first guesses. Before the release they should be checked against what a real storm looked like.


