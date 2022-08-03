# HandleError


`HandleError` is a member of the APLTree library. The library is a collection of classes etc. that aim to support the Dyalog APL programmer. Search GitHub for "apltree" and you will find solutions to many every-day problems Dyalog APL programmers might have to solve.

**Note:** `HandleError` releases are published as [Tatin](https://tatin.dev "Link to the principal Tatin Registry") packages, see there.

## Overview 

`HandleError` offers a method `Process` which is useful to handle application errors on a general (or application) level.

It has the following goals:

 * Create a namespace `crash` and populates it with variables providing all sorts of information potentially important for analyzing the error. This namespace is also saved in a component file by default. This is mainly for applications that cannot save an error workspace because threads are used.
 * Create an HTML page with essential information regarding the error.
 * Attempt to save an error workspace. In order to achieve that all running threads are killed; the main thread will always survive anyway. 

All options can be changed by setting variables in the namespace returned by `CreateParms` which can be passed as right argument to the main function `Process`. For defaults just pass an empty vector to `Process`. 

You can make `Process` execute your own code by setting `logFunction` and/or `customFns`.

## Examples 

This example sets error trapping for an application so that `HandleError.Process` uses defaults (empty right argument): 
```
      ⎕TRAP←(405 'E' '#.HandleError.Process ⍬')
```

This is typically done at a very early stage of an application, when we don't know yet where to save stuff etc.

This example uses the internal function `SetTrap` for setting `⎕TRAP`:

```
      ⎕TRAP←#.HandleError.SetTrap ⍬
```

`#.HandleError.SetTrap` checks whether the right argument is empty (early stage) or a parameter space and whether it is running under a development EXE or a runtime EXE and acts accordingly.

This example calls `HandleError.CreateParms` in order to create a command namespace and then modifies some of them. Note the usage of `#.⎕SHADOW` in order to make sure that the `⎕TRAP` statement can always "see" the right argument: it's kind of a global but at the same time it is local in the sense that it will disappear when the function quits that contains these lines:

```
      #.⎕SHADOW'HandleErrorParms'
      #.HandleErrorParms←#.HandleError.CreateParms
⍝ The variables and their settings:
 addToMsg              
 checkErrorFolder    1 
 createHTML          1 
 customFns             
 customFnsParent       
 enforceOff          0 
 errorFolder           
 logFunction           
 logFunctionParent     
 off                 1 
 returnCode          1 
 saveCrash           1 
 saveErrorWS         1 
 saveVars            1 
 signal              0 
 trapInternalErrors  1 
 trapSaveWSID        1 
 windowsEventSource    

      ⎕TRAP←(405 'E' '#.HandleError.Process #.HandleErrorParms')
```

A detailed documentation is available via

```
]adoc HandleError
```
