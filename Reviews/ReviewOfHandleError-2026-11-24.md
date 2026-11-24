# Review of HandleError 5.2.0

Reviewed on 2026-11-24: branch `dev` at commit 2ba53b9 ("Version 5.2.0"). This merges two reviews that were carried out independently of each other. Both covered every function in `APLSource/HandleError`, the API covers, `ADOC_Doc`, `History`, `README.md`, `Misc/CrashStormGuard.md`, the package and Cider configuration, and the leading comments of the test cases.

Findings marked "verified" were reproduced with `dyalog.exe -script` in 18.2, 20.0 and 21.0, with identical results. "Both" in the summary means that the two reviews found it separately. Everything else comes from reading the code. The test suite was not run and nothing in the project was changed.

Line numbers follow the APL convention: the header is line 0, so `Process[26]` is the 27th line of the file.

## Summary

The code does what the documentation says in the normal case, and the storm guard's rule, file format and fail-open path are sound. Most of the serious findings are on paths where something else has already gone wrong.

| # | Finding | Kind | Verified | Found by |
|---|---------|------|----------|----------|
| 1 | A `signal` that nothing catches makes `Process` run again and again for one error | Bug, missing documentation | yes | both |
| 2 | `SaveCrash` loses the crash when a saved variable contains an instance, a class or a GUI object | Bug | yes | both |
| 3 | A failure in the first steps of `Process` also skips the log entry, the Windows event and the custom function | Bug | read | both |
| 4 | "customFns failed" piles up in the application's own `addToMsg` | Bug | yes | one |
| 5 | An error in `Process` outside its own `:Trap` re-enters `Process`; a parameter space with a missing field triggers it | Robustness | mechanism | one |
| 6 | `SetTrap` returns two different shapes; catenating to it fails in development only | Bug | yes | both |
| 7 | The storm guard mutes a slowly recurring error for weeks | Design | read | one |
| 8 | A log function that fails or cannot be found leaves no trace | Bug | read | both |
| 9 | What happens after `Process` returns is not documented | Missing documentation | yes | both |
| 10 | The zip in `Dist/` does not match the committed source | Release | yes | one |

Suggested order of work before a release: rebuild `Dist/` (10), document the trap that `signal` needs and add the guard (1), retry `SaveCrash` without `Vars` (2), protect the two records (3), stop writing into the caller's `addToMsg` (4), merge the parameter space with the defaults (5), fix the shape of `SetTrap`'s result (6). Finding 7 is a design decision.

## 1. Serious findings

### 1.1 A `signal` that nothing catches loops

Verified. `Process[48]` executes `⎕SIGNAL parms.signal`. A `⎕TRAP` stays in force while its own `E` expression runs, so the event is raised on the failing line again, the trap search finds `0 'E' '…Process…'` again (0 covers 1 to 999), and `Process` runs a second time. With `⎕TRAP←0 'E' '#.HE.Process ''#.P'''` and `P.signal←911`, one `1÷0` made `Process` run until a custom function set `signal` back to 0. Without that it does not stop until the workspace is full, and with the defaults every round writes a workspace, a component file and an HTML page and runs the custom function. From the second round on `LastErrorNumber` is the signal number and the original error is gone from the report.

It only works when a trap for that event is found before HandleError's. `Test_13` does exactly that: `⎕TRAP←((501 'E' '→∆CarryOn')…),1 SetTrap '#.HandleErrorParms'`. A `:Trap 501` works too, but only when it sits lower in the stack than the `⎕TRAP` that `SetTrap` filled. `SetTrap` returns only the `0 'E'` item, so it offers no help.

The documentation of `signal` is one sentence: "This can be used for a restart attempt." It should say which event numbers are sensible (500 to 999; `⎕SIGNAL 1001` is a DOMAIN ERROR), that the application's trap for the event must come first, with an example, and that `off` is ignored when `signal` is not 0.

