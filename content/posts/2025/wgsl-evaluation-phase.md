---
title: "WGSL: Three ways to declare pure values"
date: 2025-01-02T14:00:00-04:00
draft: false
toc: true
---

-----
*2025-01-02*

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

> 💡  The objects created by `const`, `override`, and `let` are simpler
than `var`iables because you can reason about them while knowing
much less about how computers work.
To fully understand *variables*, you need to understand how memory is structured, with allocations
and addresses, and how different parts of the system can --- sometimes concurrently ---
read from and write to that memory.
That gets pretty deep pretty fast.

## So why are there three of them?

There are three kinds of (pure) value declarations because 
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
  A ->>C: device.createComputePipeline({module: m, constants:{}})
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
  Note right of C: Shader execution.<br/>Evaluate runtime expressions.<br/>Set `let` values when execution reaches them.
  A ->> C: device.queue.onSubmittedWorkDone()
  C -->> A: a pending Promise&lt;undefined&gt;
  C -->> A: fulfill Promise
  deactivate C
```

I've abstracted away the first part, where you get the GPUAdapter, then the GPUDevice from the adapter, and the middle parts where you create resources, bind group layouts, and record the commands to
be executed on the GPU.

The parts shown in more detail are:

1. Shader creation time.
    * A shader is created when the application calls `device.createShaderModule`, providing
       a `GPUShaderModuleDescriptor` which itself contains the WGSL source code.
    * **This is when all `const`-expressions are evaluated.**
         * The values of `const`-expressions depend *only on the WGSL source code*.
         * They can use literals (like 3.14), arithmetic and logical operators,
             calls to built-in functions with the `@const` attribute, and
             `const`-declared values.
    * **This is also when the values for `const`-declarations
               are set once and for all.**
2. Pipeline creation time.  Let's take the compute pipeline case; the render pipeline case is similar.
    * A compute pipeline is created when the application calls `device.createComputePipeline` (or its async variant),
      providing a GPUComputePipelineDescriptor, which in turn has a GPUProgrammableStage.
    * The GPUProgrammableStage holds the shader module created in step 1.
    * **This is when all `override`-expressions are evaluated.**
         * They can use `const`-expressions (including `const`-declared values),
             arithmetic and logical operators,
             calls to built-in functions with the `@const` attribute, and
             `override`-declared values.
    * **This is also when the values for `override`-declarations
               are set once and for all.**
    * 💡 Why didn't we evaluate override expressions earlier?
      This is where it gets interesting.
      In addition to the shader module, the GPUProgrammableStage also holds a
      [constants](https://www.w3.org/TR/webgpu/#dom-gpuprogrammablestage-constants) dictionary
      mapping names and ids to scalar values. Which matters because...
    * When evaluating override-expressions, **the value of an `override` declaration can be
      set by an entry the `constants` dictionary** *instead of taking the value of its initializer*:
         * If `constants` has an entry (*key*,*X*), and *key* is
           the identifier for an override, then the value for that override will be *X*.
         * Alternately, if an override declaration has an [`@id(E)`](https://www.w3.org/TR/WGSL/#id-attr) attribute, and *key* is the string
           for the (decimal) value of *E*, then the value for that override will be *X*.
         * If an override declaration has no such matching entry,
           then it takes the value computed for its initializer expression.
    * Corner cases include:
         * An override controlled in this way must be of scalar type.
         * The *X* value given on the API side is interpreted as a double-precision float.
            The value must be convertible to the override's type.
         * If an override has no initializer, and isn't matched this way, then pipeline creation fails.
3. Shader execution.
    * This happens when the shader is a stage in a pipeline, a command to execute the pipeline
       is written to a command buffer, that command buffer is submitted to a queue,
       and *finally* the enqueued command eventually is scheduled to run on the GPU.
    * The shader executes starting at the designated entry point function, and continues executing
      until that entry point finishes.
    * Each time `let`-declaration is reached, it takes the value of its initializer expression.
    * Any expression which hasn't already been evaluated in steps 1 or 2 above is called a **runtime expression**,
      and is **evaluated when execution reaches it**.
    * Runtime-expressions can **depend on things that are only known at shader execution time**,
         some of which **may change during shader execution**, including
         the values stored in variables, pipeline inputs, and the result of executing functions.


> 💡 The initializer to a let declaration can also be a const-expression or an override-expression,
depending on which functions and declarations it mentions.

Here's an example compute shader that copies one array to another.
It demonstrates `const`, `override` and runtime expressions.
Each invocation copies up
to `block_size` adjacent elements from the source to the destination.
```rust
const block_size = 8;
override wg_size = 16;

