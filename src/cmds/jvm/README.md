# JVM ecosystem filters

Filters for JVM-based build tools.

| Module           | Tool(s)                              | Modes                                                                                  |
|------------------|--------------------------------------|----------------------------------------------------------------------------------------|
| `gradlew_cmd.rs` | `./gradlew`, `gradlew.bat`, `gradle` | Build / Test / ConnectedTest / Lint / Dependencies — streaming line filter + passthrough |
| `mvn_cmd.rs`     | `mvn`, `./mvnw`, `mvnw.cmd`          | Test / Compile / Package / Passthrough — buffered single-pass filter per phase           |

## Maven (`mvn_cmd.rs`)

Phase routing (`detect_phase`):

| Phase       | Goals                                                  | Filter                  |
|-------------|--------------------------------------------------------|-------------------------|
| `Test`      | `test`, `integration-test` (Failsafe = Surefire shape) | `filter_surefire`       |
| `Compile`   | `compile`, `test-compile`                              | `filter_compile`        |
| `Package`   | `package`, `install`, `verify`, `deploy`               | `filter_package`        |
| `Passthrough` | `clean`, `site`, `dependency:*`, `--version`, `--help`, empty, any unrecognised goal | none |

Key behaviours:

- **ANSI strip first** in every filter — real Maven output contains colour escapes.
- **English-footer guard** — if neither `BUILD SUCCESS` nor `BUILD FAILURE` appears as a trimmed line suffix, return the ANSI-stripped raw input unchanged. Protects non-English locales.
- **Verbose bypass** — `-X`, `--debug`, `-e`, `--errors` skip filtering (`run_passthrough`). User asked for detail; respect it.
- **Surefire block collapse** — Surefire emits `[INFO] Running <FQN>` … `[INFO] Tests run: N, Failures: F, Errors: E, …, Time elapsed: T s - in <FQN>`. The filter buffers each block and emits it only when `F > 0` or `E > 0`. Passing blocks (the bulk of healthy-project output) are dropped silently. Failing blocks are emitted with framework stack frames stripped via a deny-list (`at org.junit.`, `at java.util.`, `at sun.reflect.`, etc.).
- **Duration normalisation** — `Time elapsed: 2.341 s` → `Time elapsed: T s` and `[INFO] Total time: 49.550 s` → `[INFO] Total time: T s` for deterministic test output.
- **Wrapper detection** — `./mvnw` (POSIX) and `mvnw.cmd` (Windows) detected via string-literal `Command::new` (semgrep-safe); falls back to `resolved_command("mvn")`.
- **Reactor Summary preservation** — for multi-module builds, the trailing `Reactor Summary for <root>` block with per-module SUCCESS/FAILURE rows is kept (toggled by a `[INFO] Reactor Summary for ` header and cleared on `BUILD SUCCESS` / `BUILD FAILURE`).
- **Failure cap** — both the count of emitted failing test classes and the size of the `[ERROR] Failures:` summary block are bounded by `limits.mvn_max_failures` (default `25`). Excess emissions are replaced by a single `... +N more failing test classes` / `... +N more failures` tail to keep large failure sets compact. Set the limit to `0` in `config.toml` to opt out of the cap entirely.

Token-savings tests run inline as part of `cargo test --all` and verify ≥90% savings for `mvn test` and ≥85% for `mvn install` on full synthetic fixtures (gzipped, ~1100 lines each). The `flate2` dependency (already in `Cargo.toml`) decompresses the ~3 KB gzipped fixtures in milliseconds.

### Integrity-check whitelist

`Commands::Mvn` is intentionally omitted from `is_operational_command` in `src/main.rs`, matching the gradle precedent (`Commands::Gradlew` also omitted). The whitelist guards SHA-256 hook-integrity verification; filter modules invoked through an already-verified hook do not need a second check on their own dispatch path. Per the comment above the function, the whitelist is opt-in by design and a forgotten command fails open rather than creating false confidence about what's protected.

## Gradle (`gradlew_cmd.rs`)

See module docs and the gradle PR (`feat/gradlew-android-support`) for rationale. Streaming filter chosen because Gradle output is task-line-based, not block-based — unlike Maven Surefire.
