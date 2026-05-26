# Input Surface Testing

When an issue adds or changes a user-facing input surface, test the **grammar** the implementation introduced, not only the downstream behavior. Happy-path tests that exercise the business logic but never touch the parser/validator/schema will silently miss bugs in the parsing layer itself.

## The principle

> When implementation changes the shape of user input, tests must exercise the shape itself. Happy-path behavior is not enough.

The trap is subtle: the issue's acceptance criterion can be fine ("supports multiple review states"), the business logic can be fine (the aggregation function genuinely handles a list), and the tests can be fine *for what they test* — and the feature can still be broken at the parser, because the test invoked the function directly instead of going through the CLI/HTTP/schema layer that the user actually uses.

This is implementation-time discipline. The issue author can't predict every grammar choice the implementer will make; the implementer is the only one who knows which shape was actually picked.

## When this applies

This file applies to an issue if **any** of the following are true:

- The issue adds a new CLI flag, subcommand, positional, or argument schema.
- The issue changes the accepted form of an existing CLI argument (a scalar becomes a list, a positional becomes a flag, a string becomes an enum).
- The issue adds or changes an HTTP endpoint's query params, request body shape, or content negotiation.
- The issue adds or changes a function/tool schema exposed to callers (LLM tool definitions, RPC signatures, library entry points).
- The issue adds or changes a config-file key (TOML/YAML/JSON), env-var, or precedence rule.
- The issue adds or changes a structured input file format or user-authored manifest.

If none of those apply, this file isn't relevant — the issue's tests can stay on the business-logic side. The most common cases this file is for are CLI work and HTTP API work.

## What counts as an input surface

A non-exhaustive list:

- CLI flags, subcommands, and positionals (`clap`, `cobra`, `argparse`, `click`, etc.).
- HTTP query parameters, request bodies, headers, and content-type negotiation.
- Function or tool schemas — LLM tool definitions, RPC method signatures, public library APIs.
- Config files (TOML/YAML/JSON/INI) — including the precedence rules between config and other input.
- Environment variables — including the precedence rules between env and CLI/config.
- Structured input files the program reads (CSV, JSONL, fixture files).
- User-authored manifests (Dockerfiles, `package.json`, `pyproject.toml`-style configuration).

The unifying property: anyone outside the program supplies the bytes, the program parses/validates them, and the program acts on the parsed result.

## The checklist

For each changed input surface, exercise at least one test for each of the following grammar dimensions that applies:

1. **Accepted value forms.** Each shape the surface accepts gets at least one positive test that goes through the real parser. For a CLI flag, that means invoking the binary; for an HTTP endpoint, that means an actual request; for a tool schema, that means a call that goes through the schema validator.

2. **Missing or empty values.** What happens with `--flag` and no value? `--flag=`? An empty string in JSON? An absent key vs. a null value? Each of these can mean different things, and the parser's behavior should be intentional, not accidental.

3. **Repeated inputs.** If the surface accepts multiple values via repetition (`--foo a --foo b`), test it. If it *doesn't* accept repetition, test that the user gets a clear error rather than silent last-wins.

4. **Mutually exclusive inputs.** If two flags can't both be set, the parser should reject the combination — not silently use one. Test both directions (A then B, B then A) since precedence-based "exclusion" is often actually last-wins-in-disguise.

5. **Precedence / default behavior.** When the same logical setting can come from multiple sources (flag > env > config > default), test that precedence works. Set it via env, override with flag, confirm the flag wins. Then drop the flag, confirm env wins. Etc.

6. **Invalid enum values.** If a flag/field accepts a fixed set of values, an unsupported value should produce an actionable diagnostic, not a generic failure. Test with a string that's plausibly close to a valid value ("ready" vs valid "open") to confirm the error names the value and lists the valid set.

7. **Unsafe-mutation guards.** If the surface controls a destructive operation (overwrite, delete, force-push), the dangerous path should require an explicit opt-in. Test that the guard fires when the opt-in is absent.

8. **Output/error channel contract.** If the command emits structured output (JSON to stdout, say), test that diagnostics and progress go to stderr — not into the JSON stream. A consumer parsing stdout shouldn't see a warning line corrupting their parse.

9. **Ordering / positional placement.** Where a flag can appear in different positions (before or after a subcommand, before or after positionals), confirm both forms work or both don't.

10. **Attached vs separated values.** `--foo=bar` and `--foo bar` are usually meant to be equivalent. Confirm they are. `--foo=` (attached empty) and `--foo` (missing value) are usually meant to be different — confirm the parser distinguishes them.

11. **Short vs long flag equivalence.** If both `-f` and `--foo` exist, exercise at least one test through each. Often the parser routes them separately and only one path was implemented.

12. **Boolean negation.** If `--foo` is a boolean, is there a `--no-foo`? Should there be? If yes, test it; if no, decide consciously.

Not every dimension applies to every surface. The point of the list is to make the question "did I think about this?" explicit, not to demand 12 tests for every flag.

## A worked example

The CLI `repo-review aggregate` takes a `--review-state` flag and the issue's acceptance criterion says "can read multiple review states."

The naive test path:

