# pmpx-plugin-bun

> 中文版见 [README_CN.md](README_CN.md)

The **bun** backend for [pmpx](https://crates.io/crates/pmpx). It maps pmpx's verbs onto bun
commands, and does nothing else: no file reads, no environment, no network.

```console
$ pmpx install react       # in a bun project → bun add react
$ pmpx build --outdir dist #                  → bun run build --outdir dist
$ pmpx exec vite           #                  → bunx vite
```

## The mapping

| pmpx | bun |
| ---- | --- |
| `install` | `bun install` |
| `install <pkg>...` | `bun add <pkg>...` |
| `remove <pkg>...` | `bun remove <pkg>...` |
| `run <script> [arg]...` | `bun run <script> [arg]...` |
| `build [arg]...` | `bun run build [arg]...` |
| `test [arg]...` | `bun run test [arg]...` |
| `update` | `bun update` |
| `update <pkg>...` | `bun update <pkg>...` |
| `exec [arg]...` | `bunx [arg]...` |

## `build` and `test` have to go through `run`

This is the one thing to know about this backend, and it is not a style choice.

**`bun build` and `bun test` are bun's own built-in commands** — a bundler and a test runner —
and they **shadow** the scripts of the same name in `package.json`. Dropping the `run` does not
run your script with slightly different arguments; your script is never reached at all.

Measured, with a `package.json` whose `build` and `test` scripts echo the arguments they were
given:

```console
$ bun build --foo
error: Missing entrypoints. What would you like to bundle?
                              # ↑ bun's bundler, not package.json's build script  ✗

$ bun run build --foo
SCRIPT GOT:--foo              # ↑ the script runs, and gets its arguments          ✓

$ bun test --foo
                              # ↑ bun's test runner runs; the script is never reached ✗

$ bun run test --foo
SCRIPT GOT:--foo              # ↑ the script runs                                     ✓
```

The `build` case at least fails loudly. The `test` case does not: `bun test` happily runs bun's
own test discovery and reports a result, so a project whose `test` script does something else
gets the wrong thing run with no warning. Both are therefore mapped to `bun run build` and
`bun run test`, always.

`install`, `add`, `remove`, `update` and `bunx` are unaffected — those are bun's own commands
doing exactly what the verb means, and none of them is shadowed by a script.

## No `--` is inserted

Unlike backends for tools that reject unknown options, bun forwards what it does not recognise
to the script, and it strips a `--` when one is present. Measured with a script that echoes its
arguments:

```console
$ bun run probe --foo      → GOT:--foo
$ bun run probe -- --foo   → GOT:--foo
```

Both work, so this backend inserts nothing: what pmpx passes through is what bun sees.

## Install

```console
$ pmpx plugin add bun
```

## Detection

From `pmpx-plugin.toml`, which travels with this crate:

| File | Weight | What it proves |
| ---- | ------ | -------------- |
| `bun.lock` | strong (100) | the project was actually resolved by bun (current, text format) |
| `bun.lockb` | strong (100) | the same, in the older binary format |
| `package.json` | weak (10) | the ecosystem, not the tool |

Both lockfile spellings are listed because bun renamed its lockfile: `bun.lockb` is what a
project that has not been migrated still carries, and it is just as much proof as `bun.lock`.
`package.json` is weak because in this family every Node package manager claims it.

## Requirements

Rust **1.82+**, which is the contract crate's floor.

## Repository

<https://github.com/pmpx-rs/pmpx-plugin-bun>

## License

MIT — see [LICENSE](LICENSE).