Suggestion in the code: a guard in `Process` that does not handle the error again when `qdmx.EN≡parms.signal`.

### 1.2 `SaveCrash` loses the crash when `Vars` cannot be serialised

Verified. `SaveVisibleVars` collects name class 2, and an array of instances is name class 2. `⎕FAPPEND` rejects a namespace with "Object oriented namespace cannot be serialised" as soon as it contains an instance of a class, a reference to a class or a GUI object, also when that sits inside an array. One such variable anywhere in the failing function's namespace is enough.

When `crash ⎕FAPPEND tno` at `SaveCrash[4]` fails, line 8 appends `qdmx.DM` instead, and that `qdmx` describes the failure of `⎕FAPPEND`, not the crash. The component file then holds three lines saying that `SaveCrash[4]` had a DOMAIN ERROR. `EN`, the stack, `Message` and all the healthy variables are lost. The comment on line 4 expects a WS FULL; this cause is more common.

References to plain namespaces, to `#`, to `⎕NULL` and a .NET object were accepted.

For a threaded application, which cannot save a workspace, the component file is the main evidence, and the HTML page is then all that is left.

Suggestion: when the first `⎕FAPPEND` fails, try again without `crash.Vars` (or with only the variables that can be written one by one) and record the reason in `crash`, the way `SaveFailed` does for the workspace.

### 1.3 A failure in the first steps skips the two records

The log entry and the Windows event were moved in front of `⎕SAVE` in 5.2.0 so that they exist whatever happens later. They are still behind four steps that have no trap of their own: `CheckErrorFolder`, the first lines of `CreateCrashNamespace`, `CreateFilename` and the `⎕TKILL` line. When one of those fails, the outer `:Trap` at `Process[26]` takes over and everything up to `:EndTrap` is skipped, including `WriteToLogFile`, `WriteToWindowsEvents` and the custom function, although none of them needs the error folder. The application then `⎕OFF`s with no record anywhere that it crashed.

The likely trigger is `3 ⎕MKDIR` in `CheckErrorFolder`: no permission, an invalid path, a full or unreachable drive. That is the case in which the two records would be the only ones. `Test_31` covers the situation (a file in the way) but asserts only that `signal` still happens.

Suggestion: give `CheckErrorFolder` a trap of its own, carry on without a folder, and put the problem into the note that goes to the log. The later steps already cope with a folder that is not there.

### 1.4 "customFns failed" accumulates in `addToMsg`

Verified. `ExecuteCustomFns[15]` appends the failure note to `parms.addToMsg`. `parms` is a reference to the application's parameter space, so the note stays there after `Process` has finished. Three crashes with a custom function that cannot be found leave three identical "customFns failed" lines in `addToMsg`.

In an application that carries on after a crash (`signal`, or `off←0` in a server) each later crash reports the notes of all earlier ones, in the log file, in the Windows event and in the HTML report. It also turns a simple character vector into a nested one behind the application's back.

Suggestion: keep the note out of the user's parameter, for example in `crash` or in a variable of its own that `Process` resets on entry.

### 1.5 An error in `Process` outside its traps re-enters `Process`

The mechanism is the one verified for 1.1. These lines are not protected: `Process[24]`, `[25]`, the evaluation of `parms.trapInternalErrors` at `[26]`, and everything from `[47]` on.

The realistic trigger is a parameter space that was not created by `CreateParms`, or one created by an older version and kept in a saved workspace, so that `trapInternalErrors`, `signal`, `off` or `stormGuard` is missing. `Process` does not merge what it gets with the defaults. A missing `trapInternalErrors` gives a VALUE ERROR at `[26]`, the application's trap calls `Process` with the same name, and so on.

Suggestion: create the defaults first and copy the caller's values over them. That also makes a parameter space written against 5.1.0 safe to use with 5.2.0.

