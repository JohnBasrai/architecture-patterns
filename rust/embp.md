# Explicit Module Boundary Pattern (EMBP)
*Documented by John Basrai, May 2025. Updated July 2026. This work is licensed under CC BY 4.0*

## 1.0 Overview

The **Explicit Module Boundary Pattern (EMBP)**—also referred to as the **Gateway Module Pattern**—is a Rust architectural pattern for structuring crates using explicit module gateways such as mod.rs, lib.rs, or main.rs. These gateway files define the public API of a module and serve as centralized control points for visibility and inter-module dependencies.

**Pattern Name:**   Explicit Module Boundary Pattern (EMBP)<br>
**Also Known As:**  Gateway Module Pattern<br>
**Acronym:**        EMBP<br>
**Version:**        1.5<br>
**Tested with:**    Rust edition 2024<br>

---

## 2.0 Core Rules

### 2.1 Individual Module Files (Siblings)

Each sibling module:
- exports as `pub` what it needs to share with siblings, crate or workspace
- focuses on their specific domain logic  
- may import symbols from any of their siblings without needing to know their filename
- may import symbols from other modules in the crate without knowledge of the internal structure
- uses visibility modifiers to control access levels (e.g., to current module only or to entire crate)

### 2.2 The Module Boundary (`mod.rs`)

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

### 2.3 Import Patterns Summary

| Context                  | Pattern                      | Example                       |
| ------------------------ | ---------------------------- | ----------------------------- |
| **Binary Entry Point**   | `super::Symbol`              | `use super::{run, init};`     |
| **Sibling to Sibling**   | `crate::Symbol`              | `use crate::Credentials;`     |
| **External Module**      | `crate::Symbol`              | `use crate::AppUser;`         |
| **Binary to Library**    | `crate_name::module::Symbol` | `use myapp::domain::AppUser;` |
| **Library to Workspace** | `use lib::{Symbol}`          | `use lib::{AppId, JobApplication}` |

### 2.4 Visibility Levels

```rust
pub struct Public;            // Visible if re-exported in mod.rs
pub(crate) struct CrateWide;  // Visible within entire crate
pub(super) struct ParentOnly; // Visible to parent module only
struct Private;               // Module-private only
```

### 2.5 Binary Crate Entry Points

For binary crates (`src/bin/name/`) where you want to use EMBP, the entry point file (`main.rs`) can serve dual roles:

- **Binary Entry Point** - Contains `#[tokio::main]` or `fn main()`
- **Module Gateway** - Declares submodules and controls public exports

**⚠️ Important:** This pattern adds complexity and may not be worth it for simple binaries (see section 6.0 "Binary Crate Limitations").

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

### 2.6 Gateway File Naming

The `mod.rs` and sibling-file forms are semantically equivalent to the Rust compiler:

```
src/config/mod.rs   // directory-based gateway (older convention)
src/config.rs       // file-based gateway (Rust 2018+ idiomatic)
```

EMBP uses the sibling-file form (`config.rs`) as the preferred gateway naming convention. It is idiomatic in Rust 2018+ and avoids the ambiguous tab titles that arise when multiple `mod.rs` files are open in an editor simultaneously.

Note: a `src/config.rs` alongside a `src/config/` directory is *not* a violation of EMBP — `config.rs` *is* the module gateway for that directory.

### 2.7 Bare `mod` Declarations

The `mod x;` declaration must always be bare — never `pub mod x` or `pub(crate) mod x`.

Both expose the module *path*, letting callers write `crate::x::Symbol` or `myapp::x::Symbol` and bypass the gateway entirely. Only a bare `mod x;` keeps the module private to the declaring file.

Visibility belongs on the re-export line, not the `mod` line:

```rust
// ❌ Wrong — exposes the module path; allows bypassing the gateway
pub mod alert;
pub(crate) mod common;

// ✅ Correct — module stays private; the use line controls what's visible
mod alert;
mod common;

pub use alert::AlertElem;         // external public API
pub(crate) use common::DataType;  // crate-internal only
```

### 2.8 Three-Source Rule

At any depth in the module tree, a file has exactly three permitted sources for symbols:

1. **Own definitions** — types, functions, and constants defined in the same file
2. **One level down** — immediate child modules declared with `mod child;`, used directly as `child::Type`
3. **Crate root** — `crate::Symbol` (flat, no intermediate path segments)

Everything else is forbidden:

- **No `super::Symbol`** — use `crate::Symbol` instead; the crate root re-export is always reachable and is the correct path
- **No `crate::module::Symbol`** — do not drill past the crate root into a sub-module path
- **No `crate::sibling::Something`** — sibling access must go through the crate root re-export, never directly into a sibling's module path

Import direction is always **upward toward root**. A module never reaches sideways into a sibling or downward into a sibling's children.

```rust
// ❌ Wrong — drilling into a sibling's sub-module
use crate::model::cluster::BatteryCluster;

// ❌ Wrong — super:: instead of going through the crate root
use super::Credentials;

// ✅ Correct — crate root re-export (flat)
use crate::BatteryCluster;
use crate::Credentials;

// ✅ Correct — own immediate child module (one level down)
mod cluster;
use cluster::BatteryCluster;
```

**Why not `super::`?** In a deeply nested module, `super::` reaches the *immediate* parent, not necessarily the crate root. If the parent gateway has not re-exported a symbol, the call fails. And if the parent has re-exported it, `crate::Symbol` already works — so `super::` is never the right choice.

### 2.9 `pub use` vs `pub(crate) use` and Visibility Cascade

Gateway files use two re-export visibility levels:

- `pub use x::Symbol` — **external public API**; visible to consumers outside the crate
- `pub(crate) use x::Symbol` — **crate-internal only**; supports the crate's own implementation; not visible to external consumers

The choice drives a **cascade rule**: visibility must be consistent down the chain. If a gateway re-exports a symbol as `pub(crate) use`, the symbol's definition in its source module must also be `pub(crate)`, not `pub`. A `pub` definition re-exported as `pub(crate) use` is inconsistent — it implies the symbol *could* be public but is being arbitrarily restricted.

```rust
// lib.rs — gateway sets the visibility
mod alert;

pub use alert::AlertElem;           // external API
pub(crate) use alert::AlertState;   // crate-internal

// alert.rs — definition visibility must match
pub struct AlertElem { ... }        // ✅ matches pub use above
pub(crate) enum AlertState { ... }  // ✅ matches pub(crate) use above
```

**Corollary:** If every re-export in a `lib.rs` is `pub(crate) use`, the crate exposes no external public API. A workspace gateway crate that depends on it must provide its own explicit `pub use` re-exports to restore external access. This is intentional when a crate is purely an implementation detail with another crate acting as its public face.

---

## 3.0 Advanced Patterns

### 3.1 Workspace Gateway Pattern

In multi-crate workspaces, one crate can serve as the primary public API gateway for related functionality:

```rust
// lib/src/lib.rs - Workspace gateway crate
mod domain;
mod repository;

// Re-export domain types for easy access
pub use domain::{
    AppId, JobApplication, JobApplicationPtr,
    ApplicationRepository, ApplicationRepositoryPtr,
};

// Re-export selected repository functions
pub use repository::{create_memory_repository, create_mongo_repository};
```

This allows other workspace members to import cleanly:

```rust
// services/src/application_service.rs
use lib::{AppId, JobApplication, ApplicationRepositoryPtr};
// Instead of: use lib::domain::{AppId, JobApplication};
//             use lib::repository::ApplicationRepositoryPtr;
```

**Benefits:**
- **Cross-crate consistency**: All workspace members use the same import paths
- **Refactoring isolation**: Internal reorganization of lib crate doesn't break dependents
- **API control**: Gateway crate controls exactly what's exposed to workspace

### 3.2 Trait Pointer Co-location

When defining traits, co-locate the corresponding Arc pointer type in the same module:

```rust
// domain/repository.rs
pub trait ApplicationRepository: Send + Sync {
    async fn create(&self, app: JobApplicationPtr) -> Result<()>;
    // ... other methods
}

// Pointer type immediately follows trait definition
pub type ApplicationRepositoryPtr = Arc<dyn ApplicationRepository + Send + Sync>;
```

**Benefits:**
- **Logical grouping**: Trait and its pointer type are always together
- **Import simplicity**: Single import gets both trait and pointer
- **Maintenance**: Changes to trait bounds automatically apply to pointer type

### 3.3 Module Visibility Strategy

EMBP uses private module declarations with selective public exports:

```rust
// services/src/lib.rs - Correct EMBP pattern
mod application_service;  // Private module
mod contact_service;      // Private module  
mod config;               // Private module

// Public exports - gateway controls exactly what's visible
pub use application_service::ApplicationService;
pub use contact_service::ContactService;
pub use config::{ServiceConfig, RepositoryType};
```

