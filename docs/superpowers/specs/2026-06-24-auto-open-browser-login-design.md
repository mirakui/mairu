# Design: Automatically open browser for `mairu login` authentication URL

- Issue: https://github.com/sorah/mairu/issues/50
- Date: 2026-06-24
- Status: Approved (design)

## Background

When running `mairu login`, the user must manually copy/click the printed
authentication URL to continue:

```
:: mairu ::
:: mairu :: Open the following URL to continue
:: mairu :: >   http://127.0.0.1:62592/auth/xxxxxxxxxxxxxx
:: mairu ::
```

The issue requests behavior similar to `aws sso login` in aws-cli, where the
authentication URL is opened automatically in the user's default browser. The
issue suggests the [`webbrowser`](https://crates.io/crates/webbrowser) crate as
the Rust equivalent of CPython's `webbrowser` module used by aws-cli.

## Goals

- Automatically open the authentication URL in the default browser during
  `mairu login` (and `mairu exec`-triggered login).
- Keep the printed URL as a reliable fallback for headless / SSH / CI use.
- Provide a way to disable the auto-open behavior.

## Non-Goals

- Changing the OAuth flows themselves (Authorization Code / Device Code).
- Changing the agent process or gRPC protocol. The browser is opened entirely
  on the client side (`mairu login` / `mairu exec` run client-side and already
  print the URL via `crate::terminal::send`).

## Decisions (confirmed)

| Topic | Decision |
| --- | --- |
| Trigger mode | Auto-open by default (opt-out), matching `aws sso login`. |
| Disable mechanism | `--no-browser` flag **and** `MAIRU_NO_BROWSER` env var. The env var is read manually via `std::env::var_os(...).is_some()` (presence-based, mirroring the existing `MAIRU_NO_AUTO_AGENT`) rather than via clap `env =`, because a clap `bool` flag with `env =` only accepts `true`/`false` and aborts the command on `MAIRU_NO_BROWSER=1`. |
| Scope | Both Authorization Code and Device Code flows. |
| Crate | `webbrowser` 1.2.1 (latest stable as of 2026-04-16). |
| Headless safety | Skip auto-open when output is not a terminal (`is_terminal()` is false). |

## Architecture

`mairu login` runs client-side, connects to the agent over gRPC, and the two
flow handlers in `src/cmd/login.rs` print the authentication URL:

- `do_oauth_code` — prints `short_authorize_url` (the URL from the issue).
- `do_oauth_device_code` — prints `verification_uri_complete` (falls back to
  `verification_uri`).

We add a small, focused client-side module that opens the URL, and wire it into
both handlers **after** the URL is printed.

### Component: `src/browser.rs` (new module)

Two clearly-separated responsibilities so the decision logic is unit-testable
without ever launching a real browser:

```rust
/// Pure decision logic. Unit-tested.
pub fn should_open(no_browser: bool, is_terminal: bool) -> bool {
    !no_browser && is_terminal
}

/// Best-effort browser launch. Not unit-tested (would spawn a browser).
/// Never returns an error to the caller: a failure to open the browser must
/// not fail the login, because the URL is already printed as a fallback.
pub async fn open_url(url: &str) {
    // webbrowser::open_browser_with_options is synchronous and may spawn a
    // subprocess, so run it on a blocking thread.
    // Use BrowserOptions::with_suppress_output(true) to avoid noisy output
    // from launchers such as xdg-open.
    // On failure, tracing::debug!/warn! and return.
}
```

`open_url` takes an owned `String` (or `&str` cloned) into `spawn_blocking`.

### Component: flag/env plumbing

Add an identical pure CLI flag to both `LoginArgs` (`src/cmd/login.rs`) and
`ExecArgs` (`src/cmd/exec.rs`):

```rust
/// Do not automatically open the authentication URL in a browser.
/// Can also be requested by setting the MAIRU_NO_BROWSER environment variable.
#[arg(long, default_value_t = false)]
pub no_browser: bool,
```

The `MAIRU_NO_BROWSER` env var is **not** wired via clap's `env =`. A clap
`bool` flag with `env =` only accepts the literal `true`/`false` from the
environment and aborts the command on any other value (so `MAIRU_NO_BROWSER=1`
would error). Instead the env var is read by presence at the call site in
`cmd::login::login()`, mirroring the existing `MAIRU_NO_AUTO_AGENT`
(`src/cmd/agent.rs`) — setting the variable to any value (even empty) disables
auto-open:

```rust
let no_browser = args.no_browser || std::env::var_os("MAIRU_NO_BROWSER").is_some();
```

The effective value is resolved once and passed to both flow handlers. Because
`mairu exec`'s login path also goes through `cmd::login::login()`, the env var
is honored for both entry points (and `src/cmd/exec.rs::login()` still sets
`no_browser: args.no_browser` so the CLI flag propagates from `mairu exec`).

### Wiring into the flows

`login()` already holds `args: &LoginArgs`. Thread `args.no_browser` into both
handlers:

- `do_oauth_code(agent, server, no_browser: bool)`
- `do_oauth_device_code(agent, server, no_browser: bool)`

In each handler:

1. Print the existing URL message (unchanged) — the URL is **always** shown.
2. Compute `crate::browser::should_open(no_browser, crate::terminal::is_terminal().await)`.
3. If true:
   - Print one extra line, e.g.
     `:: mairu :: Attempting to open the URL in your browser...`
   - `crate::browser::open_url(<url>).await`.

For `do_oauth_code` the URL is `short_authorize_url`. For
`do_oauth_device_code` it is `authorize_url` (= `verification_uri_complete`,
falling back to `verification_uri`), which is the value already selected for
printing.

## Data flow

```
mairu login / mairu exec
  -> login(args)                       // client-side
     -> do_oauth_code / do_oauth_device_code(..., no_browser)
        -> terminal::send(url message)            // always
        -> if should_open(no_browser, is_terminal):
             terminal::send("Attempting to open ...")
             browser::open_url(url)               // best-effort, spawn_blocking
        -> listen_for_callback / poll completion  // unchanged
```

## Error handling

- A browser-open failure (no `DISPLAY`, no browser, launcher error) is logged at
  `debug`/`warn` and otherwise ignored. The login proceeds using the
  already-printed URL.
- Headless/CI: when `/dev/tty` cannot be opened, `terminal::is_terminal()`
  returns false and `should_open` returns false, so no launch is attempted.

## Testing

- Unit test `browser::should_open` for all combinations of `no_browser` and
  `is_terminal`:
  - `(false, true)  -> true`
  - `(true,  true)  -> false`
  - `(false, false) -> false`
  - `(true,  false) -> false`
- `open_url` is intentionally not unit-tested (it would launch a real browser);
  it is kept thin so the only logic worth testing lives in `should_open`.
- Manual verification: `mairu login <server>` opens the browser; with
  `--no-browser` or `MAIRU_NO_BROWSER=1` it does not; over SSH without a display
  it falls back to the printed URL without error.

## Documentation

- `README.md`: document the auto-open behavior and how to disable it
  (`--no-browser` / `MAIRU_NO_BROWSER`).

## Definition of done

- `cargo build`, `cargo test`, `cargo clippy`, and `cargo fmt --check` all pass.
```