A minor case of the same kind, verified: called when no error is pending (`⎕DMX.DM` is empty), `1⊃parms.LastError` at `Process[51]` is an INDEX ERROR, and line 57 has the same expression. By then the crash files for the non-error have been written. It only happens in development and only when somebody calls `Process` directly. `StormSignature` already allows for a short `DM`.

### 1.6 `SetTrap` returns two shapes

Verified:

```
      HandleError.SetTrap ⍬        ⍝ development, no force
 0 1000  S                         ⍝ depth ¯2: one trap item, not enclosed
      1 HandleError.SetTrap ⍬      ⍝ forced, or runtime
  0 E #.HE.Process ⍬               ⍝ depth ¯3: a one-item vector of trap items
```

Both can be assigned to `⎕TRAP` as they are. Catenating a trap item, as `Test_13` does and as 1.1 requires, works in runtime and fails with a DOMAIN ERROR in development:

```
⎕TRAP←(⊂999 'C' '→0'),HandleError.SetTrap ⍬
```

So the line a developer needs for `signal` breaks where it gets written and tried first. `SetTrap[8]` should read `r←⊂(0 1000)'S'`.

### 1.7 The storm guard mutes a slow, steady error

The backoff counts occurrences and knows nothing about time, except for the reset after `stormResetAfter` quiet hours. An error that occurs once an hour is no storm, yet it gets the full treatment at occurrences 512 and 1024 and at none of the 511 in between, which is three weeks. An error that occurs once a day, slightly less than 24 hours apart, never resets: after the 16th day the next reports come on days 32 and 64. If the gap is slightly more than 24 hours, every occurrence is reported. The outcome therefore depends on a few minutes either way.

The log entry and the Windows event are still written, so nothing is lost entirely, but the custom function (the email) is not called.

Suggestion: also grant the full treatment when the signature's last full treatment is older than some period, an hour for example. That needs one more time per signature in `StormGuard.txt`. The design document does not discuss this case.

### 1.8 A failing log function leaves no trace

`WriteToLogFile[23]` has an empty `:Else`. When the log function crashes, or when `logFunction` has no dot and `logFunctionParent` is empty ("Cannot find the log function"), nothing is recorded anywhere. A custom function with the same mistakes is reported in the HTML page (`Test_37` to `Test_39`). A failing `WriteToWindowsEvents`, `WriteHtmlFile` and `SaveVisibleVars` are swallowed in the same way, and a `windowsEventSource` that is not registered leaves no trace at all.

The report of a failing custom function has gaps too. `SaveCrash` runs before `ExecuteCustomFns`, so the component file never contains the note, and the log is written earlier still. With `createHTML←0` the failure is invisible.

Suggestion: collect internal problems in one place in `crash` (log function, custom function, storm guard, Windows event, save of the workspace) and write the component file after the custom function, as the HTML page already is.

### 1.9 What happens after `Process` returns

This is not explained anywhere, and it decides how an application has to be built around HandleError.

- Verified: when the `E` expression returns without branching, the failing function stays suspended on the stack. In development that follows the message "HandleError.Process caught DOMAIN ERROR" and is what one wants. In a runtime with `off←0` and `signal←0` there is nothing to suspend into; `Test_12` suggests the outcome is a CONTINUE workspace.
- The documentation of `off` says "A 0 suppresses this" and stops there. The ADOC text recommends `killThreads←0` "if you intend to let the application continue", but not how the failing thread ends or what the application has to provide. The only way to continue is `signal` together with a trap of the application's own (see 1.1).
- The order in `Process[47]` to `[59]` is `signal` first, then `off`, then `enforceOff`. `enforceOff` is documented as "you can force `HandleError` to `⎕OFF` no matter whether it's a runtime EXE or not", yet with `off←0` and `enforceOff←1` there is no `⎕OFF`, while threads are killed in development and nothing is printed to the session. With `signal` not 0, neither of the other two matters.
- The message printed in development shows only the first line of `DM`.

A short section "Carrying on after an error" would cover this and 1.1 together.

### 1.10 The zip in `Dist/` is older than the source

