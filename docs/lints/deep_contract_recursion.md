# `deep_contract_recursion`

**Default Severity:** `warn`

**Target Resource:** [CPU — Wasm call stack depth and metered instructions](../cost_rationale.md#per-lint-resource-summary)

## What it does

Detects direct recursion in Soroban smart contract functions — i.e., a function that calls itself.

## Why is this bad?

Soroban contracts run inside a WebAssembly virtual machine that imposes strict limits on call stack depth. Each recursive call:

1. **Consumes Wasm stack space** — the call stack is orders of magnitude shallower than a native OS stack; deep recursion quickly triggers a stack overflow.
2. **Burns through the CPU budget** — every recursive dispatch crosses the function call boundary and incurs metered instruction cost.
3. **Prevents static gas estimation** — unbounded recursion makes it impossible for the Soroban host to predict the contract's resource usage before execution.

In practice, any recursive pattern in a Soroban contract function should be rewritten as an iterative loop.

## Example

```rust
// ❌ Bad: direct recursion in a contract function
fn factorial(n: u32) -> u32 {
    if n <= 1 {
        1
    } else {
        n * factorial(n - 1)  // each call adds a stack frame
    }
}
```

```rust
// ✅ Good: rewritten as an iterative loop
fn factorial(n: u32) -> u32 {
    let mut result = 1u32;
    for i in 2..=n {
        result *= i;
    }
    result
}
```

## What is not reported

- Indirect / mutual recursion (function A calls B, B calls A) — this requires cross-function body analysis and is not implemented yet.
- Recursive calls where the callee is from a different crate (e.g. `soroban_sdk::Env::invoke_contract`).
- Functions annotated with `#[allow(deep_contract_recursion)]`.
