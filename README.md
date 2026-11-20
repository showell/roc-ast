# roc-ast

**roc-ast prints the parse tree of any Roc file, as Roc's own parser builds
it, as plain text that any program can read.**

The code here was written by AI coding agents (Claude), with human supervision. Point an agent at it, and at [FORMAT.md](FORMAT.md), to build a tool on its trees.

It reads one file at a time, on its own: the app around it, its packages
and its types stay out of it. A file with a type error, a module of a
half-finished refactor or a platform's header all have trees.
That is what a lint, a refactoring tool, a syntax colorer or a codemod
needs: the real structure of the code, with every node's kind and span,
from the parser the compiler itself uses, so it agrees with Roc about what
the code says.

It is one static binary for Linux on x86-64, and it stands alone: download
it, give it paths on stdin, and read the trees.

**It matches one Roc nightly**, the one [`roc-version`](roc-version) names
and the output's first line repeats. [The pin](#the-pin-one-roc-nightly) says what
that means for your code.

## An example

`tests/greet.roc` is a small module:

    module [greet]

    greet : Str -> Str
    greet = |name| "hello ${name}"

Running roc-ast on it prints this, whole:

    $ echo tests/greet.roc | roc-ast --flat
    roc-ast flat 1 nightly-2026-10-30-0602c5f
    == tests/greet.roc
    n	0	file	1	1	4	31
    n	1	module	1	1	1	15
    n	2	exposes	1	8	1	15
    n	3	exposed-lower-ident	1	9	1	14
    n	4	text	0	0	0	0
    a	5	value	greet
    n	1	statements	0	0	0	0
    n	6	s-type-anno	3	1	3	19
    a	7	name	greet
    n	7	ty-fn	3	9	3	19
    n	8	ty	3	9	3	12
    a	9	name	Str
    n	8	ty	3	16	3	19
    a	10	name	Str
    n	6	s-decl	4	1	4	31
    n	11	p-ident	4	1	4	6
    a	12	raw	greet
    n	11	e-lambda	4	9	4	31
    n	13	args	0	0	0	0
    n	14	p-ident	4	10	4	14
    a	15	raw	name
    n	13	e-string	4	16	4	31
    n	16	e-string-part	4	17	4	23
    a	17	raw	hello 
    n	16	e-ident	4	25	4	29
    a	18	raw	name
    n	16	e-string-part	4	30	4	30
    a	19	raw	

The first line names the output format and the Roc nightly whose parser
wrote the tree. Each `==` line starts a file. Each `n` line is a node, its
fields separated by tabs: the number of its parent, Roc's name for its kind,
and its span (start line, start column, end line, end column). Nodes are
numbered by the order of their lines, so the `e-lambda` is node 13, and its
parent is node 11, the `s-decl` on line 4. Each `a` line is an attribute of
a node: node 15, the lambda's parameter, has the `raw` text `name`.

Without `--flat`, roc-ast prints the same tree laid out for people to read,
as the compiler's own snapshot tests print it.

[FORMAT.md](FORMAT.md) is the full contract: the fields, the escapes, how
the tree is taken from Roc's printer, and the exit codes.

## Build your own tools on it

The output is lines of tab-separated fields, so a tool in any language can
read it. As a small example, this lists every name an expression uses, with
its line:

    $ echo tests/greet.roc | roc-ast --flat | awk -F'\t' '/^== /{path=substr($0,4); n=0; next} $1=="n"{n++; kind[n]=$3; line[n]=$4} $1=="a" && kind[$2]=="e-ident" && $3=="raw"{print path":"line[$2]"\t"$4}'
    tests/greet.roc:4	name

A tool should rely on three things:

- **The exit code.** 0 means every file's tree is on stdout. 1 means a file
  had a parse problem, or could not be read, and stderr names it. That
  file gets no tree, since a tree of broken code would be a guess. The
  trees of the files before it are already written, whole, and the run
  stops there.