Verified. `Dist/aplteam-HandleError-5.2.0.zip` was built at 07:56 and contains `ADOC_Doc.apla` as saved at 07:48 (7379 bytes). The source was saved again at 07:58 (7364 bytes) and committed with "Version 5.2.0". The zip still has the earlier wording of the storm guard paragraph:

```
zip:    ... and no more than `stormCeiling` crashes get it within `stormResetAfter` hours.
source: ... and no more than `stormCeiling` crashes get it within one hour.
```

The source is right. Every other file in the zip is identical to the working copy. The package needs a rebuild before it is published.

## 2. Smaller bugs

1. `CheckErrorFolder[6]` parses the command line of a stand-alone EXE with `'"'∊⍵`, which looks for a quote anywhere. Verified: `C:\Apps\app.exe "arg one" x=1` yields `arg one`, so the error folder becomes `%LOCALAPPDATA%\arg one\Errors`. The test must be whether the first character is a quote.
2. `CreateFilename[3]` takes the name from `crash.WSID`, while `CheckErrorFolder` falls back to the EXE name or `ERROR_WS`. Verified: in a clear workspace the files are called `CLEAR WS_20261124081809080.html`, with a blank, and with an empty `⎕WSID` they start with a bare underscore. Both functions should use the same fallback.
3. `parms.∆ERROR_WS` and `parms.crashFilename` are set at `Process[30]` and `[31]` before anything is written. `∆ERROR_WS` is documented as the name of the error workspace, but it is set when `saveErrorWS` is 0, when the save failed and when threads made it impossible. With the storm guard holding a crash back, neither name refers to any file, and code that runs after a `signal` cannot tell. Suggestion: a Boolean such as `stormFull` next to `stormSignature` and `stormCount`.
4. `crash.SaveFailed` is set by `SaveErrorWorkspace` but is shown nowhere: not in the HTML page, the log or the Windows event. When `saveCrash` is 0 it is lost. Also, `SaveErrorWorkspace[11]` stores `qdmx.Message`, which may be empty when the reason is in `EM` only.
5. `SaveCrash` declares a shy result `r` and never assigns it, so a caller that uses the result gets a VALUE ERROR. Nothing does at present.
6. The API cover `ReportErrorToWindowsLog←{##.ReportErrorToWindowsLog ⍵}` turns the shy result into a displayed one. The `Process` cover uses `1:shy←` for exactly that reason.
7. `ReportErrorToWindowsLog` is public but does not check the operating system; on Linux and macOS it fails inside `⎕NEW EventLog`.
8. When the default folder is used and `LOCALAPPDATA` is not defined, `CheckErrorFolder[12]` builds `/Foo\Errors`, a folder in the root of the current drive.
9. A relative `errorFolder` that contains a separator (`sub/Errors`) is not made absolute by `CheckErrorFolder[18]`, whereas `Errors` is. The Windows event then says "Details can be found in sub\Errors".
10. `logFunction` and `customFns` are executed with `⍎` as they come. A niladic function would be called at that point and the next line would fail in a confusing way. A check with `11 ⎕ATX` before `⍎` would turn that into a clear message.
11. `⎕WSID` and the file name in `<title>` and `<h1>` of the HTML page are not escaped. A workspace called `R&D` is enough.

## 3. Inconsistencies

### The parameter space

1. Verified: `CreateParms[32]` sets `⎕IO←0 ⋄ ⎕ML←3` before `r←⎕NS''`, and the new namespace inherits both. `parms.(⎕IO ⎕ML)` is `0 3`, and `parms.(1⊃LastError)` returns the second line of the error message. Anybody who evaluates an expression inside the parameter space gets this, and a custom function receives that space as its argument. `∆List` sets its own `⎕IO`, so the function does not seem to need it. If it is a leftover, `1 1` would match the package; if it is wanted, it deserves a comment and a line in the documentation.
2. `SetTrap` localises `⎕ML←0`, `Process` uses 1 and `CreateParms` 3.
3. `Process[25]` injects `EN`, `ENX`, `EM` and `Message` into the parameter space. Nothing in HandleError reads them and no documentation mentions them.
4. `CheckErrorFolder` changes `parms.errorFolder` but returns `⍬`, and `Process` calls it without using a result. `ExecuteCustomFns` and `StormGuard` follow the `parms←Fn parms` convention.