**Why Private Modules:**
- **Gateway control**: Only the gateway decides what's public
- **Prevents bypassing**: No way to access internal module structure
- **Clean boundaries**: External code must use the intended API
- **Refactoring freedom**: Internal module organization can change without breaking external code

**❌ Anti-pattern: Public Modules**
```rust
// DON'T: This defeats the purpose of EMBP
pub mod application_service;  // Allows bypassing the gateway
pub mod contact_service;      // External code can import directly
```

Using `pub mod` allows external code to bypass the gateway with imports like `use services::application_service::SomeInternalType`, which violates EMBP principles.

### 3.4 External Crate Type Aliases

Gateways can provide semantic aliases for types from external crates:

```rust
// services/src/lib.rs - Aliases for external crate types
pub type ServiceResult<T> = anyhow::Result<T>;
pub type ServiceError = anyhow::Error;

// External usage becomes more semantic and stable
// fn create_user() -> ServiceResult<User> instead of anyhow::Result<User>
```

**This differs from trait pointers (3.2):**
- **Trait pointers**: Aliases for traits we define in our crate
- **External aliases**: Semantic names for types from crates.io dependencies

**Benefits:**
- **Semantic clarity**: `ServiceResult<T>` is more domain-specific than `anyhow::Result<T>`
- **Dependency hiding**: External code doesn't directly depend on `anyhow`
- **API stability**: Can change underlying error crate without breaking external API
- **Consistency**: All service functions use the same result type naming

---

## 4.0 Benefits

1. **Explicit Dependencies** - All inter-module dependencies visible in `mod.rs`
2. **Controlled Boundaries** - Clear separation between public API and internals
3. **Refactoring Safety** - Changes to internals don't break external consumers
4. **Complete Internal Refactoring Freedom** - Rename files, reorganize modules, move code around within a module without breaking any external code. Only need to update the `mod.rs` gateway, not hunt down imports throughout the codebase
5. **Eliminates Brittle Deep Imports** - Siblings access each other through gateways using simple `super::Symbol` paths instead of fragile direct file access. When internal structure changes, only the gateway needs updating
6. **Documentation** - Gateway files (`mod.rs`, `main.rs`, `lib.rs`) serve as module documentation
7. **Layered Architecture** - Natural enforcement of architectural layers
8. **Binary Organization** - Clean separation between entry point logic and implementation
9. **Workspace API Control** - Gateway crates provide unified APIs across workspace members

**Note:** While EMBP theoretically supports deeper module hierarchies (e.g., `crate::domain::user::profile::Symbol`), this pattern has been primarily tested and validated with single-level gateway access patterns.

---

## 5.0 Example Structure

### 5.1 Single Crate Example

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

### 5.2 Workspace Example

```
job-tracker/
├── Cargo.toml          ← Workspace manifest
├── src/
│   └── main.rs         ← Binary entry point (jt command)
├── lib/
│   ├── Cargo.toml      ← Domain model crate
│   └── src/
│       ├── lib.rs      ← Workspace gateway
│       ├── domain/
│       │   ├── mod.rs  ← Domain gateway
│       │   ├── app_id.rs
│       │   └── job_application.rs
│       └── repository/
│           ├── mod.rs  ← Repository gateway (private)
│           ├── memory.rs
│           └── mongo.rs
└── services/
    ├── Cargo.toml      ← Business logic crate
    └── src/
        ├── lib.rs      ← Services gateway
        ├── application_service.rs
        └── contact_service.rs
```

---

## 6.0 Binary Crates: Practical Limitations and Workarounds

**⚠️ EMBP may not be suitable for simple binary crates** due to Rust's module resolution constraints.

### 6.1 Issues Encountered

- **Module resolution conflicts** - Binary entry points have different scoping rules than library modules
- **"Too many leading `super` keywords"** errors when trying to import through gateways
- **Import path complexity** - `crate::` vs `super::` resolution varies between binary and library contexts

### 6.2 When EMBP Still Applies to Binaries and Multi-Library Workspaces

**Binary Applications:**
- **Complex module hierarchies** requiring strict boundaries
- **Shared binary/library codebases** where consistency matters
- **When module organization becomes difficult to manage** (team and project dependent)

**Multi-Library Workspaces:**
- Each library crate has its own EMBP gateway (`lib.rs`)
- Internal modules use EMBP gateways (`mod.rs`)
- Clear dependency direction between libraries (see section 5.2 workspace example with `lib` and `services` crates)
- Workspace-level coordination through a primary gateway crate