- **The first line.** It names the format and the Roc nightly. A tool should
  check both before it trusts a tree.
- **Spans count bytes.** Columns are bytes from 1, and an end column is one
  past the node's last byte, so `line`, `col` to `end_line`, `end_col` is the
  node's source text exactly. A node with no span of its own, such as
  `args`, has 0 in all four fields.

## Install

roc-ast runs on Linux on x86-64. The binary is static, so it runs on any
distribution. To install the current release:

    curl -fsSLO https://github.com/showell/roc-apps/releases/download/toolchain-2026-10-30-0602c5f/roc-ast-flat1-0602c5f-linux-x86_64.tar.gz
    echo "5ccb92676cecb41a8b9251737eb418470cd94d7557c9fc5ebf4e2d89e540f8c5  roc-ast-flat1-0602c5f-linux-x86_64.tar.gz" | sha256sum -c
    tar -xzf roc-ast-flat1-0602c5f-linux-x86_64.tar.gz

That unpacks `bin/roc-ast`. Put it on your `PATH` or call it by its path.
The file [`release`](release) always names the current asset's URL and
sha256.

## Companion: roc-types

[roc-types](https://github.com/showell/roc-types) is roc-ast's companion.
roc-ast gives the syntax of any one file. roc-types gives the type of every
expression and pattern, as Roc's checker infers it, but it needs a whole app
that checks. So a tool reads the tree from roc-ast and asks roc-types only
where a type decides: whether `List.map(xs, f)` can become `xs.map(f)`, say,
depends on the type of `xs`, which only the checker knows. Both print spans the
same way, lines and byte columns from 1 with an exclusive end, so a tree's
node and a type's record meet by their spans.

## The pin: one Roc nightly

Roc's new compiler is published as nightly builds, and its parser changes
between them: a nightly may add syntax, rename a node or change a span.
roc-ast is built from Roc's source at one commit, so its trees are that
parser's. Code written for a later nightly may use syntax this parser
refuses, and a tool written against these trees may need
changes for another nightly's.

So check the nightly in the first line of the output against the one your
code is built with. It is also in [`roc-version`](roc-version). A new
nightly needs a new build of roc-ast and a new release. The parser moves
less than the checker, so roc-ast's pin can trail the compiler you build
with, as long as the trees of your code come out whole.

In this repository the pin is held in two places that must agree:

- `roc-version` names the Roc nightly and its commit;
- `build.zig.zon` depends on Roc's source at that commit, and `build.zig`
  refuses to build when the two differ.

## How it works

`main.zig` parses each file with Roc's `parse` module, the parser `roc
check` and `roc build` use, and refuses the file if the tokenizer or the
parser reported any problem. It then builds the tree the compiler's
snapshot tests print, in memory, and walks it, writing a line for each node
and attribute instead of the printed layout. Its header comment, and
FORMAT.md, say how a printed item becomes a node or an attribute.

## How Shale uses it

[Shale](https://github.com/showell/shale) is a scripting tool whose
[Roc recipes](https://github.com/showell/shale/blob/master/cookbook/roc/README.md)
check and rewrite Roc code, and they read it through roc-ast:

- **The reader.** Shale's
  [`roc_tree(paths)`](https://github.com/showell/shale/blob/master/c/preludes/linux.shale)
  runs `roc-ast --flat` once for all the paths and reads its lines in the
  host ([c/host/flat_tree.c](https://github.com/showell/shale/blob/master/c/host/flat_tree.c)),
  giving each file's nodes with their kinds, spans, parents, children and
  attributes. It checks the header's format number first. A file roc-ast
  refuses becomes that file's error, and the files after it are read in
  another run, so every other file still has its tree.
  [parsed.shale](https://github.com/showell/shale/blob/master/cookbook/roc/parsed.shale)
  is the recipes' way in: it reports the refused files after the findings,
  and fails the run.
- **The hunts.** The checks in the cookbook read trees, so comments and
  strings stay out of the code they judge:
  [hunt_names](https://github.com/showell/shale/blob/master/cookbook/roc/hunt_names.shale)
  and the rest, each a line in the cookbook's README.
  [check_foreign_code](https://github.com/showell/shale/blob/master/cookbook/roc/check_foreign_code.shale)
  runs them all over another project's tree. It only parses, so it works on
  a project at any pin.
- **The rewrites and reports.**
  [rewrite_method_calls](https://github.com/showell/shale/blob/master/cookbook/roc/rewrite_method_calls.shale)
  finds each call to rewrite in the tree and asks roc-types whether the
  rewrite is the same call.
  [print_module_types](https://github.com/showell/shale/blob/master/cookbook/roc/print_module_types.shale)
  reads a module's declarations and counts their uses in other files from
  their trees.

The Roc apps in [roc-apps](https://github.com/showell/roc-apps) read trees
the same way. The Safari tutorial's
[color_roc.shale](https://github.com/showell/roc-apps/blob/master/canvas_apps/safari/tutorial/color_roc.shale)
colors each Roc block of the book from roc-ast's tree, and
[roc-syntax-color](https://github.com/showell/roc-syntax-color), a lexer
written in Roc, is held byte for byte to what color_roc makes of the same
code.

With Shale, `shale install.shale` in this checkout downloads the release,
checks its sha256 and unpacks it at `roc-ast/bin/roc-ast` under Shale's
`tools` mount, where `roc_tree` looks for it.

## Building from source

Building roc-ast compiles Roc's parser and the modules it needs, and Roc's
own build runs as a dependency's. It is built on GitHub Actions by
[`.github/workflows/build.yml`](.github/workflows/build.yml), which runs when
started by hand. In a fork, run the `build` workflow from the Actions
tab, or:

    gh workflow run build.yml -f ref=master

The workflow:

- installs the Zig that `zig-version` names, the one the pinned Roc builds
  with (from [patched-zig-compiler](https://github.com/showell/patched-zig-compiler));
- checks out Roc at the pinned commit in full and builds against it with
  `--fork`, since Roc's `vendor/` is outside the archive `build.zig.zon`
  names;
- builds static, for `x86_64-linux-musl` at the baseline CPU, ReleaseSafe;
- runs a smoke test and the fixtures, so a build that fails them makes no
  artifact;
- tars `bin/roc-ast` as `roc-ast-flat<format>-<commit>-linux-x86_64.tar.gz`,
  where `<format>` is the flat format's number and `<commit>` is Roc's
  short commit, and keeps it as the run's artifact, with its sha256 in the
  log.

To build on your own machine, follow the same steps; the workflow file has
the exact commands. To move the pin, change `roc-version`, point
`build.zig.zon` at the new commit (`zig fetch --save=roc
https://github.com/roc-lang/roc/archive/<commit>.tar.gz`), build, and run
the tests.

## Tests

`shale test.shale` runs the installed roc-ast (or `exe=` another) on the
fixtures in `tests/` and judges each run by its exit, the trees it writes
and some of its lines. It runs on Shale.

- `greet`: `tests/greet.roc`, a small module, parsed, its header and nodes
  checked;
- `broken`: `tests/broken.roc`, a parse error, refused with no tree;
- `then_broken`: the two in one run, greet's tree written, then the refusal;
- `plain`: greet without `--flat`, as Roc's printer prints it;
- `bad_arg`: an argument other than `--flat`, refused.

The build workflow checks the first two on every build, with plain shell
commands.

## Publishing a release

Publishing is by hand. Download the workflow's artifact
(`gh run download`), publish it as a release of this repository under a new
name, and put its URL and sha256 in `release`, then in this README's example
header and install commands. A published asset stays as it is; `release`
names where each one lives.

## License

The Universal Permissive License 1.0 ([LICENSE](LICENSE)). The binary is
built from [Roc](https://github.com/roc-lang/roc)'s parser, under the same
license; Roc's copyright notice is in [LICENSE-ROC](LICENSE-ROC).
