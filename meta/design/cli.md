# CLI Refactor: Replace `parseArgs` with `cac`

## Summary

Replace Node.js's built-in `parseArgs` with [cac](https://github.com/cacjs/cac) (v6.7.14) for CLI argument parsing. This fixes [#8410](https://github.com/rolldown/rolldown/issues/8410) (camelCase options silently ignored) and [#3248](https://github.com/rolldown/rolldown/issues/3248) (`-s inline` position restriction). cac is the same library used by Vite and tsdown.

## Motivation

The current implementation (`packages/rolldown/src/cli/arguments/index.ts`) has 16 hand-rolled workarounds on top of `parseArgs`. The root cause of #8410 is that `parseArgs` treats `--moduleTypes` as an unknown boolean (since it only knows `--module-types`), silently dropping the value into positionals. cac handles camelCase/kebab-case interchangeably, fixing this class of bugs.

## Current Implementation

The CLI entry point is `bin/cli.mjs` which imports `dist/cli.mjs`. The source is at `src/cli/index.ts`.

### Pipeline

```
bin/cli.mjs
  → src/cli/index.ts (entry)
    → checkNodeVersion()
    → parseCliArguments()                    ← THIS IS WHAT WE'RE REPLACING
      → arguments/index.ts
        → getCliSchemaInfo()                 — flatten valibot schema into { key: { type, description } }
        → build `options` object             — for parseArgs registration AND help.ts consumption
        → parseArgs({ options, strict: false, tokens: true, allowPositionals: true })
        → iterate tokens:
          → kebab→camelCase conversion
          → --no-* prefix stripping
          → type-based handling (boolean/string/object/array/union)
          → requireValue check
          → collect invalid options
        → warn about invalid options
        → assemble rawArgs
      → arguments/normalize.ts
        → prototype pollution guard
        → dot-notation unflattening (a.b → { a: { b } })
        → validateCliOptions() via valibot
        → split into input/output based on schema keys
        → merge positionals into input.input
    → process --environment (KEY:VALUE → process.env, separate from object parsing)
    → if --config: bundleWithConfig(configPath, cliOptions, rawArgs)
      → loadConfig() → resolve config (may be function receiving rawArgs)
      → merge cliOptions into config
      → bundleInner() or watchInner()
    → if input specified: bundleWithCliOptions(cliOptions)
    → if --version: print version
    → else: showHelp()
```

### Key files

| File                              | Role                                                                              |
| --------------------------------- | --------------------------------------------------------------------------------- |
| `cli/index.ts`                    | Entry point — orchestrates the pipeline                                           |
| `cli/arguments/index.ts`          | Core parsing — `parseArgs` + 16 hacks (~193 lines)                                |
| `cli/arguments/normalize.ts`      | Splits flat options into `input`/`output`, validates with valibot                 |
| `cli/arguments/alias.ts`          | Short flags, defaults, `reverse`, `requireValue` config                           |
| `cli/arguments/utils.ts`          | `setNestedProperty`, `camelCaseToKebabCase`, `kebabCaseToCamelCase`               |
| `cli/commands/help.ts`            | Custom help text generation (reads `options` export)                              |
| `cli/commands/bundle.ts`          | `bundleWithConfig`, `bundleWithCliOptions`, watch mode                            |
| `cli/logger.ts`                   | consola logger, replaced with plain `console.log` when `ROLLDOWN_TEST=1`          |
| `utils/validator.ts`              | valibot schemas for all CLI options, `getCliSchemaInfo()`, input/output key lists |
| `utils/flatten-valibot-schema.ts` | Recursively flattens valibot object schemas into `{ key: { type, description } }` |

### What `parseCliArguments()` returns

```ts
interface NormalizedCliOptions {
  input: InputOptions; // platform, external, moduleTypes, transform, checks, ...
  output: OutputOptions; // dir, file, format, sourcemap, minify, name, globals, ...
  help: boolean;
  config: string;
  version: boolean;
  watch: boolean;
  environment?: string | string[];
}

// Plus rawArgs: Record<string, any> — all parsed args including unknown ones
```