### `SetTrap` and `Process`

1. The name must be fully qualified. `SetTrap[17]` tests `⎕NC` inside HandleError's own namespace, and `Process[13]` executes the name there. Verified: `'Parms'` passed from the namespace that contains `Parms` is rejected with "not a named namespace", although it is one. The documentation says only "the name (but not a ref!) of a parameter space". The same applies to dotted names in `logFunction` and `customFns`.
2. `SetTrap[11]`: a variable holding `⎕NULL` has name class 2 (verified), so the `:AndIf` can never decide anything.
3. `SetTrap[16]`, the test for `[]` in the formatted argument, is redundant. A reference is rejected at line 12, and `⎕NULL` or a name like `#.[Namespace]` is rejected by line 17 as well, with the same message (verified that both are rejected).
4. The comment on `SetTrap[19]`, "At a very early stage error trapping does not make sense", sits on the branch that sets a trap.
5. `Process[18]`, `:OrIf 0=≢parms`, is dead: when the line is reached, `parms` is a reference and `≢` gives 1.
6. `⎕DL 0.2` at `Process[39]` has no comment saying what it waits for.

### Threads

1. `killThreads` kills the threads even when no workspace will be saved (`saveErrorWS←0`, or the storm guard holding back). That is defensible before `⎕OFF`, but the documentation gives saving the workspace as the only reason.
2. Threads that crashed at the same time and are waiting at `:Hold` are killed too, so their crashes are never reported.

### Names of folder and files

1. `CheckErrorFolder` normalises the folder and `CreateFilename[3]` appends `/`, so on Windows `crashFilename` has both separators: `C:\...\err1/CLEAR WS_2026...`. The same string goes into `∆ERROR_WS`.
2. `crash.addedMsg` is the only lower-case name among `EN`, `DM`, `XSI` and the rest. It is also a copy taken before the custom function runs.

### The log entry and the Windows event

1. The log function gets `LastError`, which is `DM`, and the error number. `⎕DMX.Message` and `OSError` appear only in the HTML page and the component file. For a file or network error those two often hold the actual reason, and a crash the storm guard holds back has no HTML page.
2. The log entry never says where the crash files are. The Windows event has the folder but not the `DM` lines.
3. In the Windows event the error number is labelled `RC=` (`WriteToWindowsEvents[9]`), which reads like the `returnCode` parameter.
4. `WriteToWindowsEvents[10]` tests `0<≢parms.errorFolder`, which is always true after `CheckErrorFolder`, and "Details can be found in" is written even when `saveCrash`, `saveErrorWS` and `createHTML` are all 0.

### The HTML page

1. `crash.TID` is collected but not shown, although `⎕TNUMS` is. In a threaded application the number of the failing thread belongs on the page.
2. The page has no date or time apart from the file name, and shows neither the storm guard's signature and count nor `SaveFailed`, `crash.Trap`, `crash.AN` or HandleError's own version.
3. `AssembleHTML[22]` reduces `⎕WSID` to its last part only when the path uses `/`; a Windows path is shown in full.
4. `AssembleHTML` separates lines with a bare CR (`⎕UCS 13`), so the file has neither Windows nor Unix line endings. Browsers cope; most editors show one long line.
5. The two leading blanks of each label end up inside `<b>`.

### Log function and custom function

1. `customFns` is documented as a "fully qualified name", `logFunction` just as a "name", and both are resolved the same way. When the name contains a dot, the parent parameter is ignored without notice. A name without a dot and without a parent can never work.
2. The sentence about the parents, "Needed only in case the parent is a class instance since it may use dotted syntax", is hard to follow. What it means is: give the bare function name and put the instance into the parent parameter.