**Considerations:**
- **Team size**: Larger teams benefit more from explicit boundaries
- **Module complexity**: Business logic complexity matters more than raw module count
- **Shared patterns**: If the project uses EMBP elsewhere, consistency may be valuable
- **Future growth**: Anticipating growth may justify early adoption

**Scale considerations:** EMBP may become valuable with as few as 5+ modules, depending on complexity. At very large scales (15+ modules), EMBP could either become critical for managing complexity or too cumbersome to be practical - this remains unexplored territory.

**Recommendation:** Use EMBP for library crates first. For binaries and multi-library workspaces, evaluate based on actual complexity and team needs rather than arbitrary module counts.

---

## 7.0 Anti-Patterns to Avoid

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

❌ **`pub mod` or `pub(crate) mod` Declarations**

```rust
// DON'T: exposes the module path — callers can bypass the gateway
pub mod alert;
pub(crate) mod common;
```

✅ **Bare `mod` Declaration**

```rust
// DO: module stays private; gateway controls what's visible via use lines
mod alert;
mod common;
```

❌ **`super::` for Cross-Module Access**

```rust
// DON'T: super:: may not reach the right level and hides intent
use super::Credentials;
use super::model::Config;  // especially wrong — drilling through super
```

✅ **`crate::Symbol` for Cross-Module Access**

```rust
// DO: always go through the crate root re-export (flat)
use crate::Credentials;
use crate::Config;
```

❌ **Workspace Gateway Bypassing**

```rust
// DON'T: Bypass workspace gateway for deep imports
use lib::domain::job_application::JobApplication;
use lib::repository::memory::MemoryRepository;
```

✅ **Workspace Gateway Usage**

```rust
// DO: Use workspace gateway exports
use lib::{JobApplication, JobApplicationPtr, create_memory_repository};
```

---

## 8.0 Pattern Limitations

**⚠️ Convention-Based, Not Enforced:** EMBP relies on team discipline rather than compiler enforcement. Internal items must be `pub` for gateways to re-export them, which means external code *can* bypass gateways and access internals directly. The pattern's benefits depend on following the convention consistently.

**Mitigation:** Use code reviews, linting rules, or team guidelines to catch gateway bypassing. Consider `pub(crate)` for items that should only be visible within the current crate. Custom tooling could be developed to automatically detect and flag imports that bypass `mod.rs` gateways. Checking the module gateway files is a quick check.

---

## 8.1 Relationship to Other Rust Design Frameworks

EMBP is a **module organization pattern** that works in concert with broader API design principles. Here's how it relates to other frameworks:

| Dimension | EMBP | Gjengset's Four Pillars | Official Guidelines | Rust Book |
|-----------|------|-------------------------|-------------------|-----------|
| **Focus** | Module boundaries & gating; import discipline | API design quality (unsurprising, flexible, obvious, constrained) | Standards for public APIs (documentation, naming, type safety) | Foundational language concepts & patterns |
| **Scope** | How modules expose/hide internals | How types and functions feel to users | Documentation, naming conventions, error handling | Ownership, borrowing, modules (basic) |
| **Private Fields** | Enables via private module declarations + gateways | Supports via encapsulation (less surprising, more constrained) | Encourages via accessibility levels | Explains how `pub struct` fields are accessible |
| **Cross-Module Access** | Three-source rule; `crate::` from root | N/A (API-level concern, not module-level) | Prefer shallow, explicit imports | N/A (not about module design) |
| **Type Safety** | Prevents bypassing via module gating | Constrains through type system (newtype pattern) | Uses type safety to guide users | Foundation for all type safety |
| **Validation** | Work-ledger refactoring (HIGH priority: private fields + accessors ~30 changes) | Work-ledger audit findings (MEDIUM: newtype wrappers ~81 changes; LOW: field naming) | Checked in code review; linting (clippy) | Foundational; verified by compiler |

### Complementary Approaches

