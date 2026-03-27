# [Project Name] — CLI Tool

## Project Overview

<!-- Replace this block with your project description -->
[One paragraph describing what this CLI does, who uses it, and what problem it solves.]

## Tech Stack

- **Language:** [TypeScript (Node.js) / Rust / Go / Python]
- **Argument Parsing:** [Commander.js / clap / cobra / click]
- **Output Formatting:** [chalk+ora / colored / lipgloss / rich]
- **Config:** [cosmiconfig / XDG config dirs / TOML]
- **Testing:** [Vitest / cargo test / go test / pytest]
- **Distribution:** [npm / cargo / brew / pip / standalone binary]

## File Structure

```
src/
  cli.ts                  # Entry point: parse args, dispatch to commands
  commands/
    init.ts               # One file per command
    build.ts
    deploy.ts
    config/
      set.ts              # Subcommands get a directory
      get.ts
      list.ts
  lib/
    config.ts             # Read/write config from disk
    output.ts             # Print helpers: info, warn, error, success, table, json
    prompt.ts             # Interactive prompts (inquirer/prompts)
    http.ts               # HTTP client for API calls
    fs.ts                 # File system helpers with error handling
    errors.ts             # Error classes
  types.ts                # Shared types
  constants.ts            # Exit codes, default paths, version
tests/
  commands/               # Integration tests per command
    init.test.ts
    build.test.ts
  lib/                    # Unit tests for library code
    config.test.ts
  fixtures/               # Sample project directories for testing
    valid-project/
    broken-config/
```

**Rules:**
- One command per file. The file exports a function that receives parsed args and returns an exit code.
- Commands never call `process.exit()` directly. Return exit codes up to the entry point.
- All user-facing output goes through `lib/output.ts`. Commands never call `console.log` directly.
- Config, HTTP, and FS operations are in `lib/`. Commands orchestrate, libs execute.

## Code Style and Conventions

### General
- Use early returns. Validate inputs at the top of every command, bail immediately.
- Prefer synchronous code where possible. CLIs should feel instant.
- No classes unless the language idiom demands it (Rust/Go: use structs). Functions compose better.
- Every string shown to the user is a UX decision. Be concise, be helpful.

### Argument Design
```bash
# Good: predictable, Unix-like
myctl deploy --env production --force
myctl config set key value
myctl users list --format json --limit 50

# Bad: inconsistent, confusing
myctl --deploy --env=production -f
myctl setConfig key=value
myctl users --list --json -n 50
```

- Required arguments are positional. Optional modifiers are flags.
- Use `--long-flag` as primary, `-s` as shorthand for frequent flags.
- Boolean flags are `--force`, not `--force=true`. Use `--no-color` for negation.
- Every command has `--help`. Every flag has a description.
- Support `--json` flag on every command that produces output. Machine-readable by default.
- Support `--quiet` and `--verbose` globally. Default is human-friendly.

### Output Formatting
```
# Human output (default)
  Deploying to production... done (2.3s)
  URL: https://app.example.com

# JSON output (--json)
{"status":"deployed","url":"https://app.example.com","duration_ms":2300}

# Error output (always stderr)
Error: Config file not found at ~/.myctl/config.toml
  Run `myctl init` to create one.
```

- Use stdout for data, stderr for progress/errors. This lets piping work: `myctl list --json | jq .`
- Spinners and progress bars go to stderr.
- Colors are opt-out via `--no-color` or `NO_COLOR=1` env var. Detect TTY.
- Tables for list output. Aligned columns, truncate long values with ellipsis.
- Always print the next action on error. "Run `myctl init` to fix this."

### Exit Codes
```
0   Success
1   General error (catch-all)
2   Invalid usage / bad arguments
64  Config file missing or invalid
65  Authentication failure
66  Network / API error
130 User interrupted (Ctrl+C)
```

Define exit codes as named constants. Never use bare numbers.

## Error Handling

- Wrap all errors with context. "Failed to read config" is useless. "Failed to read config at ~/.myctl/config.toml: permission denied" is actionable.
- Separate user errors (wrong input) from system errors (disk full, network down).
- User errors: short message + suggestion. No stack trace.
- System errors in `--verbose` mode: full stack trace and debug info.
- Network errors: include the URL that failed, the HTTP status, and whether retries were attempted.

```typescript
// Good: actionable error
class ConfigNotFoundError extends CLIError {
  constructor(path: string) {
    super(
      `Config file not found at ${path}`,
      `Run \`myctl init\` to create a default config.`,
      ExitCode.CONFIG_ERROR
    );
  }
}

