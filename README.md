# HandleError


`HandleError` is a member of the APLTree library. The library is a collection of classes etc. that aim to support the Dyalog APL programmer. Search GitHub for "apltree" and you will find solutions to many every-day problems Dyalog APL programmers might have to solve.


## Overview 

`HandleError` offers a method `Process` which is useful to handle application errors on a general level.

It has the following goals:

 * Create a namespace `#.Crash` and populate it with variables providing all sorts of information potentially important for analyzing the error. This namespace is also saved in a component file by default. This is mainly for application that cannot save an error workspace because threads are used.
 * Creates an HTML page with essential information regarding the error.
 * Attempt to save an error workspace. In order to achieve that all running threads are killed; the main thread will always survive anyway. All options can be changed by setting variables in the namespace returned by `CreateParms` which can be passed as right argument to the main function `Process`. For defaults just pass an empty vector to `Process`. 

You can make `Process` execute your own code by setting these two names in the parameter space:

 * `logFunction` is supposed to be the name of a monadic function that returns a shy result. The right argument is expected to process simple strings as well as vectors of strings.
 * `customFns` is supposed to  be the name of a monadic function that returns a shy result. It can be used to do anything you like. The right argument is the namespace with the configuration settings, as returned by `CreateParms`. A typical application for this is sending an email to notify certain users  about the crash.

However, if the log function is an instance method of a class then one cannot specify a name: it needs to be a reference. This can be achieved by specifying just the name of the method on `logFunction` but a reference pointing to an instanced or a namespace on `logFunctionParent`. Accordingly the same holds true for `customFns` and `customFnsParent`.

## Precondition 

By default the variable `errorFolder` in a parameter space created by a call to `CreateParms` carries "Errors\". That means that `HandleError` attempts to write any files into this folder relative to whatever the current directory is. That means you have to make sure that this directory exists, otherwise it won't be able to create any file at all.

## Examples 

This example sets error trapping for an application so that `HandleError.Process` uses defaults (empty right argument): 
```
      ⎕TRAP←(405 'E' '#.HandleError.Process ⍬')
```

This example uses the internal function `SetTrap` for setting `⎕TRAP`:
```
      ⎕TRAP←#.HandleError.SetTrap ⍬
```

`#.HandleError.SetTrap` checks whether the right argument is empty (early stage) or a parameter space and whether it is running under a development EXE or a runtime EXE and acts accordingly.

This example calls `HandleError.CreateParms` in order to create a command namespace and then modifies some of them. Note the usage of `⎕SHADOW` in order to make sure that the `⎕TRAP` statement can always "see" the right argument: it's kind of a global but at the same time it is local in the sense that it will disappear when the application finishes:

```
      #.⎕SHADOW'HandleErrorParms'
      #.HandleErrorParms←#.HandleError.CreateParms
 #.HandleErrorParms.∆List
 addToMsg                    
 createHTML                1 
 customFns                   
 customFnsParent
 errorFolder         Errors\ 
 logFunction                 
 logFunctionParent                 
 off                       1 
 returnCode                1 
 saveCrash                 1 
 saveErrorWS               1 
 signal                    0 
 trapInternalErrors        1 
 windowsEventSource          

      ⎕TRAP←(405 'E' '#.HandleError.Process #.HandleErrorParms')
```

By editing or `⎕CR`ing the following function you can see what the command space contains together with a description.

```
      ps←#.HandleError.CreateParms
      ⎕CR 'ps.∆Reference'
```

That would result in something similar to this:

createHTML

: Boolean that defaults to 1. A 0 suppresses the creation of the HTML file

customFns 

: Name of a monadic function to be executed by "Process". Useful to send an email, for example. Either specify a fully qualified name of specify `customFnsParent` as well.

customFnsParent 

: Reference pointing to a namespace or an instance where `customFns` can be found.

errorFolder 

: Folder that keeps the component file (crash), the HTML page and the error WS

logFunction

: Name of the logging function to be used. Either specify a fully qualified name of specify `logFunctionParent` as well.

logFunctionParent 

: Reference pointing to a namespace or an instance where `logFunction` can be found.

off 

: By default this function executes `⎕OFF` with ""returnCode"" when in Runtime.  A 0 suppressed this.

returnCode

: The return code passed on to `⎕OFF`

saveCrash

: Boolean that defaults to 1. A 0 suppresses the creation of #.Crash & a  component file saving this NS

saveErrorWS 

: Boolean that defaults to 1. A 0 suppresses the creation of crash
 
signal

: When "off" is 0 and "signal" is not 0 then "signal" is signaled by "Process". Can be used for a restart attempt.

trapInternalErrors

: By default all internal errors are trapped: "Process" should never crash in  an app.

windowsEventSource

: Name of the Windows Event Log to write to. Ignored when empty.

addToMsg 

:      Will be added to log file entries as well as Windows Event Log messages. Mainly for test cases