```python
def test_aggregate_handles_list_of_states():
    result = aggregate_states(["open", "ready"])
    assert result == {...}
```

This passes. It also misses the real bug.

The real bug: the CLI parser stores `--review-state` as a scalar. Invoking:

```bash
repo-review aggregate --review-state open --review-state ready --json
```

…overwrites `open` with `ready`. The aggregation function correctly receives `["ready"]` (one element) and produces output that looks fine to a casual reader, but it isn't aggregating two states; it's aggregating one.

The fix is to test through the real parser:

```python
def test_aggregate_cli_accepts_repeated_review_state():
    result = run_cli([
        "aggregate",
        "--review-state", "open",
        "--review-state", "ready",
        "--json",
    ])
    parsed = json.loads(result.stdout)
    assert set(parsed["states_aggregated"]) == {"open", "ready"}
```

This test fails until the parser is changed to accumulate values. Going through `argparse.action='append'` (or the equivalent in `clap`, `cobra`, etc.) is the actual fix.

Same principle for the other dimensions on the checklist:

```python
def test_aggregate_rejects_invalid_review_state():
    result = run_cli(["aggregate", "--review-state", "redy"], expect_failure=True)
    assert "redy" in result.stderr      # the bad value is named
    assert "ready" in result.stderr      # the valid set is listed

def test_aggregate_json_output_is_parseable_with_warnings():
    result = run_cli(["aggregate", "--review-state", "open", "--json", "--verbose"])
    json.loads(result.stdout)            # must not raise; warnings went to stderr
```

Each test exercises a specific grammar guarantee.

## Where this hooks into the loop

This file applies *during* implementation — not at claim time (you don't know yet what shape the implementation will take) and not solely at close time (the tests need to exist before the close).

In practice:

1. **As you implement**, when you reach for a new flag/param/key, pause and ask: "what grammar am I introducing here?" Run through the relevant items on the checklist above. Write the grammar tests *as part of* implementing the feature.

2. **At close time** (see [closing-and-sync.md](closing-and-sync.md) step 1, acceptance verification), one of the questions to ask yourself is:

   > Did tests cover the input grammar introduced by this issue, not just the happy-path command behavior?

   If the answer is no and the issue changed an input surface, you're not done. Either write the tests or, if the missing tests reveal a deeper design issue, route the surprise through [discovery.md](discovery.md).

3. **In the close `--reason`**, if grammar-level work was a meaningful part of the implementation, mention it. A reason like "Added aggregate command with repeated `--review-state` support and enum validation — commit a1b2c3d" tells a future agent more than "Implemented aggregate — commit a1b2c3d."

## When `plan-to-beads` can help

The planning side can carry some of this load, but not all of it. The issue author can call out input-grammar requirements *that are part of the spec* — "supports multiple review states," "rejects unknown states with an actionable error," "respects flag-over-env precedence." See `plan-to-beads/field-semantics.md` for the acceptance-criteria pattern.

What the issue author *can't* do is predict every grammar dimension the implementer might introduce. If the implementer chooses `--review-state` repetition over `--review-states=open,ready` (comma-separated), the dimensions that need testing differ between the two choices. That call belongs to the implementer; the grammar tests follow from the call.

The division of labor:

- **`plan-to-beads`** (acceptance criteria): name the grammar requirements that are part of the spec.
- **`beads-to-done`** (this file): for every grammar choice the implementer makes — whether spec'd or chosen at implementation time — test the choice through the real surface.

## Checklist (the short version)

When this issue adds or changes an input surface:

1. **Name the surface.** CLI flag? HTTP param? Config key? Schema field?
2. **List the new accepted forms.** What can users now type/send/configure?
3. **List the rejected forms.** What should fail, and how?
4. **Walk the dimensions above.** Mark which apply, write at least one test for each that does.
5. **Verify the output/error channel contract.** Diagnostics on stderr, structured output on stdout, exit codes as documented.
6. **Confirm at close time.** Acceptance verification asks: "did I test the grammar I introduced?"

## Pitfalls

| Anti-pattern | What it looks like | The fix |
|---|---|---|
| Testing only the business function | `def test_aggregate(): assert aggregate(["a","b"]) == ...` | Add a test that goes through the real parser/validator. |
| "It worked when I ran it once" | Manual `repo-review aggregate --flag a --flag b` worked at the terminal; no automated test. | Manual checks don't survive refactors. Codify each grammar dimension. |
| Trusting the framework | "argparse handles repeated flags by default" — but the action was `'store'`, not `'append'`. | The framework's defaults are not your defaults; test what your config actually does. |
| Skipping channels | Tests assert `result.stdout` parses as JSON, but never check that the warnings landed on stderr. | A `--json` consumer doesn't care if your code "logs" the warning — they care whether their parser breaks. Check both streams. |
| Missing the precedence ladder | Tests check that the flag works and that the env var works, but never that the flag overrides the env. | Precedence bugs only surface when both sources are set. Test the conflict explicitly. |
| Letting last-wins masquerade as exclusion | Two flags that should be mutually exclusive instead silently use whichever came last. | Test both orderings; assert the parser rejects rather than picks. |