### Storm guard

1. Verified: the second line of `DM` carries no namespace or class, so the signature is `11 Run[5]` for a `Run` in any namespace. The design note mentions approximations but not this one. `crash.XSI` and `crash.LC` are already available when the guard runs and would give the qualified name.
2. `stormCeiling←0` holds back every crash, including the first; `stormResetAfter←0` gives every crash the full treatment. Both verified, both follow from the rules, neither is documented.
3. The leading comment of `StormGuard` refers to `Misc/CrashStormGuard.md`, which is not part of the package.

### Wording

1. `SaveErrorWorkspace[2]`: "Otherwise we we can't a WS anyway".
2. `ADOC_Doc` and `CreateParms` still speak of "the method `Process`" and "the `HandleError` method", from the time this was a class.

## 4. Documentation

### `CreateParms` (the parameter table)

1. `saveCrash`: "A 0 suppresses the creation of `crash` & a component file". Only the component file is suppressed; `crash` is always built because the HTML page needs it.
2. `saveVars`: "Is ignored when `saveCrash` is 0". It is not: `CreateCrashNamespace[18]` looks at `saveVars` alone, and the copies end up in the saved workspace.
3. `addToMsg`: "Mainly for test cases" is out of date, since `ADOC_Doc` recommends it for custom functions. The table also does not say that a list of texts is accepted since 5.1.0, or that the text goes into the HTML page and into `crash.addedMsg`.
4. `off`, `enforceOff`, `signal`: see 1.1 and 1.9. The defaults of `returnCode`, `signal`, `customFns` and `logFunction` are not stated.
5. `logFunction`: says only "Name of the logging function". What it is called with (several calls, a character vector or a vector of them) is in ADOC only.
6. `errorFolder`: the rules cover a workspace called "Foo". The two special cases are missing: `ERROR_WS` for a clear workspace and the name of the EXE when `⎕WSID` is empty. The description should also mention `StormGuard.txt`, and that `Process` overwrites `errorFolder` with the resolved path.
7. `windowsEventSource`: nothing says that the source must exist.
8. The table is alphabetical except for `addToMsg` at the end.
9. `∆List` is not documented anywhere.

### `ADOC_Doc`

1. The paragraph "Set this to 0 if you intend to let the application continue" belongs to `killThreads`, but since the 5.1.0 paragraph was inserted above it, "this" reads as `enforceOff`.
2. "kill all running threads but the main thread": the thread handling the error survives as well, and when that is not the main thread no workspace is saved. `README.md` says it correctly.
3. "just before attempting to save the workspace": the threads are killed before the storm guard, the log entry and the Windows event.
4. "all variables visible at the moment of the crash": `SaveVisibleVars` saves name class 2 in one namespace, the one the failing function was running in. References (name class 9) and the variables of callers in other namespaces are not included.
5. The list of variables a custom function finds in the parameter space names `crashFilename`, `LastError` and `LastErrorNumber`. Missing are `∆ERROR_WS` (mentioned only in the comment of `Process`, and holding the same value as `crashFilename`), `EN`, `ENX`, `EM` and `Message`. `stormSignature` and `stormCount` are mentioned in the storm guard section only.
6. "The last two steps are necessary because…" follows a list of variables, not steps. The line and the two after it start with one blank instead of two and so fall out of the list item.
7. "`customFns` shall not return a result. If it does it must be shy": a non-shy result is merely displayed. `CreateParms` calls the function monadic; ADOC says "must accept a right argument". One wording would do.
8. Nothing describes the artefacts as a set: `<name>_<YYYYMMDDhhmmssmmm>` with the extensions `.html`, `.dws` and `.dcf`, local time in the name, the `crash` namespace as the first component, and `StormGuard.txt` with UTC times.
9. Windows Event Log: the section names .NET and the bridge DLLs as requirements and says a runtime application needs "you taking action" without saying which. It should add that the event source has to exist: creating one needs administrator rights, and without it `WriteEntry` fails, silently inside `Process`. `WriteWindowsLog_[7]` has a comment admitting that this is not checked.
10. `⎕USING` names `system.dll`, the .NET Framework assembly. `⎕NEW EventLog` works with the Framework bridge (verified). Whether `EventLog` is found under .NET (Core) was not checked.
11. The storm guard section does not say that a crash that is held back still kills threads and still ends in `signal` or `⎕OFF`, that the custom function is not called for it, or that the guard fails open. All of that is in the design note, which users of the package do not get.
12. `ReportErrorToWindowsLog` is public, but its argument, `(appName message)`, is described only in the internal function.

