# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

neotest-scala is a Neovim plugin (Lua) that adapts [neotest](https://github.com/rcarriga/neotest) for Scala test frameworks. It supports three test frameworks (utest, munit, scalatest) and two runners (bloop, sbt), with optional DAP debugging via nvim-metals.

## Development

This is a pure Lua Neovim plugin with no build step or test suite. To test changes, load the plugin in Neovim and run Scala tests through neotest.

Format Lua files with [StyLua](https://github.com/JohnnyMorganz/StyLua) using the config in `stylua.toml` (spaces, Unix line endings, auto-prefer double quotes):
```
stylua lua/
```

## Architecture

All source code is in `lua/neotest-scala/` (3 files, ~760 lines total).

### init.lua — Adapter (neotest interface)
Implements the `neotest.Adapter` interface that neotest calls into:
- `root` — Detects Scala projects by `build.sbt` presence
- `is_test_file` — Matches `.scala` files containing "test", "spec", or "suite" in the filename
- `discover_positions` — Uses **tree-sitter queries** to find `object_definition`/`class_definition` (namespaces) and `call_expression` with `test(...)` (test cases)
- `build_spec` — Delegates to the configured framework to build a shell command, optionally attaches DAP strategy config
- `results` — Delegates to the framework to parse CLI output into pass/fail results

Configuration is handled via the `__call` metamethod: options (`args`, `runner`, `framework`, `bloop_project`) can each be a static value or a function returning a dynamic value.

### framework.lua — Framework implementations
Each framework (utest, munit, scalatest) is a factory function returning a `neotest-scala.Framework` table with:
- `build_command(runner, project, tree, name, extra_args)` — Constructs the bloop/sbt CLI command with the correct test path syntax for that framework
- `get_test_results(output_lines)` — Parses CLI output lines (after ANSI stripping) into a `{test_id = "passed"|"failed"}` map
- `match_func` (scalatest only) — Custom matching for position IDs since scalatest output format differs

**Test path construction** varies by position type (test, namespace, file, dir) and by framework. Each framework's `build_test_path` walks the neotest tree to construct fully-qualified test identifiers including the package name.

**Output parsing** differs per framework:
- utest: Lines starting with `+` (pass) or `X` (fail)
- munit: Lines starting with `+` (pass) or `==> X` (fail), grouped under namespace headers
- scalatest: Lines starting with `- ` followed by ` *** FAILED ***` (fail) or not (pass), grouped under namespace headers

### utils.lua — Shared helpers
- `get_position_name` — Strips quotes from tree-sitter captured test names
- `get_package_name` — Reads the first line of a Scala file to extract the `package` declaration

### Key design notes
- Project name resolution: first tries parsing `name := "..."` from `build.sbt`, falls back to `bloop projects` CLI for bloop runner
- Runner can also be read from `vim.g["test#scala#runner"]` for vim-test compatibility
- DAP strategy builds `nvim-metals` compatible launch configs; individual test debugging uses `ScalaTestSuitesDebugRequest`
- ScalaTest only supports FunSuite style currently