The migration only changes how we get from `process.argv` to this return type. Everything downstream (`cli/index.ts`, `commands/bundle.ts`, `commands/help.ts`) stays the same.

### Note: `--environment` is not an object option

`--environment` uses a different syntax from object options and is processed separately in `cli/index.ts` (not in the parser):

- Object options: `--module-types .png=dataurl,.svg=text` — uses `=` and `,`, parsed in `arguments/index.ts`
- `--environment`: `--environment PRODUCTION,FOO:bar,HOST:http://localhost:4000` — uses `:` and `,`, processed in `cli/index.ts` by writing to `process.env`

The `:` separator matches Rollup's `--environment` syntax. Schema type is `string | string[]`, not object. Unaffected by the cac migration.

## Strategy

Use cac with `run: true` and a default command with `allowUnknownOptions()`.

**What cac gives us for free:**

- camelCase/kebab-case conversion (fixes #8410)
- `--no-*` boolean negation with correct defaults
- `<required>` value validation via `checkOptionValue()` (replaces `requireValue`)
- `[optional]` value parsing — fixes `-s inline` position restriction (#3248)
- Dot-notation nesting via `setDotProp` (replaces manual unflattening)
- Short flag aliases and stacking (`-ms` = `--minify --sourcemap`)
- Array auto-accumulation for repeated flags

**What we still implement ourselves:**

- Object parsing (`--module-types .a=text,.b=json` — split on `,` then `=`). Supports both comma-separated single flag and repeated flags.
- Unknown option warning — `allowUnknownOptions()` suppresses cac's error, we add our own warning with the same message format.
- Prototype pollution guard — cac's `setDotProp` doesn't guard against `__proto__`, `constructor`, `prototype`.
- Input/output option splitting — rolldown-specific logic that splits flat CLI options into `InputOptions` and `OutputOptions` based on valibot schema keys.
- Custom help text — don't use `cli.help()`, keep our custom generator with sorting, padding, examples, notes.
- Duplicate option filtering — like Vite's `filterDuplicateOptions`, take last value for non-array types (`external` and `input` keep arrays, everything else takes last).
- rawArgs assembly — merge known + unknown options for config function passthrough.

**Key architectural decisions:**

- `run: true` — let cac handle validation (`checkOptionValue`, `checkUnknownOptions`).
- Default command with `allowUnknownOptions()` — suppresses unknown option errors so we can warn instead.
- Do NOT call `cli.help()` or `cli.version()` — these bypass our custom handling by calling `console.log` and setting `run = false` internally.
- Catch `CACError` from `checkOptionValue()` and reformat to match existing error messages.

**Net result:** ~130 lines removed, 10 of 16 hacks eliminated. The remaining 6 are rolldown-specific logic that any CLI library would need.

## Implementation Plan

### Step 1: Rewrite `arguments/index.ts`

This is the core change. Replace the entire `parseCliArguments()` function body.

**cac setup:**

```ts
import cac from 'cac';

const cli = cac('rolldown');
```

**Option registration** — loop over `schemaInfo` + `alias` and register each option:

```ts
for (const [key, info] of Object.entries(schemaInfo)) {
  if (info.type === 'never') continue;
  const config = alias[key as keyof CliOptions];

  // Build the rawName string for cac
  // e.g. "-d, --dir <dir>" or "-m, --minify" or "--no-treeshake"
  let rawName = '';
  if (config?.abbreviation) rawName += `-${config.abbreviation}, `;
  if (config?.reverse) {
    rawName += `--no-${camelCaseToKebabCase(key)}`;
  } else {
    rawName += `--${camelCaseToKebabCase(key)}`;
  }

  // Bracket syntax determines how cac handles the option:
  // - No brackets → boolean (registered in mri's boolean list)
  // - <required> → string, checkOptionValue throws CACError if value is missing
  // - [optional] → string, no error if value is missing (gets true)
  if (info.type !== 'boolean' && !config?.reverse) {
    if (config?.requireValue) {
      rawName += ` <${config?.hint ?? key}>`;
    } else {
      rawName += ` [${config?.hint ?? key}]`;
    }
  }

  cli.option(rawName, info.description ?? config?.description ?? '', {
    default: config?.default,
  });
}
```

**Default command with `allowUnknownOptions`:**

Register a default command (empty name) that accepts variadic positional args. Use `allowUnknownOptions()` so cac doesn't throw on unknown flags (we want to warn, not error). Use `run: true` so cac's built-in validation runs.

```ts
const cmd = cli.command('[...input]', '');
cmd.allowUnknownOptions();
cmd.action((input: string[], options: Record<string, any>) => {
  // `input` = positional args
  // `options` = all parsed options (known + unknown)
  // cac has already run checkOptionValue() at this point,
  // so <required> options are validated.
  // Post-process and return result.
});
cli.parse(process.argv);
```

With `run: true` + default command:

- `checkOptionValue()` runs automatically — `<required>` bracket syntax throws `CACError` when value is missing (e.g. `rolldown -d` with no dir). Catch `CACError` and format to our existing error message.
- `checkUnknownOptions()` runs but is suppressed by `allowUnknownOptions()` — unknown options silently pass through into the `options` object.
- The action callback receives positionals and options directly.

**Post-processing** inside the action callback:

1. **Unknown option warning**:
   - Compare parsed option keys against known schema keys
   - Warn on unrecognized keys (same message format as current)
   - Include unrecognized options in `rawArgs`

2. **Filter duplicate options** (like Vite's `filterDuplicateOptions`):
   - cac auto-accumulates repeated flags into arrays (e.g. `--format cjs --format esm` → `["cjs", "esm"]`)
   - For non-array schema types, take the last element
   - For array types (`external`, `input`), keep the array

3. **Object parsing** (`key=val,key=val`):
   - For schema type `'object'` (e.g. `moduleTypes`, `globals`, `transform.define`), split the string value on `,` then `=`
   - Two syntaxes: comma-separated in one flag (`--module-types .a=text,.b=json`) and repeated flags (`--module-types .a=text --module-types .b=json`). With cac, repeated flags auto-accumulate into arrays, so handle both string and array values.
   - ~20 lines, carried over from current implementation

4. **rawArgs assembly**:
   - Merge known + unknown options into `rawArgs` for config function passthrough

**Export the same `options` object** for `help.ts` — derive it from the schema + alias registration loop (same data, different shape than parseArgs format).

### Step 2: Simplify `arguments/normalize.ts`

**Remove the dot-notation unflattening loop** (lines 28-37). cac's `mri` method already calls `setDotProp` which splits on `.` and nests. The options object from cac already has `{ transform: { define: {...} } }` instead of `{ 'transform.define': '...' }`.

**Remove the prototype pollution guard** (lines 26-29). Move it to `arguments/index.ts` post-processing, applied before passing to normalize. (cac's `setDotProp` does not guard against `__proto__`.)

**Keep**: input/output splitting, valibot validation, positional handling.

### Step 3: Simplify `arguments/alias.ts`

**Remove `reverse` field** — cac handles `--no-*` natively via `--no-${key}` in the rawName string. The registration loop reads `reverse` to build the rawName, but after migration we can encode this directly in the rawName and drop the field.

**Remove `requireValue` field** — with `run: true`, cac's `checkOptionValue()` automatically enforces `<required>` bracket syntax and throws `CACError` when a required-value option has no value. We catch the error and format it to our existing message.

**Keep**: `abbreviation`, `default`, `hint`, `description`.

### Step 4: Simplify `arguments/utils.ts`

**Remove `kebabCaseToCamelCase`** — cac's `camelcaseOptionName` does this automatically.

**Keep**: `setNestedProperty` (used in `normalize.ts` for input/output splitting), `camelCaseToKebabCase` (used in `help.ts` and option registration).

### Step 5: Update `commands/help.ts`

The `options` export from `arguments/index.ts` changes shape. `help.ts` reads `options` to generate help text.

**Approach: Keep custom help generator.** Do NOT use `cli.help()` — it produces a different format and would break snapshots. Instead, build the `options` export during the registration loop so `help.ts` can consume it with minimal changes.

Do NOT call `cli.version()` or `cli.help()` — these intercept `process.exit()` and bypass our custom handling. Register `--help` and `--version` as plain boolean options.

### Step 6: Update `cli/index.ts`

Minimal changes. The `parseCliArguments()` return type stays the same. With `run: true`, cac may throw `CACError` for missing required values (e.g. `rolldown -d`). Wrap the `cli.parse()` call in a try/catch inside `parseCliArguments()` and convert `CACError` to the existing `logger.error` + `process.exit(1)` pattern.

## Breaking Changes

### Intentional (fixing bugs)

| Change                                      | Before                                                                                                    | After                                                                                                |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| camelCase options work                      | `--moduleTypes .png=dataurl` silently ignored                                                             | Works correctly                                                                                      |
| `-s` position restriction removed ([#3248]) | `-s inline` only works as last argument due to `parseArgs` `strict: false` not knowing `-s` takes a value | `-s inline` works in any position — mri knows `-s` takes an optional string via `[optional]` bracket |

[#3248]: https://github.com/rolldown/rolldown/issues/3248

### Behavioral differences (non-breaking but observable)

| Change                         | Before                                             | After                                                   | Impact                                                                                                                                                                                                                                                                                                                                     |
| ------------------------------ | -------------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Numeric string coercion        | `--code-splitting.min-size 1000` → string `"1000"` | → number `1000`                                         | mri coerces numeric-looking values because `getMriOptions` only populates `alias` and `boolean` lists, never `string`. mri's `toVal` checks `x * 0 === 0` and returns the number. Downstream valibot accepts both. **Low risk.** Could be fixed by populating `opts.string` in a custom mri options builder, but not worth the complexity. |
| `--no-*` on unknown options    | `--no-foo` → warns "foo is unrecognized"           | `--no-foo` → mri sets `foo: false` in parsed options    | `allowUnknownOptions()` suppresses cac's error. Our custom unknown-option check sees `foo` and warns. **Same behavior, but the value is `false` instead of being absent.** Low risk.                                                                                                                                                       |
| `--` delimiter behavior        | `parseArgs` with `strict: false` ignores `--`      | cac collects args after `--` into `options['--']` array | No current code reads `options['--']`. **No impact.**                                                                                                                                                                                                                                                                                      |
| `-s` (sourcemap) without value | Sets `sourcemap: true` via manual default          | cac `[optional]` with `default: true` → same result     | **No change.**                                                                                                                                                                                                                                                                                                                             |
| Short flag stacking            | `-ms` not supported (parseArgs limitation)         | `-ms` would set `minify: true, sourcemap: true`         | cac supports single-dash flag stacking. **New capability, not breaking.**                                                                                                                                                                                                                                                                  |

### Potential breaking changes to watch for

| Change                                   | Risk                                                                                    | Mitigation                                                                                                                           |
| ---------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `--no-preserve-entry-signatures` default | cac auto-sets default to `true` for `--no-*` options, but current default is `"strict"` | Pass explicit `{ default: 'strict' }` in option config. Verified: cac checks `this.config.default == null` so explicit default wins. |
| Error message format for missing values  | Current: `Option \`--dir\` requires a value but none was provided.`                     | cac throws `CACError`: `option \`-d, --dir <dir>\` value is missing`. Catch and reformat to match existing message.                  |
| Unknown option warning message format    | Current: `Option \`foo\` is unrecognized. We will ignore this option.`                  | `allowUnknownOptions()` suppresses cac's error. We implement our own warning with the same message format. **No change.**            |
| `=` syntax for values                    | `--format=cjs` works in both. `--customArg=customValue` (for rawArgs) works in both.    | mri splits on `=`. **No change.**                                                                                                    |

## Edge Cases

### `--sourcemap` dual behavior

`-s` alone → `true` (boolean). `-s inline` → `"inline"` (string). `--sourcemap hidden` → `"hidden"`.

**cac solution:** Register as `-s, --sourcemap [type]` with `{ default: true }`. The `[optional]` bracket means cac does NOT register `-s` as a boolean in mri. Instead mri treats it as a string option:

- `-s` at end of args or followed by `-flag` → mri returns `true` (no next arg to consume)
- `-s inline` → mri consumes `inline` as the value → `"inline"`
- `--sourcemap hidden` → `"hidden"`

This is actually **better** than current `parseArgs` behavior. With `strict: false`, `parseArgs` doesn't know `-s` takes a value, so `-s inline` loses `inline` to positionals. That's why the current help says "pass the `-s` on the last argument". With cac, `-s inline` works in any position.

### `--config` optional value

`-c` alone → `true` (triggers auto-detect). `-c rolldown.config.js` → `"rolldown.config.js"`.

**cac solution:** Register as `-c, --config [filename]`. cac handles this natively — `[optional]` means value is optional.

### `--no-preserve-entry-signatures` with string default

Current code: `alias.preserveEntrySignatures = { default: 'strict', reverse: true }`. When `--no-preserve-entry-signatures` is passed, it should set `preserveEntrySignatures: false`. When not passed, the default should be `"strict"` (not `true`).

**cac solution:** Register with `{ default: 'strict' }`. cac respects explicit defaults over its auto-`true` behavior for `--no-*` options. Verified in cac source: `if (this.negated && this.config.default == null) { this.config.default = true; }` — the `== null` check means our explicit `'strict'` wins.

### Object options with comma in values

`--transform.define __A__=A,__B__=B` — the value is `__A__=A,__B__=B`, which gets split on `,` then `=`.

**cac behavior:** cac treats this as a single string value `"__A__=A,__B__=B"` for the `--transform.define` option. Our post-processing splits it into `{ __A__: 'A', __B__: 'B' }`. **No change needed.**

### Prototype pollution

`--__proto__.polluted true` or `--constructor.prototype.polluted true`.

**cac behavior:** cac's `setDotProp` does NOT guard against prototype keys. It will happily set `obj.__proto__.polluted = true`.

**Solution:** In post-processing, before calling `normalizeCliOptions`, iterate over parsed options and delete any key that starts with `__proto__`, `constructor`, or `prototype`.

### Unknown options in rawArgs

`rolldown -c rolldown.config.js --customArg=customValue` — unknown options should appear in `rawArgs` so the config function can read them.

**cac behavior with `allowUnknownOptions()`:** `checkUnknownOptions()` runs but is suppressed — unknown options silently appear in the parsed `options` object. We collect them into `rawArgs` by comparing against known schema keys.

## Files Changed

| File                         | Action       | Details                                         |
| ---------------------------- | ------------ | ----------------------------------------------- |
| `cli/arguments/index.ts`     | Rewrite      | Replace parseArgs with cac, add post-processing |
| `cli/arguments/normalize.ts` | Simplify     | Remove unflattening loop + prototype guard      |
| `cli/arguments/alias.ts`     | Simplify     | Remove `reverse`, `requireValue` fields         |
| `cli/arguments/utils.ts`     | Simplify     | Remove `kebabCaseToCamelCase`                   |
| `cli/commands/help.ts`       | Minor update | Adjust to new `options` export shape            |
| `cli/index.ts`               | No change    | Same interface                                  |
| `cli/commands/bundle.ts`     | No change    | Same interface                                  |
| `tests/cli/__snapshots__/*`  | Update       | Regenerate snapshots                            |

Net reduction: ~130 lines. 10 of 16 hacks eliminated entirely.

## Test Cases

`[EXISTING]` = covered in `packages/rolldown/tests/cli/cli-e2e.test.ts`. `[MISSING]` = needs to be added.

| #   | Feature                         | Status       | Example                                                                         | Notes                                                                                       |
| --- | ------------------------------- | ------------ | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 1   | `--version` / `-v`              | `[EXISTING]` | `rolldown --version`                                                            |                                                                                             |
| 2   | `--help` / `-h`                 | `[EXISTING]` | `rolldown --help`                                                               |                                                                                             |
| 3   | Help for empty args             | `[EXISTING]` | `rolldown`                                                                      |                                                                                             |
| 4   | Help precedence over other opts | `[MISSING]`  | `rolldown lib -o dist/lib.js --help`                                            | [#8523](https://github.com/rolldown/rolldown/issues/8523)                                   |
| 5   | Boolean options                 | `[EXISTING]` | `rolldown index.ts --minify -d dist`                                            | Also tests short flag `-m`                                                                  |
| 6   | String options                  | `[EXISTING]` | `rolldown index.ts --format cjs -d dist`                                        |                                                                                             |
| 7   | Short flags                     | `[EXISTING]` | `rolldown index.ts -d dist -s`                                                  | `-d`, `-s`, `-m`, `-c` etc. already covered across tests                                    |
| 8   | Array (repeated flags)          | `[EXISTING]` | `rolldown index.ts --external node:path --external node:url -d dist`            |                                                                                             |
| 9   | Object (repeated flags)         | `[EXISTING]` | `rolldown index.ts --module-types .123=text --module-types .b64=base64 -d dist` | Tests repeated `--module-types` flags                                                       |
| 9a  | Object (comma-separated)        | `[EXISTING]` | `rolldown index.ts --module-types .123=text,notjson=json,.b64=base64 -d dist`   | Tests single flag with comma-separated pairs                                                |
| 10  | `--no-*` boolean negation       | `[EXISTING]` | `rolldown index.ts --no-external-live-bindings ...`                             | One of three `--no-*` options tested                                                        |
| 11  | Nested dot-notation             | `[EXISTING]` | `rolldown index.js --transform.define __DEFINE__=defined`                       | Tests single and comma-separated values                                                     |
| 12  | Positionals as input            | `[EXISTING]` | `rolldown 1.ts --input ./2.js`                                                  |                                                                                             |
| 13  | Config loading (`-c`)           | `[EXISTING]` | `rolldown -c rolldown.config.ts`                                                | Tests `.js`, `.cjs`, `.ts`, `.cts`, `.mts`, auto-detect, multiple configs, multiple outputs |
| 14  | Config function + rawArgs       | `[EXISTING]` | `rolldown -c rolldown.config.js --customArg=customValue`                        | Unknown options passed to config function                                                   |
| 15  | CLI overrides config            | `[EXISTING]` | `rolldown -c rolldown.config.js --format cjs`                                   |                                                                                             |
| 16  | `--environment`                 | `[EXISTING]` | `rolldown -c --environment PRODUCTION,FOO:bar`                                  |                                                                                             |
| 17  | `requireValue` validation       | `[EXISTING]` | `rolldown 1.ts -d` / `rolldown 1.ts -o`                                         | `-d`, `--dir`, `-o`, `--file` without value -> error                                        |
| 18  | Invalid option value            | `[EXISTING]` | `rolldown index.ts --format INCORRECT`                                          |                                                                                             |
| 19  | Unknown option warns (no error) | `[EXISTING]` | `rolldown index.ts --someRandomFlag -d dist`                                    |                                                                                             |
| 20  | Watch mode                      | `[EXISTING]` | `rolldown index.ts -d dist -w -s`                                               | Tests `-w`, watch hooks, multiple configs, `ROLLDOWN_WATCH` env                             |
| 21  | camelCase input ([#8410])       | `[MISSING]`  | `rolldown index.ts --moduleTypes .png=dataurl -d dist`                          | The bug that prompted the migration -- camelCase options are silently ignored               |

[#8410]: https://github.com/rolldown/rolldown/issues/8410

CLI tests: `cd packages/rolldown/tests && pnpm test:cli`

## Related

- [#8410 -- CLI silently mishandles camelCase options](https://github.com/rolldown/rolldown/issues/8410)
- [#8408 -- Closed PR that attempted narrow camelCase fix](https://github.com/rolldown/rolldown/pull/8408)
- [Vite CLI source](https://github.com/vitejs/vite/blob/main/packages/vite/src/node/cli.ts) -- reference for cac usage patterns
