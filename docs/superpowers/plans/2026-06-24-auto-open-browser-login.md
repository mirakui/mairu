# Auto-open Browser for `mairu login` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Automatically open the authentication URL in the default browser during `mairu login` (and `mairu exec`-triggered login), defaulting to on and disable-able via `--no-browser` / `MAIRU_NO_BROWSER`.

**Architecture:** A new client-side module `src/browser.rs` exposes a pure, unit-tested decision function `should_open(no_browser, is_terminal)` and a thin best-effort `open_url(url)` that wraps the `webbrowser` crate in `spawn_blocking`. The two login flow handlers in `src/cmd/login.rs` print the URL as before, then conditionally open it. A `--no-browser`/`MAIRU_NO_BROWSER` arg is added to both `LoginArgs` and `ExecArgs`.

**Tech Stack:** Rust, tokio, clap (derive + `env`), `webbrowser` 1.2.1, `tracing`, `indoc`.

**Spec:** `docs/superpowers/specs/2026-06-24-auto-open-browser-login-design.md`

---

## File Structure

- `Cargo.toml` — add `webbrowser = "1.2.1"` dependency.
- `src/browser.rs` (new) — `should_open()` (pure, tested) + `open_url()` (best-effort launch).
- `src/lib.rs` — register `pub mod browser;`.
- `src/cmd/login.rs` — add `no_browser` to `LoginArgs`; thread into `do_oauth_code` / `do_oauth_device_code`; call the browser module.
- `src/cmd/exec.rs` — add `no_browser` to `ExecArgs`; propagate into the constructed `LoginArgs`.
- `README.md` — document auto-open behavior and how to disable it.

---

## Task 1: Add the `webbrowser` dependency

**Files:**
- Modify: `Cargo.toml` (alphabetical `[dependencies]` block, between `url` and `zeroize` — actually `webbrowser` sorts after `url` and before `zeroize`)

- [ ] **Step 1: Add the dependency**

In `Cargo.toml`, the `[dependencies]` table is roughly alphabetical. Insert the following line after the `url = { ... }` line:

```toml
webbrowser = "1.2.1"
```

- [ ] **Step 2: Fetch and verify it builds**

Run: `cargo build`
Expected: Compiles successfully; `Cargo.lock` updated with `webbrowser` 1.2.x.

- [ ] **Step 3: Commit**

```bash
git add Cargo.toml Cargo.lock
git commit -m "build: add webbrowser crate dependency (#50)"
```

---

## Task 2: Create `src/browser.rs` with the decision function (TDD)

**Files:**
- Create: `src/browser.rs`
- Modify: `src/lib.rs` (add module registration)
- Test: inline `#[cfg(test)] mod tests` in `src/browser.rs`

- [ ] **Step 1: Register the module**

In `src/lib.rs`, add this line alongside the other top-level utility modules (e.g. right after `pub mod auto;` / near `pub mod utils;`):

```rust
pub mod browser;
```

- [ ] **Step 2: Write the failing test**

Create `src/browser.rs` with ONLY the test module and a stub so it compiles-and-fails meaningfully. Start with:

