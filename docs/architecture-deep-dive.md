# WasmEdge: Deep Architectural Analysis

> A comprehensive guide for core contributors and future maintainers.

---

## Table of Contents

1. [High-Level Purpose](#1-high-level-purpose)
2. [Architecture Overview](#2-architecture-overview)
3. [Core Abstractions](#3-core-abstractions)
4. [Module Execution Flow](#4-module-execution-flow)
5. [Data Flow](#5-data-flow)
6. [Design Patterns](#6-design-patterns)
7. [Dependency Structure](#7-dependency-structure)
8. [Testing Framework](#8-testing-framework)
9. [Active Development Zones](#9-active-development-zones)
10. [Complexity and Technical Debt](#10-complexity-and-technical-debt)
11. [Performance Analysis](#11-performance-analysis)
12. [Extension Mechanisms](#12-extension-mechanisms)
13. [Contribution Entry Points](#13-contribution-entry-points)
14. [Roadmap Signals](#14-roadmap-signals)
15. [Strategic Summary](#15-strategic-summary)

---

## 1. High-Level Purpose

### What problem does this repository solve?

WasmEdge is a **lightweight, high-performance, and extensible WebAssembly (Wasm)
runtime** written in C++17. It solves the problem of safely executing
untrusted or third-party code inside a host application or OS environment.
WebAssembly bytecode runs inside a deterministic sandbox, providing strong
memory-safety and capability-based security guarantees regardless of the
underlying hardware architecture.

### Role in the larger ecosystem

WasmEdge occupies several roles simultaneously:

- **CNCF sandbox project** — positioned alongside containers (Docker/Kubernetes)
  as a complementary, lower-overhead isolation primitive.
- **Edge / serverless runtime** — used by edge CDN providers and serverless
  platforms that need sub-millisecond cold starts.
- **AI inference engine** — its WASI-NN plugin family (GGML/llama.cpp,
  TensorFlow Lite, etc.) turns WasmEdge into a portable AI inference runtime
  consumable from any Wasm guest.
- **Embedded extension host** — applications embed WasmEdge to let users supply
  plugins written in any language that compiles to Wasm (Rust, C/C++, Go,
  Python, etc.).
- **Component Model pioneer** — one of the first runtimes implementing the Wasm
  Component Model specification, enabling language-agnostic module composition.

### Target users

| User type | Use case |
|---|---|
| Application developers | Embed WasmEdge to safely run user-supplied or third-party code |
| Cloud / edge operators | Run Wasm workloads in Kubernetes (via runwasi) or serverless |
| AI/ML engineers | Run LLM inference via the WASI-NN plugin |
| Wasm ecosystem contributors | Prototype and test new Wasm proposals |
| Runtime implementers | Study a reference C++ implementation of the Wasm spec |

### Design philosophy

- **Spec-first** — every Wasm proposal is tracked and implemented against the
  official specification test suite.
- **Zero-overhead abstractions** — heavy use of C++17 templates (CRTP,
  `if constexpr`, `std::apply`) to eliminate runtime overhead from the plugin
  and host-function binding layer.
- **No exceptions** — errors propagate via `Expect<T>` (a specialization of
  `Expected<T, ErrCode>`), keeping the hot path free of exception-handling
  overhead and making error handling explicit at every call site.
- **Layered extensibility** — the plugin ABI, C API, and Component Model each
  provide distinct extension points at different levels of abstraction.

---

## 2. Architecture Overview

### Major modules

| Directory | Module | Responsibility |
|---|---|---|
| `include/` + `lib/loader/` | **Loader** | Deserializes `.wasm` binary into an in-memory AST |
| `include/` + `lib/validator/` | **Validator** | Type-checks and validates the AST against the Wasm spec |
| `include/` + `lib/executor/` | **Executor** | Instantiates modules and runs instructions |
| `include/` + `lib/vm/` | **VM** | Thread-safe orchestration facade over Loader/Validator/Executor |
| `include/` + `lib/aot/` | **AOT cache** | Manages on-disk cache of AOT-compiled native binaries |
| `include/` + `lib/llvm/` | **LLVM compiler / JIT** | Compiles Wasm AST to native code via LLVM |
| `include/` + `lib/ast/` | **AST** | Data-only node types representing the parsed Wasm module |
| `include/` + `lib/runtime/` | **Runtime instances** | Mutable objects instantiated from AST nodes (memory, table, …) |
| `include/` + `lib/common/` | **Common** | Shared types: `ErrCode`, `Expect<T>`, `Configure`, `Span`, enums |
| `include/` + `lib/host/` | **Host / WASI** | Built-in WASI implementation and virtual filesystem |
| `include/` + `lib/api/` | **C API** | Stable C ABI exported as `libwasmedge.so` / `wasmedge.dll` |
| `include/` + `lib/plugin/` | **Plugin loader** | Loads `*.so` / `*.dll` plugin files and registers them with the VM |
| `include/` + `lib/system/` | **System** | OS abstractions: `mmap`, signal handlers, shared library loading |
| `include/` + `lib/driver/` | **CLI driver** | Entry points for the `wasmedge` and `wasmedgec` binaries |
| `include/` + `lib/po/` | **Program options** | Command-line argument parsing |
| `plugins/` | **Plugin impls** | Optional host-function bundles (WASI-NN, image, crypto, …) |
| `tools/` | **CLI binaries** | `wasmedge` and `wasmedgec` main() wrappers |
| `test/` | **Tests** | Google Test suites mirroring the `lib/` tree |

### How modules interact

```
User / Embedding Application
        │
        ▼
┌────────────────────────────────────────────────────────────────────┐
│  C API  (lib/api / include/api/wasmedge/wasmedge.h)                │
│  (stable ABI — all language bindings target this layer)            │
└───────────────────────────┬────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│  VM  (lib/vm / vm.h)                                               │
│  Composes Loader + Validator + Executor                             │
│  Holds StoreManager, thread-safety (std::shared_mutex)             │
└────┬───────────────┬──────────────────────────┬────────────────────┘
     │               │                          │
     ▼               ▼                          ▼
┌─────────┐   ┌────────────┐          ┌──────────────────┐
│  Loader │   │  Validator │          │    Executor      │
│ (AST)   │──▶│ (FormCheck)│─────────▶│  instantiate()   │
└─────────┘   └────────────┘          │  runFunction()   │
                                      └────────┬─────────┘
                                               │
                    ┌──────────────────────────┤
                    ▼                          ▼
           ┌────────────────┐       ┌──────────────────┐
           │ StoreManager   │       │  StackManager    │
           │ (module map)   │       │  (value/frames)  │
           └────────────────┘       └──────────────────┘
                    │
       ┌────────────┼──────────────────────┐
       ▼            ▼                      ▼
  ModuleInstance  MemoryInstance    FunctionInstance
  TableInstance   GlobalInstance    ComponentInstance
```

```
                    LLVM Compilation Path
                    ─────────────────────
  AST::Module ──▶ llvm::Compiler ──▶ LLVM IR ──▶ Machine Code
                                                      │
                               JIT (in-process) ◀────┤
                               AOT (on-disk .so) ◀───┘
```

### System layers

1. **Binary / Wire layer** — `lib/loader` consumes raw bytes.
2. **AST layer** — `include/ast` holds immutable, parsed nodes.
3. **Semantic / validation layer** — `lib/validator` annotates and checks the AST.
4. **Instantiation layer** — `lib/executor` turns AST nodes into mutable runtime instances.
5. **Execution layer** — `lib/executor` runs the instruction dispatch loop.
6. **Host interface layer** — `lib/host` and `plugins/` provide host functions callable from Wasm.
7. **Compilation layer** — `lib/llvm` and `lib/aot` optionally replace interpretation with native code.
8. **API / embedding layer** — `lib/api` wraps all of the above in a stable C ABI.

---

## 3. Core Abstractions

### Base and abstract classes

#### `Runtime::HostFunctionBase` (`include/runtime/hostfunc.h`)

The root abstraction for every function that is implemented on the host side
(in C++) and callable from Wasm.

```
HostFunctionBase
├── pure virtual run(CallingFrame, Args, Rets) → Expect<void>
├── getFuncType()  → AST::FunctionType const&
├── getCost()      → uint64_t
└── getDefinedType() → AST::SubType const&
```

Subclasses: `HostFunction<T>` (CRTP), `Wasi<T>` (WASI host functions),
every plugin function.

#### `Runtime::HostFunction<T>` (CRTP template, same file)

Provides automatic parameter/return-type extraction via `FuncTraits<&T::body>`
and delegates to `T::body(CallingFrame, …)` at compile time.  No virtual
dispatch is needed for the common hot path; only the base-class `run()` virtual
is used when the function is called through a pointer.

#### `Runtime::Instance::CompositeBase` (`include/runtime/instance/composite.h`)

Base for GC-proposal heap objects.

```
CompositeBase
├── FunctionInstance
├── StructInstance     (GC proposal)
└── ArrayInstance      (GC proposal)
```

#### `Runtime::Instance::ModuleInstance` (`include/runtime/instance/module.h`)

Owns all runtime objects belonging to one instantiated module.  Not abstract
itself but subclassed by `WasiModule` and plugin modules.

#### `common::Executable` (`include/common/executable.h`)

Interface implemented by `JITLibrary` (in-process JIT) and AOT-loaded shared
libraries.  Provides pointers to compiled function bodies, type metadata, and
intrinsics tables.

### Inheritance hierarchy (host-function side)

```
HostFunctionBase
└── HostFunction<T>                 (CRTP, include/runtime/hostfunc.h)
    ├── Wasi<WasiArgsGet>
    ├── Wasi<WasiFdRead>
    ├── Wasi<WasiClockTimeGet>
    │   …  (50+ WASI syscall wrappers in lib/host/wasi/)
    ├── WasiLoggingImpl<…>          (wasi_logging plugin)
    ├── WasiCryptoOp<…>             (wasi_crypto plugin)
    ├── NNFunc<…>                   (wasi_nn plugin)
    └── <any user-defined plugin function>
```

### How polymorphism is used

- **Static (compile-time) polymorphism via CRTP** — `HostFunction<T>` extracts
  the function signature of `T::body` at compile time using `FuncTraits`.  No
  `dynamic_cast` or `typeid` is used in the hot path.
- **Dynamic (runtime) polymorphism via virtual** — `HostFunctionBase::run()` is
  a pure virtual method.  The executor calls it through a `HostFunctionBase*`
  pointer when invoking a host function from Wasm, paying a single vtable
  lookup per host call (which is negligible compared to the ABI boundary cost).
- **`std::variant` dispatch** — `FunctionInstance::Data` is a
  `std::variant<WasmFunction, Symbol<CompiledFunction>, HostFunctionBase*>`.
  The executor uses `std::holds_alternative` / `std::visit` to choose the
  interpretation, JIT, or host-call path without inheritance overhead.

### Extensibility

- **Plugins** — a plugin shared library exports `GetDescriptor()` returning a
  `PluginDescriptor`.  The descriptor carries a factory function pointer that
  creates a `ModuleInstance`.  No recompilation of the runtime is required.
- **Component Model** — `ComponentInstance` and `Component::FunctionInstance`
  mirror `ModuleInstance` / `FunctionInstance` for the component layer.
- **Wasm proposals** — new proposals add new `Configure` flags and new AST node
  types; the loader, validator, and executor each grow new handling code gated
  on `Configure::hasProposal(...)`.

---

## 4. Module Execution Flow

This section traces a complete lifecycle from a `.wasm` file on disk to the
return value of a function call.

### Step 1 — Initialization

```cpp
Configure Conf;
Conf.addProposal(Proposal::BulkMemoryOperations);
// ...
VM::VM MyVM(Conf);
// Internal construction:
//   new Loader(Conf)
//   new Validator(Conf)
//   new Executor(Conf, &Stats)
//   new StoreManager
//   load built-in WASI module and any Plugin modules
```

### Step 2 — Loading

```
VM::runWasmFile("app.wasm", "_start")
  └─▶ VM::unsafeLoadWasm(Path)
        └─▶ Loader::parseModule(Path)
              ├─ FileManager opens the file, mmaps or reads it
              ├─ Parse magic number (0x00 0x61 0x73 0x6D) + version
              ├─ Parse each section in order:
              │    Type → Import → Function → Table → Memory →
              │    Global → Export → Start → Element → Code → Data
              ├─ Each section creates AST nodes in ast/
              └─▶ returns unique_ptr<AST::Module>
```

Key files: `lib/loader/loader.cpp`, `lib/loader/filemgr.cpp`,
`include/ast/module.h`, `include/ast/section.h`, `include/ast/type.h`.

### Step 3 — Validation

```
VM::unsafeValidate(AST::Module)
  └─▶ Validator::validate(AST::Module)
        ├─ Check section ordering
        ├─ Validate types section (recursive type groups, subtypes)
        ├─ FormChecker per function body:
        │    ├─ Tracks a type stack
        │    ├─ Checks each instruction for stack effects
        │    └─ Validates control flow (block nesting, labels)
        ├─ Validate imports, exports, start function
        └─▶ Returns Expect<void>; on error, carries ErrCode + ErrInfo
```

Key files: `lib/validator/validator.cpp`, `lib/validator/formchecker.cpp`.

### Step 4 — Instantiation

```
VM::unsafeInstantiate(AST::Module)
  └─▶ Executor::instantiateModule(Store, AST::Module)
        ├─ Resolve imports from StoreManager
        │    (host functions, WASI, registered modules, plugins)
        ├─ Allocate MemoryInstance(s)
        ├─ Allocate TableInstance(s)
        ├─ Evaluate ConstExpr for GlobalInstance initial values
        ├─ Wrap each code section as FunctionInstance::WasmFunction
        ├─ Execute element/data active segments
        ├─ Register the new ModuleInstance in StoreManager
        └─▶ Returns Expect<ModuleInstance*>
```

Key files: `lib/executor/instantiate/`, `include/runtime/instance/module.h`.

### Step 5 — Execution

```
VM::execute("_start", Params)
  └─▶ Executor::invoke(FunctionInstance*, Params, Rets)
        └─▶ Executor::runFunction(CallingFrame, FunctionInstance, Params, Rets)
              ├─ Push CallingFrame onto StackManager
              ├─ [Interpreter path]
              │    Loop: fetch instruction → dispatch to handler → update stack
              │    Handlers for: numeric, memory, control, reference, SIMD, GC, …
              ├─ [JIT/AOT path]
              │    Resolve compiled symbol from FunctionInstance::Data
              │    Call machine-code function via function pointer
              │    (intrinsics table provides memory/table base pointers)
              ├─ Pop CallingFrame
              └─▶ Returns Expect<vector<pair<ValVariant, ValType>>>
```

Key files: `lib/executor/engine/`, `lib/executor/engine/run.cpp`,
`include/runtime/stackmgr.h`, `include/runtime/callingframe.h`.

### Modules involved (in order)

1. `common/configure.h` — runtime feature flags
2. `loader/loader.h` — binary → AST
3. `validator/validator.h` — AST → validated AST
4. `executor/executor.h` — instantiation + execution
5. `runtime/storemgr.h` — module registry
6. `runtime/stackmgr.h` — execution stack
7. `runtime/instance/module.h` — live module state
8. `runtime/instance/function.h` — individual functions
9. `runtime/instance/memory.h` — linear memory
10. `host/wasi/` — WASI system calls
11. `plugin/` — optional extension modules
12. `llvm/compiler.h` + `llvm/jit.h` — native compilation (optional)
13. `aot/cache.h` — cache lookup (optional)
14. `vm/vm.h` — top-level facade

---

## 5. Data Flow

### Full lifecycle of data

```
┌──────────────────────────────────────────────────────────────────┐
│ SOURCE                                                           │
│  .wasm file on disk  OR  byte buffer in memory                   │
└───────────────────────────┬──────────────────────────────────────┘
                            │ raw bytes
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ LOADING (lib/loader)                                             │
│  FileManager::readFile()  →  Span<const Byte>                    │
│  parseModule()            →  AST::Module  (immutable tree)       │
│   Each section type has a dedicated parse*() method              │
│   Type definitions: AST::FunctionType, AST::SubType, …           │
│   Instructions: AST::Instruction (tagged union via OpCode enum)  │
└───────────────────────────┬──────────────────────────────────────┘
                            │ AST::Module
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ VALIDATION (lib/validator)                                       │
│  Validator traverses AST in spec-defined order                   │
│  FormChecker maintains a type-stack per function body            │
│  Annotates branch target arities on control-flow instructions    │
│  On failure: Unexpect(ErrCode) with ErrInfo for human messages   │
└───────────────────────────┬──────────────────────────────────────┘
                            │ validated AST::Module (same object)
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ INSTANTIATION (lib/executor/instantiate)                         │
│  Import resolution: look up StoreManager by module+name string   │
│  MemoryInstance: allocates page-aligned byte buffer              │
│  GlobalInstance: evaluates constant-expression initializers      │
│  FunctionInstance: stores pointer to AST code + type info        │
│  Active element/data segments: bulk-copy into memory/tables      │
│  Start function: invoked immediately at instantiation end        │
└───────────────────────────┬──────────────────────────────────────┘
                            │ ModuleInstance* in StoreManager
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ EXECUTION (lib/executor/engine)                                  │
│  CallingFrame wraps (ModuleInstance*, StackManager*)             │
│  Values stored as ValVariant (std::variant of i32/i64/f32/f64/…)│
│  Instruction handlers pop operands, compute, push results        │
│  Memory access: validated offset arithmetic → raw byte read/write│
│  Host call: HostFunctionBase::run() → back into C++ land         │
│  Control flow: structured blocks map to label stack entries      │
└───────────────────────────┬──────────────────────────────────────┘
                            │ vector<pair<ValVariant, ValType>>
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│ OUTPUT                                                           │
│  Returned to embedding application or C API caller               │
│  Side-effects: WASI writes to host file descriptors              │
│                Stdout captured or forwarded                      │
└──────────────────────────────────────────────────────────────────┘
```

### WASI data path (system calls from Wasm)

1. Wasm instruction calls an imported host function by index.
2. Executor resolves the import to a `FunctionInstance` containing a
   `HostFunctionBase*` (pointing to a `Wasi<T>` object).
3. `HostFunctionBase::run()` is called; arguments arrive as `ValVariant`
   (integers representing Wasm linear memory pointers and lengths).
4. The WASI function reads/writes Wasm linear memory via `CallingFrame::getMemoryByIndex()`.
5. The operation is performed on the host OS through `Environ`
   (which wraps the VFS and native file descriptors).
6. Return values (error codes, byte counts) are written back into `Rets`.

---

## 6. Design Patterns

### CRTP — Curiously Recurring Template Pattern

**Where**: `HostFunction<T>` and `Wasi<T>` (`include/runtime/hostfunc.h`,
`lib/host/wasi/wasibase.h`).

**Why**: Lets the compiler extract argument and return types from `T::body`
at compile time via `FuncTraits`, generating per-function type-checking and
marshaling code with zero runtime overhead.

```cpp
// Each host function is a struct with a body() method:
struct WasiArgsGet : public Wasi<WasiArgsGet> {
  Expect<uint32_t> body(const Runtime::CallingFrame &Frame,
                        uint32_t ArgvPtr, uint32_t ArgvBufPtr);
};
// HostFunction<WasiArgsGet> reads &WasiArgsGet::body's type at compile time.
```

### Factory — Plugin descriptor pattern

**Where**: `plugins/*/env.cpp` and `include/plugin/plugin.h`.

**Why**: Allows runtime loading of plugins without coupling the runtime to any
specific plugin implementation.

```cpp
// Plugin shared library exports:
extern "C" Plugin::PluginDescriptor *getDescriptor() noexcept {
  return &Descriptor;
}
// Descriptor contains Create = []() { return new MyModule(); }
```

### Expected / Result type — Railway-oriented error handling

**Where**: Throughout all modules; defined in `include/common/expected.h` and
`include/common/errcode.h`.

**Why**: Avoids exceptions on the hot execution path; makes all failure paths
visible in function signatures.

```cpp
Expect<void> Validator::validate(const AST::Module &Mod) {
  if (!checkMagic(Mod)) return Unexpect(ErrCode::Value::MalformedMagic);
  EXPECTED_TRY(validateTypes(Mod.getTypeSection()));
  // ...
  return {};
}
```

### Variant-based tagged union (discriminated union)

**Where**: `Runtime::Instance::FunctionInstance::Data`,
`AST::Instruction` (via `OpCode` enums), `ValVariant`.

**Why**: Avoids virtual dispatch for the most performance-sensitive objects;
`std::visit` / `std::holds_alternative` give branch-predictor-friendly
dispatch.

### Strategy — Configure flags

**Where**: `Configure` class (`include/common/configure.h`).

**Why**: Runtime behavior (which Wasm proposals are active, optimization level,
statistics collection) is parameterized through a single `Configure` object
passed at construction time, making the same runtime code serve many
configurations without subclassing.

### Facade — VM class

**Where**: `VM::VM` (`include/vm/vm.h`).

**Why**: Hides the three-step Loader/Validator/Executor pipeline behind a
single object; manages thread-safety via `std::shared_mutex` over all public
methods.

### Adapter — C API layer

**Where**: `lib/api/` and `include/api/wasmedge/`.

**Why**: Wraps the C++ `VM`, `Executor`, `Loader`, etc. types behind a C ABI
so that non-C++ language bindings (Go, Rust, Python, Java) can call the
runtime without linking to C++ symbols.

### Template meta-programming — FuncTraits

**Where**: `HostFunction<T>` implementation (`include/runtime/hostfunc.h`).

**Why**: Extracts argument count, argument types, and return types from a
member function pointer at compile time, enabling automatic registration of
host functions without manual type declaration.

---

## 7. Dependency Structure

### Module-level dependency graph

```
vm
 ├── loader
 │    ├── ast
 │    └── common
 ├── validator
 │    ├── ast
 │    └── common
 ├── executor
 │    ├── ast
 │    ├── runtime
 │    │    ├── instance  (module, memory, table, global, function, …)
 │    │    ├── stackmgr
 │    │    └── storemgr
 │    └── common
 └── plugin
      └── system  (SharedLibrary)

api
 └── vm  (and all transitive deps)

driver
 └── vm, po, api

host/wasi
 └── runtime, common, system

plugins/*
 └── runtime/hostfunc (HostFunction<T>)

llvm
 ├── ast
 ├── common
 └── [LLVM libraries]

aot
 ├── llvm
 └── common
```

### Central modules (most depended upon)

1. **`common`** — depended on by literally every other module.
2. **`ast`** — depended on by loader, validator, executor, llvm.
3. **`runtime`** — depended on by executor, host, plugins, api.
4. **`executor`** — depended on by vm, api, driver.

Avoid making changes to `common` or `ast` without a thorough review of
downstream impact.

---

## 8. Testing Framework

### Structure

Tests live in `test/` with a directory layout that mirrors `lib/`:

```
test/
├── spec/          Official WebAssembly spec tests (JSON-driven)
├── loader/        Unit tests for the loader
├── validator/     Unit tests for the validator
├── executor/      Unit tests for the executor
├── api/           C API integration tests
├── llvm/          JIT/AOT tests
├── component/     Component Model tests
├── host/          WASI tests
├── plugins/       Plugin integration tests (testplugin)
├── aot/           AOT cache tests
├── common/        Tests for common utilities
├── expected/      Tests for the Expected type
├── errinfo/       Error info tests
├── memlimit/      Memory limit / OOM tests
├── thread/        Concurrency tests
└── mixcall/       Mixed interpreter + JIT tests
```

### Test design philosophy

- **Google Test** framework throughout; each `test/` subdirectory compiles to a
  separate test executable registered with CMake's `ctest`.
- **Spec tests** (`test/spec/`) run against the official
  `WebAssembly/testsuite` `.wast` files (downloaded as a CMake fetch step);
  they are the primary conformance validation tool.
- **Unit tests** for individual pipeline stages test specific instructions,
  edge cases, and error conditions in isolation.
- **Plugin tests** use a dedicated `testplugin` module that registers known
  host functions; plugin correctness is verified end-to-end through the full
  VM pipeline.
- **API tests** exercise the C API; they serve as regression tests for ABI
  stability.

### Building and running tests

```bash
cmake -S . -B build -DWASMEDGE_BUILD_TESTS=ON
cmake --build build -j$(nproc)
cd build && ctest -R <test-name-regex>
```

### How to validate a new estimator / new component

For new Wasm proposals, add a corresponding spec test subdirectory.
For new host functions, add a unit test in the relevant `test/host/` or
`test/plugins/` directory following the `HostFunctionTest` pattern.

---

## 9. Active Development Zones

### Under active development (as of Q1 2026)

| Area | Signal |
|---|---|
| **Component Model** | Multiple ongoing roadmap items; `include/ast/component/`, `lib/executor/engine/component/`, `lib/runtime/instance/component/` are growing |
| **WASI preview 2** | Listed as Q2/2026 roadmap target; involves `wasi_poll` plugin and component-based WASI |
| **WASI-NN / LLM plugins** | Continuous updates to `plugins/wasi_nn/` for llama.cpp, new model architectures |
| **JIT per-function compilation** | LFX 2026/term1 item; affects `lib/llvm/` and `lib/executor/` |
| **CLI sub-commands** | LFX 2026/term1 item; affects `lib/driver/` and `lib/po/` |
| **Module instance dependency tree** | LFX 2026/term1 item; affects `lib/runtime/storemgr.h` |

### Experimental areas

- **GC proposal runtime support** — staled but has an open PR; `include/runtime/instance/struct.h`, `array.h`.
- **Exception-handling AOT/JIT** — staled; requires non-trivial LLVM landing-pad integration.
- **Typed continuations / stack-switch** — staled proposals; very early stage.

### Stable areas

- **Core interpreter** — spec-compliant, fuzzed via OSS-Fuzz; very stable.
- **WASI preview 1** — fully implemented; used in production (runwasi, Docker).
- **C API** — stable ABI; all language bindings depend on it.
- **Loader / Validator** — mature; changes only when tracking new Wasm proposals.

---

## 10. Complexity and Technical Debt

### Tightly coupled modules

- `Executor` is tightly coupled to `StackManager` and `CallingFrame`; the
  execution loop directly manipulates the stack, making it hard to insert
  tracing or debugging layers without intrusive changes.
- `VM` couples `Loader`, `Validator`, and `Executor` into one object.  While
  the facade is intentional, users who want to use only the Loader or Validator
  must replicate the VM logic themselves (or use the C API `WasmEdge_Loader*`
  sub-APIs).

### Large or complex files

- `lib/executor/engine/run.cpp` — the main instruction dispatch loop; hundreds
  of switch-case branches.  Adding a new instruction requires careful ordering.
- `lib/host/wasi/environ.cpp` — large platform-specific file that handles all
  WASI system calls; platform differences are handled with `#ifdef` rather than
  abstraction layers.
- `include/common/types.h` — aggregates a large number of type aliases and
  variant definitions used everywhere.

### Potential design inconsistencies

- **Two function instance types** — `ModuleInstance` functions and
  `Component::FunctionInstance` have parallel but separate class hierarchies;
  this duplication will likely need to be unified as the Component Model matures.
- **Plugin vs. built-in WASI** — WASI is currently a built-in module loaded
  unconditionally; the roadmap targets moving WASI preview 2 into plugins,
  which will require a cleaner separation between "always present" and
  "optionally present" host modules.
- **AOT cache key** — uses BLAKE3 of the binary + runtime version; does not
  incorporate `Configure` flags, meaning a cached binary might be used with
  incompatible proposal settings in edge cases.

### Violations of separation of concerns

- `Executor` both instantiates modules (an "allocation" concern) and runs them
  (an "execution" concern); separating these into distinct objects would improve
  testability.
- `CallingFrame` gives executing instructions direct access to the owning
  `ModuleInstance`, blurring the line between execution context and module state.

---

## 11. Performance Analysis

### Computational bottlenecks

- **Instruction dispatch loop** (`lib/executor/engine/run.cpp`) — computed
  `goto` or a large `switch` over `OpCode`; the branch predictor can struggle
  with highly polymorphic Wasm workloads.  The JIT path eliminates this.
- **Host function call boundary** — every call from Wasm to a host function
  goes through `HostFunctionBase::run()` (one virtual dispatch) plus argument
  marshaling from `ValVariant` to typed C++ values.  For WASI-heavy workloads
  (many `fd_read`/`fd_write` calls), this adds up.
- **Memory bounds checking** — every Wasm memory load/store performs a bounds
  check.  The JIT can emit guard pages to eliminate explicit checks.

### Memory-heavy components

- `MemoryInstance` — allocates full Wasm pages (64 KiB each) up front using
  `mmap`; the maximum is 4 GiB for 32-bit Wasm.  Memory64 would extend this.
- `StackManager::ValueStack` — pre-allocates 2048 `ValVariant` slots; a deeply
  recursive Wasm program can exhaust this.
- WASM-NN plugin — loads multi-gigabyte LLM model weights into native heap
  memory outside of Wasm linear memory.

### Scalability risks

- `StoreManager` uses a single `std::shared_mutex`; high-concurrency workloads
  with many module instantiations will contend on this lock.
- The interpreter is single-threaded per Wasm instance by design (the Wasm
  threading proposal requires separate linear memory views).

### Opportunities for GPU / vectorization

- The WASI-NN plugin already offloads inference to GPU via Metal (macOS) and
  CUDA (via llama.cpp).
- The SIMD proposal (`v128` instructions) is interpreter-implemented today;
  the JIT should auto-vectorize these through LLVM.
- Relaxed-SIMD (merged Q4/2024) enables non-deterministic SIMD that can map
  more directly to hardware intrinsics.

---

## 12. Extension Mechanisms

### Adding a new built-in Wasm proposal (e.g., memory64)

1. Add a new `Proposal` enum value in `include/common/enum_configure.hpp`.
2. Add a `hasProposal()` guard in `lib/loader/`, `lib/validator/`, and
   `lib/executor/` at each affected code path.
3. Add or extend AST nodes in `include/ast/` as needed.
4. Add new instruction opcodes in `include/common/enum_ast.hpp`.
5. Implement validator logic in `lib/validator/formchecker.cpp`.
6. Implement execution handlers in `lib/executor/engine/`.
7. Add spec test `.wast` files under `test/spec/`.
8. Register the proposal in `CMakeLists.txt` if it requires optional compilation.

### Adding a new host function plugin

#### 1. Create the plugin directory

```
plugins/my_plugin/
├── CMakeLists.txt
├── my_env.h / my_env.cpp     # Env struct + plugin descriptor
├── my_func.h / my_func.cpp   # Host function implementations
└── my_module.h / my_module.cpp # ModuleInstance registering functions
```

#### 2. Implement the host functions (extend `HostFunction<T>`)

```cpp
// my_func.h
struct MyFunc : public Runtime::HostFunction<MyFunc> {
  MyFunc(MyEnv &Env) : Env(Env) {}
  Expect<uint32_t> body(const Runtime::CallingFrame &Frame,
                        uint32_t Arg0);
private:
  MyEnv &Env;
};
```

The `body()` method receives the `CallingFrame` (for memory access) and typed
arguments.  Return type must be `Expect<T>` or `Expect<void>`.

#### 3. Create the ModuleInstance

```cpp
// my_module.cpp
MyModule::MyModule(MyEnv &Env) : Runtime::Instance::ModuleInstance("my_plugin") {
  addHostFunc("my_function", std::make_unique<MyFunc>(Env));
}
```

#### 4. Create the plugin descriptor and export `GetDescriptor`

```cpp
// my_env.cpp
static Plugin::PluginModule::ModuleDescriptor ModDesc[] = {{
  .Name = "my_plugin",
  .Description = "My example plugin",
  .Create = [](const Plugin::PluginModule::ModuleDescriptor *) {
    return std::make_unique<MyModule>(Env);
  },
}};
static Plugin::PluginDescriptor Descriptor{
  .Name = "my_plugin",
  .Description = "My plugin description",
  .APIVersion = Plugin::Plugin::CurrentAPIVersion,
  .Version = {0, 1, 0, 0},
  .ModuleCount = 1,
  .ModuleDescriptions = ModDesc,
};
EXPORT_GET_DESCRIPTOR(Descriptor)
```

#### 5. Register in CMake

In root `CMakeLists.txt`:
```cmake
option(WASMEDGE_PLUGIN_MY_PLUGIN "Enable my plugin" OFF)
```

In `plugins/CMakeLists.txt`:
```cmake
if(WASMEDGE_PLUGIN_MY_PLUGIN)
  add_subdirectory(my_plugin)
endif()
```

#### 6. Write tests

Add a test in `test/plugins/` following the `testplugin` pattern:
- Instantiate the plugin module via `VM::registerModule`.
- Call the exported functions via `VM::execute`.
- Assert return values.

### Adding a new Component Model component

1. Add AST nodes in `include/ast/component/`.
2. Add loader support in `lib/loader/` (parse new section types).
3. Add validator support in `lib/validator/`.
4. Add executor support in `lib/executor/engine/component/`.
5. Add `Runtime::Instance::ComponentInstance` extensions as needed.
6. Reference the Component Model spec tests in `test/component/`.

---

## 13. Contribution Entry Points

### Beginner level

- **Fix typos and improve error messages** — error strings live in
  `lib/common/` enum files; better messages help users debug their Wasm.
- **Add or improve documentation** — the `docs/` directory and inline Doxygen
  comments welcome improvements.
- **Improve test coverage** — add new unit tests for edge cases in
  `test/loader/`, `test/validator/`, or `test/executor/`.
- **Fix IWYU (Include What You Use) warnings** — there is an ongoing effort to
  clean up unnecessary includes; see the CI `IWYU_scan.yml` workflow.

### Intermediate level

- **Implement or fix a WASI host function** — `lib/host/wasi/` and
  `include/host/wasi/` are well-structured; each WASI function follows the
  `Wasi<T>` pattern.
- **Add a new plugin** — follow the plugin pattern described in
  [Extension Mechanisms](#12-extension-mechanisms); the `wasmedge_image` plugin
  is a minimal example to copy.
- **Fix conformance gaps** — run the official spec test suite and investigate
  any failures; each failure maps to a specific instruction or proposal.
- **Add statistics instrumentation** — `common/statistics.h` and
  `Executor::Statistics` can be extended to track new runtime metrics.

### Advanced architecture level

- **Component Model completion** — implement missing canonical ABI lifting/
  lowering, alias types, start sections.  See `include/ast/component/` and
  `lib/executor/engine/component/`.
- **Per-function JIT compilation** — extend `lib/llvm/jit.h` to support lazy,
  per-function compilation triggered at call time; requires changes to
  `FunctionInstance` and `Executor`.
- **Exception-handling in AOT/JIT** — map Wasm exception semantics onto LLVM
  landing pads; requires modifying the LLVM IR generation in `lib/llvm/`.
- **StoreManager scalability** — redesign the module registry to use a
  lock-free or sharded map for high-concurrency instantiation workloads.
- **Module instance dependency tree** — implement a DAG of module dependencies
  in `StoreManager` to support ordered teardown and circular-dependency
  detection.

---

## 14. Roadmap Signals

### Long-term evolution implied by the codebase

- **Component Model as the primary composition mechanism** — the growing
  `include/ast/component/` and `lib/executor/engine/component/` trees signal
  that components will eventually replace raw module linking.  WASI preview 2
  is entirely Component Model-based.
- **WASI as plugins** — the plan to move WASI preview 2 into the plugin system
  implies a cleaner separation between the runtime core and system interfaces,
  making the runtime more embeddable without WASI.
- **JIT by default, interpreter as fallback** — the addition of JIT support and
  per-function JIT roadmap items suggest the interpreter will eventually become
  a debug/validation path only.
- **AI inference as a first-class use case** — the WASI-NN plugin family is
  actively maintained and expanding; WasmEdge is positioning itself as a
  portable AI runtime.

### Architectural transitions in progress

- **WASI preview 1 → preview 2** — preview 2 uses Component Model instead of
  core Wasm imports; requires the entire Component Model stack to mature first.
- **Monolithic WASI module → plugin-based WASI** — decouples WASI from the
  runtime binary, reducing size and enabling custom WASI implementations.
- **Interpreter → JIT as default execution tier** — requires per-function JIT
  and warm-up heuristics.

### Major redesigns signalled

- **Component Model canonical section refactoring** (Q2/2026) — the `Canon`
  section handling is being redesigned for correctness and performance.
- **Component Model value/type refactoring** (completed Q4/2025) — indicates
  the AST representation is still being stabilized.
- **CLI sub-command redesign** — `lib/driver/` and `lib/po/` will gain a more
  structured sub-command architecture.

---

## 15. Strategic Summary

### Main strengths

- **Spec compliance** — the loader and validator are tested against the full
  official WebAssembly test suite; conformance is a core value.
- **Performance** — the LLVM-based JIT/AOT pipeline produces competitive native
  code; WasmEdge is benchmarked as one of the fastest Wasm runtimes.
- **Extensibility via plugins** — the plugin ABI is stable and well-documented;
  many production plugins exist (WASI-NN, image, crypto).
- **C API stability** — the C API provides a stable ABI that all language
  bindings rely on; it is versioned and breaking changes are managed carefully.
- **CNCF governance** — structured roadmap, quarterly reviews, contributor
  ladder, and code-of-conduct lower the barrier to contribution.
- **Active AI/ML ecosystem** — WASI-NN with llama.cpp integration makes
  WasmEdge uniquely positioned at the intersection of Wasm and LLM inference.

### Biggest architectural challenges

- **Component Model completeness** — the component AST, validator, and executor
  are all partial; until they reach spec completeness, the Component Model
  cannot be relied upon for production use.
- **WASI preview 2 migration** — moving from the well-understood preview 1 ABI
  to the Component-Model-based preview 2 is a multi-quarter effort with broad
  impact on users.
- **Executor complexity** — the instruction dispatch loop is large and
  hand-optimized; it is difficult to add new features (tracing, profiling,
  GC safepoints) without performance regression.
- **Platform divergence in WASI** — `lib/host/wasi/environ.cpp` contains
  significant `#ifdef` branching for Linux, macOS, and Windows; maintaining
  correctness across all three platforms is non-trivial.

### Most impactful areas for contribution

1. **Component Model** — every line of code here directly advances the
   platform's long-term architecture.
2. **WASI preview 2 plugins** — connecting preview 2 to real workloads.
3. **JIT per-function compilation** — unlocks better cold-start performance for
   large modules.
4. **Test coverage and fuzz targets** — directly improves runtime reliability.
5. **Documentation** — the codebase is large; clear documentation multiplies
   contributor effectiveness.

### Fastest way to become a core contributor

1. **Run the spec tests** — understand what they cover and find gaps.
2. **Study `include/runtime/hostfunc.h`** — it is the most important
   extensibility point and understanding it deeply enables plugin development.
3. **Implement one small WASI function** — follow the `Wasi<T>` CRTP pattern
   end-to-end; this teaches loading, validation, instantiation, execution, and
   testing in one exercise.
4. **Pick a Component Model issue** — the Component Model is the highest-impact
   area; active maintainers are highly engaged with contributors here.
5. **Participate in quarterly roadmap discussions** — proposals are community-
   driven; a well-reasoned issue comment can become a roadmap entry.
