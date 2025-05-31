# Explicit Module Boundary Pattern (EMBP)

*Documented by John Basrai, May 2025. This work is licensed under CC BY 4.0*

## Overview

The **Explicit Module Boundary Pattern (EMBP)**—also referred to as the **Gateway Module Pattern**—is a Rust architectural pattern for structuring crates using explicit module gateways such as mod.rs, lib.rs, or main.rs. These gateway files define the public API of a module and serve as centralized control points for visibility and inter-module dependencies.

**Pattern Name:**   Explicit Module Boundary Pattern (EMBP)<br>
**Also Known As:**  Gateway Module Pattern<br>
**Acronym:**        EMBP<br>
**Version:**        1.1<br>
**Tested with:**    Rust edition 2021<br>

---

## Core Rules

### 1. Individual Module Files (Siblings)

- Each sibling module exports as `pub` what it needs to share with siblings OR the crate
- Individual files focus on their specific domain logic
- May import symbols from any of their siblings without needing to know their filename
- May import symbols from other modules in the crate without knowledge of the internal structure
- Use visibility modifiers to control access levels e.g., to current module only (siblings) or to entire crate

### 2. The Module Boundary (`mod.rs`)

The `mod.rs` file serves as the module's public contract:

```rust
// Bring all submodules into scope
mod submodule_a;
mod submodule_b;

// Internal-only exports (sibling access within this module)
use submodule_a::InternalType;

// Public exports (visible outside this module)
pub use submodule_a::{PublicType, public_function};
pub use submodule_b::{AnotherPublicType};
```

**The `mod.rs` file defines the ENTIRE public interface** - if it's not in `mod.rs`, it's not public.

### 3. Import Patterns Summary

| Context                | Pattern                      | Example                       |
| ---------------------- | ---------------------------- | ----------------------------- |
| **Binary Entry Point** | `super::Symbol`              | `use super::{run, init};`     |
| **Sibling to Sibling** | `super::Symbol`              | `use super::Credentials;`     |
| **External Module**    | `crate::module::Symbol`      | `use crate::domain::AppUser;` |
| **Binary to Library**  | `crate_name::module::Symbol` | `use myapp::domain::AppUser;` |

### 4. Visibility Levels

```rust
pub struct Public;           // Visible if re-exported in mod.rs
pub(crate) struct CrateWide; // Visible within entire crate
pub(super) struct ParentOnly; // Visible to parent module only
struct Private;              // Module-private only
```

### 5. Binary Crate Entry Points

For binary crates (`src/bin/name/`) where you want to use EMBP, the entry point file (`main.rs`) can serve dual roles:

- **Binary Entry Point** - Contains `#[tokio::main]` or `fn main()`
- **Module Gateway** - Declares submodules and controls public exports

**⚠️ Important:** This pattern adds complexity and may not be worth it for simple binaries (see "Binary Crate Limitations" below).

**⚠️ Technical constraint:** When using EMBP with binary crates, the entry point file **must** be named `main.rs`. If you use a different filename (e.g., `server.rs`) to hold your `main()` function, the Rust compiler will have difficulty with module resolution and imports, leading to errors like "too many leading `super` keywords" or "unresolved imports."

```rust
// src/bin/server/main.rs  ✅ IF using EMBP
// Dual role: entry point + gateway

mod diagnostics;
mod server;
mod server_cli_args;

// Gateway exports
pub use diagnostics::generate_report;
pub use server_cli_args::Cli;

#[tokio::main]
async fn main() -> Result<(), anyhow::Error> {
    server::run().await  // Delegate to implementation
}
```

---

## Benefits

1. **Explicit Dependencies** - All inter-module dependencies visible in `mod.rs`
2. **Controlled Boundaries** - Clear separation between public API and internals
3. **Refactoring Safety**    - Changes to internals don't break external consumers
4. **Complete Internal Refactoring Freedom** - Rename files, reorganize modules, move code around within a module without breaking any external code. Only need to update the `mod.rs` gateway, not hunt down imports throughout the codebase
5. **Eliminates Brittle Deep Imports** - Siblings access each other through gateways using simple `super::Symbol` paths instead of fragile direct file access. When internal structure changes, only the gateway needs updating
6. **Documentation**         - Gateway files (`mod.rs`, `main.rs`, `lib.rs`) serve as module documentation
7. **Layered Architecture**  - Natural enforcement of architectural layers
8. **Binary Organization**   - Clean separation between entry point logic and implementation