### `README.md`

1. The list of parameters lacks `killThreads` (there since 4.1.0), `stormCeiling`, `stormGuard` and `stormResetAfter`.
2. The storm guard, the Windows Event Log and `ReportErrorToWindowsLog` are not mentioned.
3. "all running threads are killed by default" does not mention the exception in development since 5.1.0.

### `Misc/CrashStormGuard.md`

1. "This describes HandleError 5.1.0", but the table shows the Windows event as step 6, straight after the log entry. That order came with 5.2.0, and the paragraph below the table says so itself ("Up to 5.1.0 the Windows event came after `SaveCrash`").
2. The document is still written as a proposal ("A design for…", "Three new parameters", "will read"), and the item under "Open" (check the defaults against a real storm) is unresolved although 5.2.0 is in `History` and built.
3. It says "custom functions" throughout; there is one `customFns`.

### Leading comments of `Process`

1. "`⎕OFF`s in runtime or if `enforceOff` is 1" leaves out the two conditions that prevent it, `off←0` and a non-zero `signal`.
2. The right argument is not described: empty, a reference or a fully qualified name. `SetTrap` insists on a name; `Process` also accepts a reference.
3. Neither the shy result nor the fact that the parameter space is modified is mentioned. The left argument overwrites `parms.signal` in the caller's parameter space for good.

## 5. Tests

The suite covers the regular paths well. There is no test for these cases, all of which are findings above:

- a `signal` nobody catches
- the shape of `SetTrap`'s result in development without `force`
- `off←0` together with `enforceOff←1`
- a variable that `⎕FAPPEND` rejects
- a log function that fails
- the log entry when the error folder cannot be created (`Test_31` checks `signal` only)
- a parameter space with a missing field
- a storm file that cannot be written

## 6. Checked and found in order

- Errors under `⍎` get a proper signature (`11 Exe[2]`); the `⍎` prefix is on the first line of `DM` only.
- An error signalled with `⎕SIGNAL` is attributed to the calling line (`11 CallSig[2]`), which is the useful place.
- `0 ⎕TKILL` from a thread does not harm the main thread.
- `⎕XSI`, `⎕LC` and `⎕RSI` have the same length inside an `E` trap reached through `⍎`.
- `UtcNow`, `SecondsToIso` and `IsoToSeconds` round-trip in 18.2, the oldest version installed. 18.0, the declared minimum, is not installed, so that one is unverified.
- `StormRule` gives 1, 2, 4, 8… and recovers an occurrence the ceiling blocked, as the design says.
- `Caller` finds the right namespace through `⎕RSI`, with and without the API cover.
- `]adoc HandleError` reaches `ADOC_Doc` and the parameter table for a package loaded with Tatin: ADOC recognises the API and looks behind it (stated by Kai, not tested here).

## 7. Questions

1. Is `⎕IO←0 ⋄ ⎕ML←3` for the parameter space deliberate (section 3, "The parameter space")?
2. Should `Process` protect itself against the loop in 1.1, or is documenting the requirement enough?
3. Should the storm guard report a slow, steady error more often (1.7)?


