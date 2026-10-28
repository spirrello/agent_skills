---
name: code-review
description: Review a GitLab merge request for issues in GitLab CI/CD, OpenTofu/Terraform, C#, Node.js, and Rust. Reports findings in chat; can optionally post them as a comment on the MR.
argument-hint: "[MR URL | group/project!IID | IID] [post]"
allowed-tools:
  - read
  - grep
  - glob
  - exec
permissions:
  ask:
    - Exec(glab mr note)
  deny:
    - Exec(glab mr merge)
    - Exec(glab mr approve)
    - Exec(glab mr rebase)
    - Exec(glab mr create)
    - Exec(glab mr update)
    - Exec(glab mr close)
    - Exec(glab mr reopen)
    - Exec(glab mr delete)
    - Exec(glab mr checkout)
    - Exec(glab mr subscribe)
    - Exec(glab mr todo)
---

# GitLab Merge Request Review

Perform a thorough review of a GitLab merge request. By default the review is
read-only: report findings in the conversation and **never** write to GitLab.
Only when the user explicitly asks to post (see modes below) may you create
comments on the MR — and even then only via `glab mr note`; never approve,
merge, update, close, or otherwise mutate the MR, and no `glab api` calls with
POST/PUT/DELETE/PATCH.

## 1. Authenticate

Before the first `glab` call, load the token without echoing it:

```bash
export GITLAB_TOKEN="$(cat ~/.gitlab_configuration/ai)"
export GITLAB_HOST="FILLOUT"
```

Do not print, log, or write the token anywhere.

## 2. Parse arguments

`$ARGUMENTS` is free text. Extract two things from it:

- **MR target** — the first token matching one of:
  - **Full MR URL** `https://<host>/<group>/<project>/-/merge_requests/<iid>` —
    this is a *remote* MR link: it works from any directory, no checkout needed.
    Extract project path and IID; pass `-R <group>/<project>` on all `glab mr`
    calls.
  - **`<group>/<project>!<iid>`** → same, with `-R <group>/<project>`.
  - **Bare IID** → `glab mr view <iid>` inside the current repo.
- **Output mode** — the keyword `post` (or `comment`) anywhere in the input
  means post the review on the MR after reporting in chat. No keyword →
  chat-only, read-only review.

If no MR target is given, run `glab mr view` inside the current repo (it
resolves the MR for the checked-out branch). If that fails, try
`glab mr list --source-branch "$(git branch --show-current)"`. If still no MR,
ask the user for an MR URL or IID — do not guess.

## 3. Gather context

```bash
glab mr view <iid> -R <repo> -F json          # title, description, sha, pipeline, labels
glab mr view <iid> -R <repo> --comments       # existing discussion — do not re-raise resolved points
glab mr diff <iid> -R <repo> --raw --color=never
```

- Check the pipeline status in the MR metadata; report failing pipelines.
- If the diff is too large to fetch at once, list changed files via
  `glab api "projects/<url-encoded-project>/merge_requests/<iid>/diffs?per_page=100"`
  and review file-by-file.
- When surrounding context is needed, fetch the file at the MR head sha:
  `glab api "projects/<url-encoded-project>/repository/files/<url-encoded-path>/raw?ref=<sha>"`.
  If the project is checked out locally, read local files instead.
- Skip noise: generated code, vendored files, `.lock`/lockfile churn (but do
  scan lockfiles for unexpected dependency additions). Don't review whitespace-only diffs.

## 4. Apply the checklists that match the changed files

| Changed files | Checklist |
|---|---|
| `.gitlab-ci.yml`, `ci/**/*.yml`, `.gitlab/` | GitLab CI/CD |
| `*.tf`, `*.tfvars`, `terragrunt.hcl`, `.terraform.lock.hcl` | OpenTofu |
| `*.cs`, `*.csproj`, `*.sln` | C# |
| `*.ts`, `*.tsx`, `*.js`, `*.jsx`, `package.json` | Node.js |
| `*.rs`, `Cargo.toml`, `Cargo.lock` | Rust |

If no checklist applies, fall back to a general review (correctness, security,
error handling, tests). Multiple checklists can apply to one MR.

### GitLab CI/CD

- Mutable image or include refs: `:latest` tags, `include:ref` pointing at a
  branch instead of a tag/sha, `include:remote` from untrusted sources.
- Secrets: tokens/passwords in `variables:` or `script:` output; secrets as
  non-masked CI/CD variables; protected variables consumed by jobs that also
  run on unprotected branches.
- `rules:`/`only:`/`except:` logic: jobs accidentally running on the default
  branch or missing entirely from MR pipelines; `changes:` filters that silently
  skip jobs; `when: manual` gating something that must be automatic.
- `allow_failure: true` on tests, lint, or security scanning jobs.
- Artifacts/dependencies: `artifacts:paths` overly broad (e.g. `.`), missing
  `expire_in`, `dependencies:`/`needs:` mismatch that ships stale artifacts,
  `needs:artifacts:false` confusion.
- Cache: per-branch cache poisoning (`cache:key` not scoped), `cache:when`
  pushing broken caches on failure.
- Shell injection: unquoted CI variables in `script:`, `eval`, `curl | bash`,
  un-pinned tool installs (`npm i -g foo`, `apt install` without versions).
- `environment:` deploys without protection rules; `interruptible`, `timeout`,
  `retry` missing or misconfigured; runner `tags:` that may not exist.
- `extends`/`!reference`/YAML anchors resolving to something different than intended.

### OpenTofu / Terraform (we use OpenTofu — `tofu`, not `terraform`)

- Destructive plan output: `destroy`, forces replacement, or `tainted`
  resources the description doesn't explain; missing `lifecycle` blocks
  (`prevent_destroy`) on stateful resources.