@group(0) @binding(0) var<storage> source: array<u32>;
@group(0) @binding(1) var<storage,read_write> dest: array<u32>;

@compute @workgroup_size(wg_size)
fn main(@builtin(global_invocation_id) gid: vec3u) {
  let num_elem = min(arrayLength(&source), arrayLength(&dest));
  for (var i = 0u; i < block_size; i++) {
    let my_idx = gid.x * block_size + i;
    if (my_idx < num_elem) {
      dest[my_idx] = source[my_idx];
    }
  }
}
```

(I haven't tried the code. I've just compiled it.)

In summary the key differences between the ways of declaring pure values are:

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
CPUs generally support high precision numeric types, and
WGSL lets you use those types when computing const-expressions.

WGSL calls these types [AbstractInt](https://www.w3.org/TR/WGSL/#abstractint) and [AbstractFloat](https://www.w3.org/TR/WGSL/#abstractfloat),
and they each have *at least* 64 bits of precision each.
For example, a number like `123456` is an AbstractInt, and `1.3e10` is an AbstractFloat.

In this code snippet, all the expressions are `const`-expressions, and are evaluated with 64 bit precision.
```c
     const distance_to_sun_m = 149597870700;
     const year_in_s = 365.25 * 24 * 3600;
     const pi = 3.141592653589793238462643;
     const velocity_of_earth_m_per_s = 2 * pi * distance_to_sun_in_meters / year_in_s;
```

### Edge cases before runtime are errors

An error is generated when a `const`-expression or `override` exception overflows, or produces an infinity or NaN.

In this code snippet, both lines are errors.
```c
     const oops = 1.0 / 0.0;
     const oops_nan = 0.0 / 0.0;
```

Why?  Two reasons:

* **GPUs often take shortcuts with floating point math.** 
  They generally don't implement full IEEE 754 floating point rules.
  They won't behave nicely in overflow and other edge cases, and they definitely won't signal exceptions.
  So relying on specific behaviour in those cases will be non-portable, and portability is important for a web standard.
  So WGSL takes the attitude that producing those edge cases is a problem, a defect.
  Expressions evaluated before runtime (const and override) are computed on the CPU, and implementations can (mostly) reliably
  detect the edge cases at that time.  So the implementation is required to generate an error.
* **Room for expansion**. Someday we might want to use more than 64 bit precision for AbstractInt and AbstractFloat.
Today an overflowing abstract expression produces an error.
In the future if we used 128 bits or more, then that expression might not overflow.
By *not* specifying a correct answer today, we can expand the precision of the type without breaking backward compatibility of working applications.
By comparison, the Go language requires compilers to support [constant evaluation well beyond 64 bit precision](https://go.dev/ref/spec#Constants).


## Further reading

*  WGSL [&#x00a7; 2.1 Shader Lifecycle](https://www.w3.org/TR/WGSL/#shader-lifecycle)
*  WGSL [&#x00a7; 7. Variables and Value Declarations](https://www.w3.org/TR/WGSL/#var-and-value)
*  [Constants](https://go.dev/blog/constants) in the Go Blog. This is a really nice explanation of how Go treats constant values.
   I like the simplicity of treating constant values as being typeless as possible.
   Hopefully WGSL's [automatic conversions](https://www.w3.org/TR/WGSL/#feasible-automatic-conversion) for abstract types gives much of that usability.