// Bad: generic error
throw new Error('config not found');
```

- Catch `SIGINT` (Ctrl+C) gracefully. Clean up temp files, print a newline, exit 130.
- Never leave the user's project in a broken half-state. Use temp directories and atomic renames.

## Config Management

- Config lives in `~/.config/myctl/config.toml` (XDG) or `~/.myctl/config.toml`.
- Project-local config: `.myctl.toml` or `myctl.config.ts` in project root.
- Precedence: CLI flags > env vars > project config > user config > defaults.
- Sensitive values (API tokens): store in OS keychain or env vars, not plain text config.
- `myctl config set/get/list` commands for managing config without editing files.
- Config file is human-editable. Use TOML or YAML, not JSON (no comments in JSON).

## Testing

### Integration Tests (Primary)
Test the full command by executing it as a subprocess or calling the command function directly.

```typescript
describe('myctl init', () => {
  it('creates config file in empty directory', async () => {
    const dir = await createTempDir();
    const result = await runCommand(['init'], { cwd: dir });

    expect(result.exitCode).toBe(0);
    expect(result.stdout).toContain('Created config at');
    expect(fs.existsSync(path.join(dir, '.myctl.toml'))).toBe(true);
  });

  it('refuses to overwrite existing config without --force', async () => {
    const dir = await createTempDir();
    await fs.writeFile(path.join(dir, '.myctl.toml'), 'existing');

    const result = await runCommand(['init'], { cwd: dir });

    expect(result.exitCode).toBe(1);
    expect(result.stderr).toContain('already exists');
    expect(result.stderr).toContain('--force');
  });

  it('outputs JSON when --json flag is passed', async () => {
    const dir = await createTempDir();
    const result = await runCommand(['init', '--json'], { cwd: dir });

    const output = JSON.parse(result.stdout);
    expect(output.config_path).toBeDefined();
  });
});
```

### Test Helpers
- `runCommand(args, opts)` — Runs the CLI and captures stdout, stderr, exit code.
- `createTempDir()` — Creates an isolated temp directory, auto-cleaned after test.
- `createFixture(name)` — Copies a fixture directory to a temp location.
- `mockServer(routes)` — Starts a local HTTP server for API-dependent commands.

### What to Test
- Every command with valid input (happy path).
- Every command with missing/invalid arguments (error messages and exit codes).
- Flag combinations: `--json`, `--verbose`, `--quiet`, `--no-color`.
- Config precedence: flag overrides env, env overrides file.
- Destructive commands: confirm they prompt (or require `--force`).
- Piping: stdout is clean (no spinners/colors) when not a TTY.

## Security

- Never store API keys in plain text config files. Use environment variables or OS keychain.
- Sanitize file paths. Prevent path traversal when operating on user-provided paths.
- Verify TLS certificates on all HTTPS requests. No `--insecure` by default.
- If downloading executables or plugins, verify checksums.
- Prompt before destructive operations. Require `--force` to skip.
- If the CLI manages credentials, support `myctl logout` to remove stored tokens.

## Performance Guidelines

- Startup time under 100ms. Lazy-import heavy dependencies.
- For Node.js CLIs: avoid importing the entire SDK at the top level. Dynamic `import()` in commands.
- Show a spinner for any operation over 500ms. Show a progress bar for operations over 5s.
- Parallelize independent operations (multiple file copies, batch API calls).
- Cache API responses locally where appropriate (e.g., user profile, plan info) with a TTL.
- Large file operations: stream, do not load into memory.

## Common Pitfalls

1. **Do not mix stdout and stderr.** Data goes to stdout, everything else to stderr. Piping depends on this.
2. **Do not hardcode paths.** Use `os.homedir()` and `path.join`. Respect XDG on Linux.
3. **Do not ignore Ctrl+C.** Register a SIGINT handler that cleans up and exits cleanly.
4. **Do not use interactive prompts unconditionally.** Detect TTY. Fail with a helpful message if not interactive and a required flag is missing.
5. **Do not ship without shell completions.** Most arg parsers generate them. Include install instructions.
6. **Do not print raw error objects.** Users should see a sentence, not `[Object object]` or a 50-line stack trace.
7. **Do not forget `--version`.** It should print just the version number and exit.
8. **Do not require global install.** Support `npx myctl` / `uvx myctl` for zero-install usage.

## Distribution

- Pin your binary/package version in lockfiles. Reproducible installs.
- Provide a standalone binary option for non-developers (pkg, go build, cargo build --release).
- Include man pages or at minimum comprehensive `--help` output per command.
- Ship shell completions for bash, zsh, fish. Generate from your arg parser.
- Changelog follows keep-a-changelog format. Users read this.
- Semantic versioning: breaking CLI changes (renamed flags, changed output format) are major bumps.

## Git and PR Workflow

### Commit Messages
```
feat(deploy): add --dry-run flag for deploy command
fix(config): handle missing XDG_CONFIG_HOME on macOS
perf(startup): lazy-load API client to reduce startup by 80ms
docs(readme): add shell completion install instructions
```
Format: `type(scope): lowercase imperative description`

### PR Checklist
- [ ] `--help` text updated for new/changed commands
- [ ] `--json` output works and is parseable
- [ ] Exit codes are correct (not just 0 and 1)
- [ ] Error messages include next-action suggestions
- [ ] Works without TTY (piping, CI environments)
- [ ] No interactive prompts without TTY detection
- [ ] Tests cover happy path, invalid args, and error cases
- [ ] Shell completions updated if commands/flags changed
- [ ] No hardcoded paths or platform assumptions
- [ ] CHANGELOG.md updated for user-facing changes