```rust
/// Decide whether the authentication URL should be opened in a browser.
///
/// Auto-open is enabled by default. It is suppressed when the user opted out
/// (`--no-browser` / `MAIRU_NO_BROWSER`) or when stdout/tty is not a real
/// terminal (e.g. CI, piped output), in which case only the printed URL is
/// used.
pub fn should_open(no_browser: bool, is_terminal: bool) -> bool {
    // intentionally wrong to make the test fail first
    false
}

#[cfg(test)]
mod tests {
    mod should_open {
        #[test]
        fn opens_when_enabled_and_terminal() {
            assert!(crate::browser::should_open(false, true));
        }

        #[test]
        fn does_not_open_when_no_browser() {
            assert!(!crate::browser::should_open(true, true));
        }

        #[test]
        fn does_not_open_when_not_terminal() {
            assert!(!crate::browser::should_open(false, false));
        }

        #[test]
        fn does_not_open_when_no_browser_and_not_terminal() {
            assert!(!crate::browser::should_open(true, false));
        }
    }
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `cargo test --lib browser::tests::should_open`
Expected: FAIL — `opens_when_enabled_and_terminal` panics (assert! on `false`).

- [ ] **Step 4: Implement the minimal correct logic**

Replace the body of `should_open` so it returns the real decision:

```rust
pub fn should_open(no_browser: bool, is_terminal: bool) -> bool {
    !no_browser && is_terminal
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `cargo test --lib browser::tests::should_open`
Expected: PASS — all four tests pass.

- [ ] **Step 6: Add the best-effort `open_url` function**

Append the launch function to `src/browser.rs` (above the `#[cfg(test)]` module). It is intentionally thin and not unit-tested (it would spawn a real browser); all branching logic lives in `should_open`.

```rust
/// Best-effort: open `url` in the user's default browser.
///
/// Never returns an error to the caller — the URL is always printed first as a
/// fallback, so a failure to launch a browser (no DISPLAY, no browser, launcher
/// error) must not fail the login. `webbrowser::open_browser_with_options` is
/// synchronous and may spawn a subprocess, so it runs on a blocking thread.
pub async fn open_url(url: &str) {
    let url = url.to_owned();
    let result = tokio::task::spawn_blocking(move || {
        webbrowser::open_browser_with_options(
            webbrowser::Browser::Default,
            &url,
            webbrowser::BrowserOptions::new().with_suppress_output(true),
        )
    })
    .await;
    match result {
        Ok(Ok(())) => {}
        Ok(Err(e)) => tracing::warn!("Failed to open browser: {e}"),
        Err(e) => tracing::warn!("Failed to spawn browser-open task: {e}"),
    }
}
```

- [ ] **Step 7: Verify it compiles and tests still pass**

Run: `cargo test --lib browser`
Expected: PASS (the four `should_open` tests; `open_url` has no tests).

- [ ] **Step 8: Commit**

```bash
git add src/browser.rs src/lib.rs
git commit -m "feat: add browser module to open auth URLs (#50)"
```

---

## Task 3: Add `--no-browser` to `mairu login` and wire both flows

**Files:**
- Modify: `src/cmd/login.rs:1-9` (add field to `LoginArgs`)
- Modify: `src/cmd/login.rs:50-53` (pass `args.no_browser` to handlers)
- Modify: `src/cmd/login.rs:56-121` (`do_oauth_code` signature + open)
- Modify: `src/cmd/login.rs:123-183` (`do_oauth_device_code` signature + open)

- [ ] **Step 1: Add the `no_browser` field to `LoginArgs`**

In `src/cmd/login.rs`, change the `LoginArgs` struct (currently lines 1-9) to add the field after `oauth_grant_type`:

```rust
#[derive(clap::Args)]
pub struct LoginArgs {
    /// Override OAuth 2 grant type to use.
    #[arg(long)]
    pub oauth_grant_type: Option<crate::config::OAuthGrantType>,

    /// Do not automatically open the authentication URL in a browser.
    #[arg(long, env = "MAIRU_NO_BROWSER", default_value_t = false)]
    pub no_browser: bool,

    /// Credential server ID or URL to use.
    pub server_name: String,
}
```

- [ ] **Step 2: Pass `no_browser` into the flow handlers**

In `login()`, change the match (currently lines 50-53):

```rust
    match oauth_grant_type {
        crate::config::OAuthGrantType::Code => {
            do_oauth_code(agent, server, args.no_browser).await
        }
        crate::config::OAuthGrantType::DeviceCode => {
            do_oauth_device_code(agent, server, args.no_browser).await
        }
    }
```

- [ ] **Step 3: Add the parameter and the open call to `do_oauth_code`**

Change the signature of `do_oauth_code` (currently lines 56-59) to add `no_browser: bool`:

```rust
pub async fn do_oauth_code(
    agent: &mut crate::agent::AgentConn,
    server: crate::config::Server,
    no_browser: bool,
) -> Result<(), anyhow::Error> {
```

Then, immediately after the existing URL message `crate::terminal::send(&indoc::formatdoc! { ... }).await;` block (currently ending at line 116) and BEFORE the `crate::oauth_code::listen_for_callback(...)` call (line 118), insert:

```rust
    if crate::browser::should_open(no_browser, crate::terminal::is_terminal().await) {
        crate::terminal::send(&indoc::formatdoc! {"
            :: {product} :: Attempting to open the URL in your browser...
        "})
        .await;
        crate::browser::open_url(short_authorize_url.as_str()).await;
    }
```

Note: `short_authorize_url` is a `url::Url`, so use `.as_str()`. `product` is already bound earlier in the function (line 100).

- [ ] **Step 4: Add the parameter and the open call to `do_oauth_device_code`**

Change the signature of `do_oauth_device_code` (currently lines 123-126) to add `no_browser: bool`:

```rust
pub async fn do_oauth_device_code(
    agent: &mut crate::agent::AgentConn,
    server: crate::config::Server,
    no_browser: bool,
) -> Result<(), anyhow::Error> {
```

Then, immediately after the existing URL message block (the `crate::terminal::send(...).await;` that ends at line 155) and BEFORE the `let mut interval = ...` line (line 157), insert:

```rust
    if crate::browser::should_open(no_browser, crate::terminal::is_terminal().await) {
        crate::terminal::send(&indoc::formatdoc! {"
            :: {product} :: Attempting to open the URL in your browser...
        "})
        .await;
        crate::browser::open_url(authorize_url).await;
    }
```

Note: `authorize_url` is `&String` (bound at lines 143-146); it coerces to `&str` via deref. `product` is bound at line 139.

- [ ] **Step 5: Verify it compiles**

Run: `cargo build`
Expected: Compiles successfully.

- [ ] **Step 6: Verify clippy and tests are clean**

Run: `cargo clippy --all-targets -- -D warnings && cargo test --lib browser`
Expected: No clippy warnings; browser tests pass.

- [ ] **Step 7: Commit**

```bash
git add src/cmd/login.rs
git commit -m "feat: auto-open auth URL on mairu login (#50)"
```

---

## Task 4: Add `--no-browser` to `mairu exec` and propagate

**Files:**
- Modify: `src/cmd/exec.rs:2-66` (add field to `ExecArgs`)
- Modify: `src/cmd/exec.rs:508-513` (propagate into constructed `LoginArgs`)

- [ ] **Step 1: Add the `no_browser` field to `ExecArgs`**

In `src/cmd/exec.rs`, add the following field to the `ExecArgs` struct (place it near `oauth_grant_type`, currently lines 29-31, keeping related login flags together):

```rust
    /// Do not automatically open the authentication URL in a browser when logging in.
    #[arg(long, env = "MAIRU_NO_BROWSER", default_value_t = false)]
    no_browser: bool,
```

- [ ] **Step 2: Propagate into the constructed `LoginArgs`**

In the `login()` function in `exec.rs` (currently lines 508-513), the `LoginArgs` is constructed as:

```rust
        let login_args = crate::cmd::login::LoginArgs {
            oauth_grant_type: args.oauth_grant_type,
            server_name: args.server.as_ref().unwrap().to_owned(),
        };
```

Change it to include `no_browser`:

```rust
        let login_args = crate::cmd::login::LoginArgs {
            oauth_grant_type: args.oauth_grant_type,
            no_browser: args.no_browser,
            server_name: args.server.as_ref().unwrap().to_owned(),
        };
```

- [ ] **Step 3: Verify it compiles**

Run: `cargo build`
Expected: Compiles successfully (no "missing field `no_browser`" error from the `LoginArgs` literal).

- [ ] **Step 4: Commit**

```bash
git add src/cmd/exec.rs
git commit -m "feat: propagate --no-browser through mairu exec login (#50)"
```

---

## Task 5: Document the behavior in README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add a note about auto-open**

Find the section describing `mairu login` / how login prompts work (near `README.md:25`, which mentions "prompts user to login"). Add a short paragraph documenting:

```markdown
When you run `mairu login` (or when `mairu exec` triggers a login), Mairu now
opens the authentication URL in your default browser automatically. The URL is
always printed too, so you can copy it manually if needed (for example over SSH
without a display). To disable automatic opening, pass `--no-browser` or set the
`MAIRU_NO_BROWSER` environment variable.
```

Adapt the wording/placement to match the surrounding README style.

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: document auto-open browser behavior for login (#50)"
```

---

## Task 6: Final verification

**Files:** none (verification only)

- [ ] **Step 1: Format check**

Run: `cargo fmt --check`
Expected: No diff. (If it reports changes, run `cargo fmt` and amend/commit.)

- [ ] **Step 2: Full clippy**

Run: `cargo clippy --all-targets -- -D warnings`
Expected: No warnings.

- [ ] **Step 3: Full test suite**

Run: `cargo test`
Expected: All tests pass, including `browser::tests::should_open::*`.

- [ ] **Step 4: Manual smoke test (best-effort, requires a configured server)**

- `mairu login <server>` — browser opens AND the URL is printed.
- `mairu login --no-browser <server>` — browser does NOT open; URL is printed.
- `MAIRU_NO_BROWSER=1 mairu login <server>` — browser does NOT open; URL is printed.
- Login over SSH without a display — browser open fails silently (a `tracing::warn!` may appear with `RUST_LOG`), login still proceeds via the printed URL.

---

## Self-Review Notes

- **Spec coverage:** Cargo dep (Task 1) ✓; `browser.rs` module with `should_open` + `open_url` (Task 2) ✓; `--no-browser`/`MAIRU_NO_BROWSER` on `LoginArgs` (Task 3) and `ExecArgs` (Task 4) ✓; both Code and Device Code flows wired (Task 3) ✓; URL always printed first + `is_terminal` gate (Task 3) ✓; unit tests for `should_open` (Task 2) ✓; README (Task 5) ✓; DoD build/test/clippy/fmt (Task 6) ✓.
- **Type consistency:** `should_open(no_browser: bool, is_terminal: bool) -> bool` and `open_url(url: &str)` used identically in Tasks 2-4. `short_authorize_url` is `url::Url` → `.as_str()`; `authorize_url` is `&String` → deref-coerces to `&str`.
- **No placeholders:** every code step shows the real code; commands have expected output.