**Note:** While EMBP theoretically supports deeper module hierarchies (e.g., `crate::domain::user::profile::Symbol`), this pattern has been primarily tested and validated with single-level gateway access patterns.

---

## Example Structure

```
src/
├── domain/
│   ├── mod.rs          ← Gateway: defines public domain API
│   ├── user.rs         ← Sibling: user logic
│   ├── auth.rs         ← Sibling: auth logic  
│   └── cache.rs        ← Sibling: cache logic
├── repository/
│   ├── mod.rs          ← Gateway: defines public repository API
│   ├── user_repo.rs    ← Implementation
│   └── cache_repo.rs   ← Implementation
└── routes/
    ├── mod.rs          ← Gateway: defines public routes API
    └── user_routes.rs  ← Route handlers
```

---

## Binary Crates: Practical Limitations and Workarounds

**⚠️ EMBP may not be suitable for simple binary crates** due to Rust's module resolution constraints.

### Issues Encountered:

- **Module resolution conflicts** - Binary entry points have different scoping rules than library modules
- **"Too many leading `super` keywords"** errors when trying to import through gateways
- **Import path complexity** - `crate::` vs `super::` resolution varies between binary and library contexts

```rust
// src/bin/server/main.rs - Simple entry point
mod server;

#[tokio::main]
async fn main() -> Result<(), anyhow::Error> {
    server::run().await
}

// src/bin/server/server.rs - Self-contained implementation
mod diagnostics;
mod cli_args;

use diagnostics::generate_report;
use cli_args::Cli;
// ... implementation
```

### When EMBP Still Applies to Binaries:

- **Large binary applications** (10+ modules)
- **Complex module hierarchies** requiring strict boundaries
- **Shared binary/library codebases** where consistency matters

**Recommendation:** Use EMBP for library crates and large applications. For simple binaries, standard Rust module patterns often work better.

---

## Anti-Patterns to Avoid

❌ **Bypassing the Gateway**

```rust
// DON'T: Direct access bypasses mod.rs contract
use crate::domain::user::UserInternal;
```

✅ **Use the Gateway**

```rust
// DO: Access through public API
use crate::domain::User;
```

❌ **Drilling Past Gateways**

```rust
// DON'T: Bypass intermediate gateways to reach implementation files
use crate::domain::user::profile::settings::DisplayPreference;
```

✅ **Let Gateways Hoist Symbols**

```rust
// DO: Access symbols hoisted up by gateways
use crate::domain::DisplayPreference;
```

❌ **Leaky Abstractions**

```rust
// DON'T: Expose internal types in public signatures
pub fn get_user() -> InternalUserType { }
```

✅ **Clean Boundaries**

```rust
// DO: Only public types in public signatures  
pub fn get_user() -> User { }
```

---

## Pattern Limitations

**⚠️ Convention-Based, Not Enforced:** EMBP relies on team discipline rather than compiler enforcement. Internal items must be `pub` for gateways to re-export them, which means external code *can* bypass gateways and access internals directly. The pattern's benefits depend on following the convention consistently.

**Mitigation:** Use code reviews, linting rules, or team guidelines to catch gateway bypassing. Consider `pub(crate)` for items that should only be visible within the current crate. Custom tooling could be developed to automatically detect and flag imports that bypass `mod.rs` gateways.

---

## Implementation Checklist

### Mod.rs Structure

These rules focus on defining and organizing the module boundary file itself:

### 📁 `mod.rs` Structure

- [ ] Each `mod.rs` has explicit `mod` declarations for all submodules
- [ ] Each `mod.rs` has clear separation of internal `use` vs public `pub use`

### 🔀 Import Discipline

- [ ] Sibling modules import from each other using `super::`
- [ ] External modules import through `crate::module::`
- [ ] No direct imports that bypass `mod.rs` gateways

### 🔒 Visibility Hygiene

- [ ] Public APIs only expose public types, never internals

### ⚠️ Binary Crate Considerations

- [ ] Consider binary crate limitations – use standard patterns for simple binaries

### 🧱 Consistency & Discipline

- [ ] Following EMBP principles consistently across the codebase

---

## When to Use EMBP

✅ **Good for:**
- Multi-module applications
- Library crates with public APIs
- Projects requiring clear architectural boundaries
- Teams needing explicit dependency management

❌ **Overkill for:**
- Single-file applications  
- Prototypes
- Very simple projects with minimal modules
- Simple binary crates (3-5 modules)

