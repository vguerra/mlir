# Writing a Pass

[TOC]

Passes represent the basic infrastructure for transformation and optimization.
This document provides a quickstart to the pass infrastructure in MLIR and how
to use it.

See [MLIR specification](LangRef.md) for more information about MLIR and its
core aspects, such as the IR structure and operations.

See [MLIR Rewrites](QuickstartRewrites.md) for a quick start on graph rewriting
in MLIR. If your transformation involves pattern matching operation DAGs, this
is a great place to start.

## Pass Types

MLIR provides different pass classes for several different granularities of
transformation. Depending on the granularity of the transformation being
performed, a pass may derive from [FunctionPass](#functionpass) or
[ModulePass](#modulepass); with each requiring a different set of constraints.

### FunctionPass

A function pass operates on a per-function granularity, executing on
non-external functions within a module in no particular order. Function passes
have the following restrictions, and any noncompliance will lead to problematic
behavior in multithreaded and other advanced scenarios:

*   Modify anything within the parent module, outside of the current function
    being operated on. This includes adding or removing functions from the
    module.
*   Maintain pass state across invocations of runOnFunction. A pass may be run
    on several different functions with no guarantee of execution order.
    *   When multithreading, a specific pass instance may not even execute on
        all functions within the module. As such, a function pass should not
        rely on running on all functions within a module.
*   Access, or modify, the state of another function within the module.
    *   Other threads may be operating on different functions within the module.
*   Maintain any global mutable state, e.g. static variables within the source
    file. All mutable state should be maintained by an instance of the pass.

To create a function pass, a derived class must adhere to the following:

*   Inherit from the CRTP class `FunctionPass`.
*   Override the virtual `void runOnFunction()` method.
*   Must be copy-constructible, multiple instances of the pass may be created by
    the pass manager to process functions in parallel.

A simple function pass may look like:

```c++
namespace {
struct MyFunctionPass : public FunctionPass<MyFunctionPass> {
  void runOnFunction() override {
    // Get the current function being operated on.
    FuncOp f = getFunction();

    // Operate on the operations within the function.
    f.walk([](Operation *inst) {
      ....
    });
  }
};
} // end anonymous namespace

// Register this pass to make it accessible to utilities like mlir-opt.
// (Pass registration is discussed more below)
static PassRegistration<MyFunctionPass> pass(
    "flag-name-to-invoke-pass-via-mlir-opt", "Pass description here");
```

### ModulePass

A module pass operates on a per-module granularity, executing on the entire
program as a unit. As such, module passes are able to add/remove/modify any
functions freely. Module passes have the following restrictions, and any
noncompliance will lead to problematic behavior in multithreaded and other
advanced scenarios:

*   Maintain pass state across invocations of runOnModule.
*   Maintain any global mutable state, e.g. static variables within the source
    file. All mutable state should be maintained by an instance of the pass.

To create a module pass, a derived class must adhere to the following:

*   Inherit from the CRTP class `ModulePass`.
*   Override the virtual `void runOnModule()` method.

A simple module pass may look like:

```c++
namespace {
struct MyModulePass : public ModulePass<MyModulePass> {
  void runOnModule() override {
    // Get the current module being operated on.
    Module m = getModule();

    // Operate on the functions within the module.
    for (auto func : m) {
      ....
    }
  }
};
} // end anonymous namespace

// Register this pass to make it accessible to utilities like mlir-opt.
// (Pass registration is discussed more below)
static PassRegistration<MyModulePass> pass(
    "flag-name-to-invoke-pass-via-mlir-opt", "Pass description here");
```

## Analysis Management

An important concept, along with transformation passes, are analyses. These are
conceptually similar to transformation passes, except that they compute
information on a specific FuncOp, or Module, without modifying it. In MLIR,
analyses are not passes but free standing classes that are computed lazily
on-demand and cached to avoid unnecessary recomputation. An analysis in MLIR
must adhere to the following:

*   Provide a valid constructor taking an `Operation*`.
*   Must not modify the given operation.

Each of the base Pass classes provide utilities for querying and preserving
analyses for the current operation being processed. Using the example passes
defined above, let's see some examples:

### Querying Analyses

*   FunctionPass automatically provides the following utilities for querying
    analyses:
    *   `getAnalysis<>`
    *   `getCachedAnalysis<>`
    *   `getCachedModuleAnalysis<>`
*   ModulePass automatically provides the following utilities:
    *   `getAnalysis<>`
    *   `getCachedAnalysis<>`
    *   `getFunctionAnalysis<>`
    *   `getCachedFunctionAnalysis<>`

```c++
/// An interesting analysis.
struct MyFunctionAnalysis {
  // Compute this analysis with the provided operation.
  MyFunctionAnalysis(Operation *op);
};

void MyFunctionPass::runOnFunction() {
  // Query MyFunctionAnalysis for the current function.
  MyFunctionAnalysis &myAnalysis = getAnalysis<MyFunctionAnalysis>();

  // Query a cached instance of MyFunctionAnalysis for the current function. It
  // will not be computed if it doesn't exist.
  auto optionalAnalysis = getCachedAnalysis<MyFunctionAnalysis>();
  if (optionalAnalysis)
    ...

  // Query a cached instance of MyModuleAnalysis for the parent module of the
  // current function. It will not be computed if it doesn't exist.
  auto optionalAnalysis = getCachedModuleAnalysis<MyModuleAnalysis>();
  if (optionalAnalysis)
    ...
}

void MyModulePass::runOnModule() {
  // Query MyModuleAnalysis for the current module.
  MyModuleAnalysis &myAnalysis = getAnalysis<MyModuleAnalysis>();

  // Query a cached instance of MyModuleAnalysis for the current module. It
  // will not be computed if it doesn't exist.
  auto optionalAnalysis = getCachedAnalysis<MyModuleAnalysis>();
  if (optionalAnalysis)
    ...

  // Query MyFunctionAnalysis for a child function of the current module. It
  // will be computed if it doesn't exist.
  auto fn = *getModule().begin();
  MyFunctionAnalysis &myAnalysis = getFunctionAnalysis<MyFunctionAnalysis>(fn);
}
```

### Preserving Analyses

Analyses that are constructed after being queried by a pass are cached to avoid
unnecessary computation if they are requested again later. To avoid stale
analyses, all analyses are assumed to be invalidated by a pass. To avoid
invalidation, a pass must specifically mark analyses that are known to be
preserved.

*   All Pass classes automatically provide the following utilities for
    preserving analyses:
    *   `markAllAnalysesPreserved`
    *   `markAnalysesPreserved<>`

```c++
void MyPass::runOn*() {
  // Mark all analyses as preserved. This is useful if a pass can guarantee
  // that no transformation was performed.
  markAllAnalysesPreserved();

  // Mark specific analyses as preserved. This is used if some transformation
  // was performed, but some analyses were either unaffected or explicitly
  // preserved.
  markAnalysesPreserved<MyAnalysis, MyAnalyses...>();
}
```

## Pass Failure

Passes in MLIR are allowed to gracefully fail. This may happen if some invariant
of the pass was broken, potentially leaving the IR in some invalid state. If
such a situation occurs, the pass can directly signal a failure to the pass
manager. If a pass signaled a failure when executing, no other passes in the
pipeline will execute and the `PassManager::run` will return failure. Failure
signaling is provided in the form of a `signalPassFailure` method.

```c++
void MyPass::runOn*() {
  // Signal failure on a broken invariant.
  if (some_broken_invariant) {
    signalPassFailure();
    return;
  }
}
```

## Pass Manager

Above we introduced the different types of passes and their constraints. Now
that we have our pass we need to be able to run it over a specific module. This
is where the pass manager comes into play. The PassManager is the interface used
for scheduling passes to run over a module. The pass manager provides simple
interfaces for adding passes, of any kind, and running them over a given module.
A simple example is shown below:

```c++
PassManager pm;

// Add a module pass.
pm.addPass(new MyModulePass());

// Add a few function passes.
pm.addPass(new MyFunctionPass());
pm.addPass(new MyFunctionPass2());
pm.addPass(new MyFunctionPass3());

// Add another module pass.
pm.addPass(new MyModulePass2());

// Run the pass manager on a module.
Module m = ...;
if (failed(pm.run(m)))
    ... // One of the passes signaled a failure.
```

The pass manager automatically structures added passes into nested pipelines on
specific IR units. These pipelines are then run over a single IR unit at a time.
This means that, for example, given a series of consecutive function passes, it
will execute all on the first function, then all on the second function, etc.
until the entire program has been run through the passes. This provides several
benefits:

*   This improves the cache behavior of the compiler, because it is only
    touching a single function at a time, instead of traversing the entire
    program.
*   This improves multi-threading performance by reducing the number of jobs
    that need to be scheduled, as well as increasing the efficiency of each job.
    An entire function pipeline can be run on each function asynchronously.

As an example, the above pass manager would contain the following pipeline
structure:

```c++
MyModulePass
Function Pipeline
   MyFunctionPass
   MyFunctionPass2
   MyFunctionPass3
MyModulePass2
```

## Pass Registration

Briefly shown in the example definitions of the various
[pass types](#pass-types) is the `PassRegistration` class. This is a utility to
register derived pass classes so that they may be created, and inspected, by
utilities like mlir-opt. Registering a pass class takes the form:

```c++
static PassRegistration<MyPass> pass("command-line-arg", "description");
```

*   `MyPass` is the name of the derived pass class.
*   "command-line-arg" is the argument to use on the command line to invoke the
    pass from `mlir-opt`.
*   "description" is a description of the pass.

For passes that cannot be default-constructed, `PassRegistration` accepts an
optional third argument that takes a callback to create the pass:

```c++
static PassRegistration<MyParametricPass> pass(
    "command-line-arg", "description",
    []() -> std::unique_ptr<Pass> {
      std::unique_ptr<Pass> p = std::make_unique<MyParametricPass>(/*options*/);
      /*... non-trivial-logic to configure the pass ...*/;
      return p;
    });
```

This variant of registration can be used, for example, to accept the
configuration of a pass from command-line arguments and pass it over to the pass
constructor. Make sure that the pass is copy-constructible in a way that does
not share data as the [pass manager](#pass-manager) may create copies of the
pass to run in parallel.

### Pass Pipeline Registration

Described above is the mechanism used for registering a specific derived pass
class. On top of that, MLIR allows for registering custom pass pipelines in a
similar fashion. This allows for custom pipelines to be available to tools like
mlir-opt in the same way that passes are, which is useful for encapsulating
common pipelines like the "-O1" series of passes. Pipelines are registered via a
similar mechanism to passes in the form of `PassPipelineRegistration`. Compared
to `PassRegistration`, this class takes an additional parameter in the form of a
pipeline builder that modifies a provided `OpPassManager`.

```c++
void pipelineBuilder(OpPassManager &pm) {
  pm.addPass(std::make_unique<MyPass>());
  pm.addPass(std::make_unique<MyOtherPass>());
}

// Register an existing pipeline builder function.
static PassPipelineRegistration pipeline(
  "command-line-arg", "description", pipelineBuilder);

// Register an inline pipeline builder.
static PassPipelineRegistration pipeline(
  "command-line-arg", "description", [](OpPassManager &pm) {
    pm.addPass(std::make_unique<MyPass>());
    pm.addPass(std::make_unique<MyOtherPass>());
  });
```

Pipeline registration also allows for simplified registration of
specifializations for existing passes:

```c++
static PassPipelineRegistration foo10(
    "foo-10", "Foo Pass 10", [] { return std::make_unique<FooPass>(10); } );
```

### Textual Pass Pipeline Specification

In the previous sections, we showed how to register passes and pass pipelines
with a specific argument and description. Once registered, these can be used on
the command line to configure a pass manager. The limitation of using these
arguments directly is that they cannot build a nested pipeline. For example, if
our module has another module nested underneath, with just `-my-module-pass`
there is no way to specify that this pass should run on the nested module and
not the top-level module. This is due to the flattened nature of the command
line.

To circumvent this limitation, MLIR also supports a textual description of a
pass pipeline. This allows for explicitly specifying the structure of the
pipeline to add to the pass manager. This includes the nesting structure, as
well as the passes and pass pipelines to run. A textual pipeline is defined as a
series of names, each of which may in itself recursively contain a nested
pipeline description. The syntax for this specification is as follows:

```ebnf
pipeline          ::= op-name `(` pipeline-element (`,` pipeline-element)* `)`
pipeline-element  ::= pipeline | pass-name | pass-pipeline-name
```

*   `op-name`
    *   This corresponds to the mneumonic name of an operation to run passes on,
        e.g. `func` or `module`.
*   `pass-name` | `pass-pipeline-name`
    *   This corresponds to the command-line argument of a registered pass or
        pass pipeline, e.g. `cse` or `canonicalize`.

For example, the following pipeline:

```shell
$ mlir-opt foo.mlir -cse -canonicalize -lower-to-llvm
```

Can also be specified as (via the `-pass-pipeline` flag):

```shell
$ mlir-opt foo.mlir -pass-pipeline='func(cse, canonicalize), lower-to-llvm'
```

## Pass Instrumentation

MLIR provides a customizable framework to instrument pass execution and analysis
computation. This is provided via the `PassInstrumentation` class. This class
provides hooks into the PassManager that observe various pass events:

*   `runBeforePipeline`
    *   This callback is run just before a pass pipeline, i.e. pass manager, is
        executed.
*   `runAfterPipeline`
    *   This callback is run right after a pass pipeline has been executed,
        successfully or not.
*   `runBeforePass`
    *   This callback is run just before a pass is executed.
*   `runAfterPass`
    *   This callback is run right after a pass has been successfully executed.
        If this hook is executed, runAfterPassFailed will not be.
*   `runAfterPassFailed`
    *   This callback is run right after a pass execution fails. If this hook is
        executed, runAfterPass will not be.
*   `runBeforeAnalysis`
    *   This callback is run just before an analysis is computed.
*   `runAfterAnalysis`
    *   This callback is run right after an analysis is computed.

PassInstrumentation objects can be registered directly with a
[PassManager](#pass-manager) instance via the `addInstrumentation` method.
Instrumentations added to the PassManager are run in a stack like fashion, i.e.
the last instrumentation to execute a `runBefore*` hook will be the first to
execute the respective `runAfter*` hook. Below in an example instrumentation
that counts the number of times DominanceInfo is computed:

```c++
struct DominanceCounterInstrumentation : public PassInstrumentation {
  unsigned &count;

  DominanceCounterInstrumentation(unsigned &count) : count(count) {}
  void runAfterAnalysis(llvm::StringRef, AnalysisID *id, Operation *) override {
    if (id == AnalysisID::getID<DominanceInfo>())
      ++count;
  }
};

MLIRContext *ctx = ...;
PassManager pm(ctx);

// Add the instrumentation to the pass manager.
unsigned domInfoCount;
pm.addInstrumentation(
    std::make_unique<DominanceCounterInstrumentation>(domInfoCount));

// Run the pass manager on a module operation.
ModuleOp m = ...;
if (failed(pm.run(m)))
    ...

llvm::errs() << "DominanceInfo was computed " << domInfoCount << " times!\n";
```

### Standard Instrumentations

MLIR utilizes the pass instrumentation framework to provide a few useful
developer tools and utilities. Each of these instrumentations are immediately
available to all users of the MLIR pass framework.

#### Pass Timing

The PassTiming instrumentation provides timing information about the execution
of passes and computation of analyses. This provides a quick glimpse into what
passes are taking the most time to execute, as well as how much of an effect
your pass has on the total execution time of the pipeline. Users can enable this
instrumentation directly on the PassManager via `enableTiming`. This
instrumentation is also made available in mlir-opt via the `-pass-timing` flag.
The PassTiming instrumentation provides several different display modes for the
timing results, each of which is described below:

##### List Display Mode

In this mode, the results are displayed in a list sorted by total time with each
pass/analysis instance aggregated into one unique result. This view is useful
for getting an overview of what analyses/passes are taking the most time in a
pipeline. This display mode is available in mlir-opt via
`-pass-timing-display=list`.

```shell
$ mlir-opt foo.mlir -disable-pass-threading -cse -canonicalize -lower-to-llvm -pass-timing -pass-timing-display=list

===-------------------------------------------------------------------------===
                      ... Pass execution timing report ...
===-------------------------------------------------------------------------===
  Total Execution Time: 0.0203 seconds

   ---Wall Time---  --- Name ---
   0.0047 ( 55.9%)  Canonicalizer
   0.0019 ( 22.2%)  VerifierPass
   0.0016 ( 18.5%)  LLVMLoweringPass
   0.0003 (  3.4%)  CSE
   0.0002 (  1.9%)  (A) DominanceInfo
   0.0084 (100.0%)  Total
```

##### Pipeline Display Mode

In this mode, the results are displayed in a nested pipeline view that mirrors
the internal pass pipeline that is being executed in the pass manager. This view
is useful for understanding specifically which parts of the pipeline are taking
the most time, and can also be used to identify when analyses are being
invalidated and recomputed. This is the default display mode.

```shell
$ mlir-opt foo.mlir -disable-pass-threading -cse -canonicalize -lower-to-llvm -pass-timing

===-------------------------------------------------------------------------===
                      ... Pass execution timing report ...
===-------------------------------------------------------------------------===
  Total Execution Time: 0.0249 seconds

   ---Wall Time---  --- Name ---
   0.0058 ( 70.8%)  'func' Pipeline
   0.0004 (  4.3%)    CSE
   0.0002 (  2.6%)      (A) DominanceInfo
   0.0004 (  4.8%)    VerifierPass
   0.0046 ( 55.4%)    Canonicalizer
   0.0005 (  6.2%)    VerifierPass
   0.0005 (  5.8%)  VerifierPass
   0.0014 ( 17.2%)  LLVMLoweringPass
   0.0005 (  6.2%)  VerifierPass
   0.0082 (100.0%)  Total
```

##### Multi-threaded Pass Timing

When multi-threading is enabled in the pass manager the meaning of the display
slightly changes. First, a new timing column is added, `User Time`, that
displays the total time spent across all threads. Secondly, the `Wall Time`
column displays the longest individual time spent amongst all of the threads.
This means that the `Wall Time` column will continue to give an indicator on the
perceived time, or clock time, whereas the `User Time` will display the total
cpu time.

```shell
$ mlir-opt foo.mlir -cse -canonicalize -lower-to-llvm -pass-timing

===-------------------------------------------------------------------------===
                      ... Pass execution timing report ...
===-------------------------------------------------------------------------===
  Total Execution Time: 0.0078 seconds

   ---User Time---   ---Wall Time---  --- Name ---
   0.0177 ( 88.5%)     0.0057 ( 71.3%)  'func' Pipeline
   0.0044 ( 22.0%)     0.0015 ( 18.9%)    CSE
   0.0029 ( 14.5%)     0.0012 ( 15.2%)      (A) DominanceInfo
   0.0038 ( 18.9%)     0.0015 ( 18.7%)    VerifierPass
   0.0089 ( 44.6%)     0.0025 ( 31.1%)    Canonicalizer
   0.0006 (  3.0%)     0.0002 (  2.6%)    VerifierPass
   0.0004 (  2.2%)     0.0004 (  5.4%)  VerifierPass
   0.0013 (  6.5%)     0.0013 ( 16.3%)  LLVMLoweringPass
   0.0006 (  2.8%)     0.0006 (  7.0%)  VerifierPass
   0.0200 (100.0%)     0.0081 (100.0%)  Total
```

#### IR Printing

Note: The IR Printing instrumentation should only be used when multi-threading
is disabled(`-disable-pass-threading`)

When debugging it is often useful to dump the IR at various stages of a pass
pipeline. This is where the IR printing instrumentation comes into play. This
instrumentation allows for conditionally printing the IR before and after pass
execution by optionally filtering on the pass being executed. This
instrumentation can be added directly to the PassManager via the
`enableIRPrinting` method. `mlir-opt` provides a few useful flags for utilizing
this instrumentation:

*   `print-ir-before=(comma-separated-pass-list)`
    *   Print the IR before each of the passes provided within the pass list.
*   `print-ir-before-all`
    *   Print the IR before every pass in the pipeline.

```shell
$ mlir-opt foo.mlir -disable-pass-threading -cse -print-ir-before=cse

*** IR Dump Before CSE ***
func @simple_constant() -> (i32, i32) {
  %c1_i32 = constant 1 : i32
  %c1_i32_0 = constant 1 : i32
  return %c1_i32, %c1_i32_0 : i32, i32
}
```

*   `print-ir-after=(comma-separated-pass-list)`
    *   Print the IR after each of the passes provided within the pass list.
*   `print-ir-after-all`
    *   Print the IR after every pass in the pipeline.

```shell
$ mlir-opt foo.mlir -disable-pass-threading -cse -print-ir-after=cse

*** IR Dump After CSE ***
func @simple_constant() -> (i32, i32) {
  %c1_i32 = constant 1 : i32
  return %c1_i32, %c1_i32 : i32, i32
}
```

*   `print-ir-module-scope`
    *   Always print the top-level module operation, regardless of pass type or
        operation nesting level.

```shell
$ mlir-opt foo.mlir -disable-pass-threading -cse -print-ir-after=cse -print-ir-module-scope

*** IR Dump After CSE ***  ('func' operation: @bar)
func @bar(%arg0: f32, %arg1: f32) -> f32 {
  ...
}

func @simple_constant() -> (i32, i32) {
  %c1_i32 = constant 1 : i32
  %c1_i32_0 = constant 1 : i32
  return %c1_i32, %c1_i32_0 : i32, i32
}

*** IR Dump After CSE ***  ('func' operation: @simple_constant)
func @bar(%arg0: f32, %arg1: f32) -> f32 {
  ...
}

func @simple_constant() -> (i32, i32) {
  %c1_i32 = constant 1 : i32
  return %c1_i32, %c1_i32 : i32, i32
}
```