**EMBP + Gjengset's Framework:**
- EMBP controls what's publicly *accessible* (module gating)
- Gjengset's pillars control how public APIs *feel* (encapsulation, type clarity)
- **Together:** Private fields with accessor methods satisfy both—module boundaries (EMBP) + obvious/constrained design (Gjengset)
- **Example:** `ChargeCode` struct: private fields (EMBP) + `pjn()`, `subtask()`, `description()` accessors (Gjengset's "obvious, constrained")

**EMBP + Official Guidelines:**
- EMBP structures the module hierarchy
- Official Guidelines address documentation, error handling, naming within that hierarchy
- **Together:** Create a complete, professional API
- **Example:** Clear `mod.rs` gateways (EMBP) + doc comments on public items (Guidelines)

**EMBP + Rust Book:**
- Rust Book teaches language fundamentals (ownership, borrowing, modules)
- EMBP applies those fundamentals to large-scale architecture
- **Example:** The Book explains `pub struct` fields; EMBP applies that to multi-module design

### Validation in work-ledger (2026-07-04)

Three audit passes informed the refactoring:

1. **EMBP audit** — Applied to all five gateway modules (`main.rs`, `db/mod.rs`, `commands/mod.rs`, `output/mod.rs`, `tui/mod.rs`)
   - Result: Private module declarations + selective `pub use` re-exports ✓
   - Import discipline: `crate::Symbol` for cross-module access ✓

2. **Gjengset's Four Pillars audit** — Evaluated all public types
   - **Unsurprising:** ChargeCode/ReportRow field mutations prevented → private fields
   - **Flexible:** Accessor methods allow future implementation changes
   - **Obvious:** Public methods describe intent (e.g., `percent_of_week()` vs. `pct`)
   - **Constrained:** Private fields prevent accidental direct access
   - Result: HIGH priority (private fields + accessors) completed; MEDIUM (newtype wrappers) deferred; LOW (polish) deferred

3. **Official Rust API Guidelines audit** — Naming, documentation, stability
   - Result: Naming improvements (e.g., `pct` → `percent_of_week`) included in HIGH priority

---

## 9.0 Implementation Checklist

### 9.1 Module Structure

- [ ] Each `mod.rs` has explicit `mod` declarations for all submodules. Using `mod X`, and not `pub mod X`
- [ ] Each `mod.rs` has clear separation of internal `use` vs public `pub use`

### 9.2 Import Discipline

- [ ] Sibling modules import from each other using `crate::Symbol` (never `super::` or `crate::module::Symbol`)
- [ ] No `crate::module::Symbol` paths — all cross-module access goes through the crate root re-export
- [ ] No direct imports that bypass `mod.rs` gateways
- [ ] Workspace members use gateway crate exports, not deep imports

### 9.3 Visibility Hygiene

- [ ] Public APIs only expose public types, never internals
- [ ] Trait and pointer types co-located in same module
- [ ] Mixed visibility strategies used intentionally, not accidentally

### 9.4 Workspace Patterns

- [ ] Workspace gateway crate provides unified API
- [ ] Cross-crate imports use gateway exports
- [ ] Gateway crate controls workspace API surface

### 9.5 Binary Crate Considerations

- [ ] Consider binary crate limitations – use standard patterns for simple binaries

### 9.6 Consistency & Discipline

- [ ] Following EMBP principles consistently across the codebase

---

## 10.0 When to Use EMBP

✅ **Good for:**
- Multi-module applications
- Library crates with public APIs
- Projects requiring clear architectural boundaries
- Teams needing explicit dependency management
- Workspace projects with shared APIs
- Complex domain models with multiple bounded contexts

❌ **Overkill for:**
- Single-file applications  
- Prototypes
- Very simple projects with minimal modules
- Simple binary crates (3-5 modules)
- Throwaway scripts

---

## 11.0 Real-World Example: Job Tracker

A complete workspace implementation showing EMBP patterns:

```rust
// lib/src/lib.rs - Primary workspace gateway
pub mod domain;
mod repository;

pub use domain::{
    AppId, JobApplication, JobApplicationPtr,
    ApplicationRepository, ApplicationRepositoryPtr,
};
pub use repository::{create_memory_repository, create_mongo_repository};

// services/src/lib.rs - Service layer gateway  
mod application_service;
mod contact_service;
mod config;

pub use application_service::ApplicationService;
pub use contact_service::ContactService;
pub use config::{ServiceConfig, RepositoryType};

// services/src/application_service.rs - Clean cross-crate imports
use lib::{AppId, JobApplication, ApplicationRepositoryPtr};
use crate::ServiceResult;

pub struct ApplicationService {
    repo: ApplicationRepositoryPtr,
}
```

This structure allows the entire workspace to use clean, stable imports while maintaining complete internal refactoring freedom within each crate.
