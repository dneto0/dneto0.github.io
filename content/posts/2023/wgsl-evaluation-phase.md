---
title: "WGSL: Three ways to declare pure values"
date: 2023-04-01T14:34:39-04:00
draft: true
toc: true
---

-----
## Introduction

The WebGPU Shading Language (or [WGSL](w3.org/TR/WGSL))
has four ways to declare named values: `var`, `const`, `override`, and `let`.

A `var` declares a variable: a name for *storage* holding a value.
For this post we'll forget about memory for now
--- see what I did there? --- and talk about the others.

So what's up with the other three?

## Giving names to pure values

Each of `const`, `override`, or `let` gives a name to a *value* like a number, vector, array,
or structure.

**Once created, the value does not change**.
This is great because when reading code, there are fewer moving parts to keep track of.

>The objects created by `const`, `override`, and `let` are simpler
than `var`iables because you can reason about them while knowing
much less about how computers work.
To fully understand *variables*, you need to understand how memory is structured, with allocations
and addresses, and how different parts of the system can --- sometimes concurrently ---
read from and write to that memory.
That gets pretty deep pretty fast.

## So why are there three of them?

There are three kinds of (pure) value declarations because there are
**three distinct phases in the life of a WGSL program when an expression can be evaluated**.

They are:

1. Shader creation time: during the [createShaderModule](https://www.w3.org/TR/webgpu/#dom-gpudevice-createshadermodule) API call.
1. Pipeline creation time: during
     [createComputePipeline](https://www.w3.org/TR/webgpu/#dom-gpudevice-createrenderpipeline),
     [createRenderPipeline](https://www.w3.org/TR/webgpu/#dom-gpudevice-createcomputepipeline),
     and their variants.
1. Shader execution.

Take a look at the diagram.
It shows the key interactions between a WebGPU Application and the WebGPU implementation (e.g. your browser).

```mermaid
sequenceDiagram
  participant A as Application
  participant C as Browser
  note over A,C: Get a GPUAdapter from `gpu`<br/>Get a GPUDevice from the adapter
  A ->>+C: device.createShaderModule({ code: '@compute ...' })
  activate C
  Note right of C: Shader-creation time:<br/>Evaluate `const`-expressions.<br/>Set final values for `const`-declarations.
  C -->> A: a GPUShaderModule
  deactivate C
  Note left of A: m &larr; the GPUShaderModule
  A ->>C: device.createComputePipeline({module: m, entryPoint: 'main', constants:{}})
  activate C
  Note right of C: Pipeline-creation time:<br/>Evaluate `override`-expressions.<br/>Set final values for `override`-declarations.
  C -->> A: a GPUComputePipeline
  deactivate C
  Note left of A: cp &larr; the GPUComputePipeline
  Note over A,C: Create and populate buffers and textures,<br/>Create bind group layouts<br/>Record commands into command buffer cb
  %%Note over A,C: Submit commands
  A ->>C: device.queue.submit([cb])
  C-->>C: Wait to be scheduled
  activate C
  Note right of C: Shader execution.<br/>Evaluate runtime expressions<br/>Set `let` values when execution reaches them
  A ->> C: device.queue.onSubmittedWorkDone()
  C -->> A: a pending Promise&lt;undefined&gt;
  C -->> A: fulfill Promise
  deactivate C
```

I've abstracted away the first part, where you get the GPUAdapter, then the GPUDevice from the adapter, and the middle parts where you create resources, bind group layouts, and record the commands to
be executed on the GPU.

The parts shown in more detail are:

1. Shader creation time.
    * This happens when the application calls `device.createShaderModule`, providing
       a `GPUShaderModuleDescriptor` which itself contains the WGSL source code.
    * **This is when all `const`-expressions are evaluated.**
         * The values of `const`-expressions depend *only on the WGSL source code*.
         * They can use literals (like 3.14), arithmetic and logical operators,
             calls to built-in functions with the `@const` attribute, and
             `const`-declared values.
    * **This is also when the values for `const`-declarations
               are set once and for all.**
2. Pipeline creation time.  (Let's take the compute pipeline case.)
    * This happens when the application calls `device.createComputePipeline` (or its async variant),
      providing a GPUComputePipelineDescriptor, which in turn has a GPUProgrammableStage.
    * The GPUProgrammableStage holds the shader module created in step 1.
    * **This is when all `override`-expressions are evaluated.**
         * They can use `const`-expressions, arithmetic and logical operators,
             calls to built-in functions with the `@const` attribute, and
             `override`-declared values.
    * **This is also when the values for `override`-declarations
               are set once and for all.**
    * Ok, hold up.  Why didn't we evaluate override expressions earlier?
      Well, this is where it gets interesting.
      In addition to the shader module, the GPUProgrammableStage also holds a
      [constants](https://www.w3.org/TR/webgpu/#dom-gpuprogrammablestage-constants) dictionary
      mapping names and ids to scalar values. Which matters because...
    * When evaluating override-expressions, **the value of an `override` declaration can be
      set by an entry the `constants` dictionary** *instead of taking the value of its initializer*:
         * If `constants` has an entry (*key*,*X*), and *key* is
           the identifier for an override, then the value for that override will be *X*.
         * Alternately, if an override declaration has an `@id(E)` attribute, and *key* is the string
           for the (decimal) value of *E*, then the value for that override will be *X*.
         * If an override declaration has no such matching entry,
           then it takes the value computed for its initializer expression.
    * Corner cases include:
         * An override controlled in this way must be of scalar type.
         * The *X* value given on the API side as a double-precision float.
            The value must be convertible to the override's type.
         * If an override has no initializer, and isn't matched this way, then pipeline creation fails.
3. Shader execution.
    * This happens when the shader is a stage in a pipeline, a command to execute the pipeline
       is written to a command buffer, that command buffer is submitted to a queue,
       and *finally* the enqueued command eventually is scheduled to run on the GPU.
    *  The shader executes starting at the designated entry point function, and continues executing
       until that entry point finishes.
    * **Runtime-expressions are evaluated when execution reaches them**.
    * In particular, each time a `let`-declaration is reached, it takes the value of its initializer,
       which is a runtime-expression.
    * Runtime-expressions can **depend on things that are only known at shader execution time**,
         some of which **may change during shader execution**, including
         the values stored in variables, pipeline inputs, and the result of executing functions.

> Technically, a runtime-expression is a const-expression if all its identifiers are either
`const`-declared names, or built-in functions with the `@const` attribute.
Similarly, it's an override-expression if all its identifiers are
`const`-declared names, built-in functions with the `@const` attribute, or `override`-declared names.

In summary the key differences are:

* `const`-declaration initializers are const-expressions:
    their values are finalized at shader-creation time, only using information in the WGSL source code.
* `override`-declarations are initialized with override-expressions:
    their values are finalized at pipeline-creation time, and can be overridden
    by a `constants` dictionary provided at that time.
* `let`-declarations are initialized with runtime-expresisons:
    their values are computed each time execution reaches them, and can use data that dynamically
    changes during runtime.

## A few more things

### Numeric const-expressions can be high precision

Const-expressions are designed to be evaluated on the CPU, once, when the shader is first created.
These days, CPUs generally support high precision numeric types.
WGSL lets you use them when computing const-expressions.
WGSL calls these types AbstractInt and AbstractFloat, and they each have *at least*
64bits of precision each.
These are also the only scalar types that automatically convert to another type.

For example, a number like `123456` is an AbstractInt, and `1.3e10` is an AbstractFloat.

WGSL takes advantage of this by allowing numeric const-expressions
having high precision numeric types
"abstract" integer and float types that are high precision,
typically 64bits.


* `const`

<!--
%%   rect aliceblue
%%      Note over A,C: Record commands to run the compute shader
%%      A ->>C: device.createCommandEncoder(...)
%%      C ->A: a GPUCommandEncoder
%%      Note left of A: ce &larr; the GPUCommandEncoder
%%      A ->C: ce.beginComputePass(...)
%%      C ->A: a GPUComputePassEncoder
%%      Note left of A: cpe &larr; the GPUComputePassEncoder
%%      A ->C: cpe.setPipeline(cp) <br/> cpe.dispatchWorkGroups(...)<br/>cpe.end()
%%      A ->C: ce.finish()
%%      C ->A: a GPUCommandBuffer
%%    end
%%    Note left of A: cb &larr; the GPUCommandBuffer
-->

## Further reading
For more, see:

*  [&#x00a7; 2. Shader Lifecycle](https://www.w3.org/TR/WGSL/#shader-lifecycle)
*  [&#x00a7; 6. Variables and Values](https://www.w3.org/TR/WGSL/#var-and-value)