- Secrets: hardcoded credentials in `.tf`/`.tfvars`; `sensitive = false` on
  outputs that carry secrets; tfvars with secrets committed.
- State: backend changes; missing encryption/locking; `terraform_remote_state`
  reaching across trust boundaries.
- IAM/security: `Action: "*"`, `Resource: "*"`, `0.0.0.0/0` ingress, public
  buckets/ACLs, `publicly_accessible = true`, unencrypted storage.
- Versioning: unpinned providers/modules; module `ref` pointing at a branch
  instead of a tag/sha; `.terraform.lock.hcl` removed or regenerated
  unexpectedly.
- Correctness traps: `count`/`for_each` changes that re-key and recreate
  resources (was a `moved` block needed?); `data` sources that drift
  (`most_recent = true`); defaults that leak env-specific config into other
  environments; missing variable `validation` blocks.
- Outputs: new outputs exposing values that should stay internal.

### C#

- Async: missing `await` (fire-and-forget), `async void`, blocking on async
  (`.Result`, `.Wait()`, `GetAwaiter().GetResult()`) → deadlocks.
- Resources: `IDisposable`/`IAsyncDisposable` not disposed; `HttpClient`
  created per request instead of `IHttpClientFactory`.
- Null safety: `!` null-forgiving used to silence warnings rather than fix
  them; missing `ArgumentNullException.ThrowIfNull` on public APIs.
- EF Core / data: N+1 queries (lazy loading in loops), missing `AsNoTracking`
  on read paths, filters applied after materialization (`.ToList()` then
  `Where`), string-interpolated `FromSqlRaw` → SQL injection.
- Exceptions: `catch (Exception)` swallowing errors, `throw ex;` losing the
  stack trace, empty `catch` blocks.
- DI: scoped services captured by singletons; service-locator patterns.
- Dates/money: `DateTime.Now` instead of `UtcNow`; `double`/`float` for money.
- Security: `Newtonsoft` `TypeNameHandling` other than `None`, weak crypto
  (MD5/SHA1/`System.Random` for security), path traversal, model changes
  without a matching EF migration.

### Node.js / TypeScript

- Type safety: new `any`, `@ts-ignore`/`@ts-expect-error` without explanation,
  unsafe `as` casts on external data (HTTP bodies, env vars, DB rows).
- Boundaries: no validation of request input (zod/io-ts or similar);
  `process.env` read deep in the codebase instead of a config module.
- Async: floating promises (missing `await`/`void`), unhandled rejections,
  `try/catch` that can't catch because the promise was never awaited.
- Injection: template strings in raw SQL; `child_process.exec` with
  interpolation (prefer `execFile`/args array); `eval`/`new Function`.
- Web: new routes without auth middleware; `res.send`/rendered templates
  passing unsanitized user data (XSS); CORS `*`; unbounded `express.json()`.
- FS/path: `path.join`/`fs` calls on user-controlled paths without traversal
  checks.
- Errors: non-`Error` throws, swallowed `catch`, `err` re-typed from `unknown`
  without a guard.
- Deps: new dependencies without lockfile updates, heavy deps for trivial
  tasks; `==`/`!=` where `===` is meant; mutation of shared module-level state.

### Rust

- `unsafe`: any new `unsafe` block needs a justification comment and a
  soundness review; `mem::transmute`, `mem::forget`, raw pointer arithmetic.
- Panics: `unwrap()`/`expect()`/`panic!`/`unreachable!`/`todo!` in code paths
  reachable with bad input — prefer `?`/`Result`; `lock().unwrap()` ignoring
  poisoning.
- Conversions: `as` casts that can truncate or wrap — prefer `TryFrom`/
  `try_into()`; integer over/underflow on untrusted lengths or indexes.
- Async: `std::sync::Mutex`/`RefCell` guards held across `.await` (deadlock /
  cancellation hazards) — should be `tokio::sync::Mutex` or dropped first;
  `block_on` inside async; `spawn`ed tasks whose `JoinHandle` is dropped
  (detached tasks that can panic silently).
- API design: `String`/`Vec` params that should be `&str`/`&[T]`; needless
  `.clone()` in hot paths; `iter().map(...).collect()` used for side effects
  instead of a `for` loop.
- Errors: `Box<dyn Error>`/`anyhow` in library code that should use `thiserror`
  types; `map_err`/`unwrap_or` that silently discards the real error.
- Cargo: wildcard/`path`/`git` dependencies instead of versions; new
  `#[allow(...)]`/clippy suppressions hiding real warnings; `'static`
  lifetimes/`Box::leak` on data that isn't actually static.

## 5. Report

Rank by impact. Correctness and security findings outrank style nits — do not
pad the report with trivia. Skip anything already resolved in the MR
discussion.

```markdown
## Verdict: approve | approve-with-comments | request-changes

### Critical — must fix before merge
- **file.tf:42** — finding — suggested fix

### Major
- **src/foo.rs:120** — finding — suggested fix

### Minor / nits
- **file.cs:8** — finding

### Summary
2–4 sentences: what the MR does, overall quality, biggest risk, whether the
pipeline status supports merging.
```

Reference exact file paths and line numbers from the diff. If the MR is clean,
say so plainly — don't invent findings.

### Posting to the MR (only when asked)

Always show the full report in chat first. If the user passed `post`/`comment`,
then post a condensed version (verdict + Critical/Major findings + summary) as
a single MR comment:

```bash
glab mr note create <iid> -R <repo> --unique -m "$(cat <<'EOF'
<condensed markdown report>
EOF
)"
```

`--unique` avoids duplicate posts when the skill is re-run on the same MR.
Only create inline diff comments (`glab mr note create --file <path> --line
<n>`) if the user explicitly asks for inline comments.
