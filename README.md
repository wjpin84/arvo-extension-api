# The Arvo extension API

Everything you need to extend [Arvo](https://github.com/wjpin84/arvo-desktop)
without any of Arvo in front of you. There is no code in this repository: a
schema, a protobuf definition, and what the two of them do not say.

Arvo itself is closed. This is not, because the people who implement a
boundary have to be able to read it.

## An extension is a folder with a manifest

`arvo-extension.json` at its root, and whatever else it ships. Arvo installs
one from a GitHub repository, or from a folder dropped into its extensions
directory.

```json
{
  "id": "midnight",
  "name": "Midnight",
  "version": "0.1.0",
  "description": "A dark theme.",
  "contributes": {
    "themes": [
      { "id": "deep", "label": "Midnight Deep", "base": "dark",
        "colors": { "--color-bg": "#0b0d12" } }
    ]
  }
}
```

[`manifest.schema.json`](manifest.schema.json) is the whole of it. Everything
in `contributes` is optional; an extension may contribute several kinds at
once.

## Two ways to contribute, and the line between them

**Declarative.** Data Arvo reads and renders with its own components: themes,
and strategies as documents. No code runs, in the window or anywhere else.
This is the wide surface and the one to reach for first.

**A provider.** A contribution that needs real logic runs as its own process,
speaking gRPC over loopback, started and stopped by Arvo. The process
boundary is the sandbox. [`PROTOCOL.md`](PROTOCOL.md) is how that process is
started and how it is spoken to; [`proto/`](proto) is what it must serve.

What is deliberately not here is a third way. No third-party code runs inside
Arvo's window, in any form, ever.

## A strategy as a document

A strategy expressible as data can be diffed, generated in batches and
shipped in an extension. The document names a **kind** and carries that
kind's parameters.

```json
"strategies": [
  {
    "name": "fast-cross",
    "label": "Faster crossover",
    "premise": "The same rule Arvo ships, over a tighter grid.",
    "interval": { "step": 1, "unit": "day" },
    "kind": "grid",
    "rule": "sma_cross",
    "axes": { "fast": [3, 5], "slow": [20, 40] }
  }
]
```

A `grid` document searches a rule **Arvo implements** over parameters you
choose. It cannot contribute a rule: a document naming a rule Arvo does not
have is refused when the manifest is read, with the list of what it does
have. Fixed parameters come from the rule and you override the ones you name.

A `rules` document — thresholds over named signals — is read but not yet run:
Arvo has no runner for one, and says so rather than dropping it.

### A rule over a value that is not there is false

The one invariant worth stating twice. A signal may be absent: not
published, or published with nothing in it. A predicate over an absent value
is **false** — never zero, never true, never the last known value. An empty
rule set never fires either, in either direction.

This exists because the alternative is quiet. An indicator with too little
history is absent where it is computed and `0.0` by the time a rule reads it,
and `0.0` is a value a threshold matches.

## Providers

See [`PROTOCOL.md`](PROTOCOL.md). In short: Arvo starts your binary with an
address to bind and a token to require, you print one line saying which port
you took, and you serve [`arvo.source.v1.Source`](proto/arvo/source/v1/source.proto)
until you are stopped. Any language with gRPC will do.

A provider declares itself in the manifest, with how to build it and what to
run — and, for a machine with no toolchain, where to download it:

```json
"providers": [
  {
    "id": "acme",
    "services": ["arvo.source.v1.Source"],
    "build": "cargo build --release",
    "run": "target/release/acme-source",
    "assets": [
      { "target": "x86_64-pc-windows-msvc",
        "url": "https://github.com/you/acme/releases/download/v0.1.0/acme-source-x86_64-pc-windows-msvc.exe",
        "sha256": "…" }
    ]
  }
]
```

`build` and `run` are argv, split on whitespace. No shell runs them, so a
manifest can name a program and its arguments and cannot smuggle a second
command through a semicolon.

When every provider offers an `assets` entry for the running platform, Arvo
downloads those and builds nothing. The `sha256` is required and checked
before the file is written: this is an executable Arvo will run with the
person's privileges.

## Versioning

The proto changes additively, as proto3 requires: new fields, new messages,
never a renumbering. A plugin built against an older copy keeps working.

Nothing here pins to a version of Arvo, and nothing should need to.

## Reference plugins

[`arvo-plugin-yahoo`](https://github.com/wjpin84/arvo-plugin-yahoo) and
[`arvo-plugin-alpaca`](https://github.com/wjpin84/arvo-plugin-alpaca) are two
real ones, with the release pipeline that produces the assets above.
