# roc-ast's flat format

This is the contract between `roc-ast --flat` and the programs that read
it. A change a reader must know of is a new format number, in the header
line and in `main.zig`'s `flat_format`. This file describes format 1.

## Input

roc-ast reads the paths of the files to parse from stdin, one a line. Spaces,
tabs and a carriage return around a path are trimmed, and a blank line is
skipped. A relative path is read from the working directory. Each file is
parsed alone, so it may be any Roc file: an app, a module, a platform's or a
package's, whether or not the code would type-check.

The one argument is `--flat`. With none, roc-ast prints each tree as the
compiler's snapshot tests print it (`(file @1.1-4.31 ...`), which is for
people to read and is not part of this contract.

## Stdout

The first line names the format and whose parser wrote the trees. It is
written and flushed before any file is read, so a reader sees it even when a
later file stops the run:

    roc-ast flat 1 <roc-version>

`<roc-version>` is the whole line of `roc-version`, such as
`nightly-2026-10-30-0602c5f`.

Then, for each file, in the order the paths came:

    == <path>
    n<TAB>parent<TAB>kind<TAB>line<TAB>col<TAB>end_line<TAB>end_col
    a<TAB>node<TAB>name<TAB>value

- **The path** is written as it was given, after the trim.
- **An `n` line** is a node. Nodes are numbered from 1 in each file, in the
  order their lines come, so the third `n` line after `==` is node 3. The
  number is not written.
- **`parent`** is the number of the node that holds this one, and 0 for the
  file's root, which is always node 1, of kind `file`. A parent always comes
  before its children.
- **`kind`** is Roc's name for the node, as its printer gives it:
  `s-decl`, `e-lambda`, `p-ident`, `ty-fn`, `e-apply`, `args`.
- **The span** is where the node is in the file. Lines and columns count
  from 1, columns are in bytes, and the end column is one past the last
  byte. A node without a span, such as `args` or `statements`, has `0` in
  all four fields. The file node starts after any leading comments, and the
  spans of two nodes can overlap (`type-mod` and `s-type-decl` both start at
  a type's name).
- **An `a` line** is an attribute of node `node`, which has already been
  written: `raw` (the source text of a name or a string part), `name`,
  `op`, `method`, `field`, `type`, and so on, as the printer names it. A
  scalar the printer gives a node without a name, such as the string in
  `(text "greet")`, is its `value`.
- **Escapes.** In a kind, a name or a value, `\`, tab, newline and carriage
  return are written as `\\`, `\t`, `\n` and `\r`. No other byte is escaped.

A node's attributes and children come in the printer's order, each child
with everything under it before the next item. So an attribute usually
follows its node's line directly, but a reader should go by the `node`
field, not by position.

## How the tree is taken from Roc's printer

The tree is the one `AST.toSExprStr` prints for the compiler's snapshot
tests, built in memory by the same code (`pushToSExprTree`) and walked
instead of printed, so nothing is parsed back out of text laid out for eyes.
The printer puts a node's head items on its own line, or with none, its
first child. On that line, a `(name value)` pair is an attribute and any
other `(...)` is a child node. On a line of its own, every `(...)` is a
child node. A string or atom anywhere is the node's `value`.

## Stderr and the exit status

| exit | meaning |
|---|---|
| 0 | every file parsed; stdout holds every file's tree |
| 1 | a file did not parse cleanly (any tokenize or parse problem), or could not be read |
| 2 | an argument other than `--flat` |

A file that does not parse is never a tree. roc-ast writes
`<path>: <N> parse problem(s): ...` to stderr and stops. The trees of the
files before it are already on stdout, whole; that file and the files after
it have none. A reader that wants the files after it runs roc-ast again
without the refused one. A path that cannot be read stops the run the same
way, with Zig's error name (`error: FileNotFound`) on stderr.

## An example

`tests/greet.roc` is:

    module [greet]

    greet : Str -> Str
    greet = |name| "hello ${name}"

From a checkout at `/home/me/roc-ast`,
`echo /home/me/roc-ast/tests/greet.roc | roc-ast --flat` prints, whole:

    roc-ast flat 1 nightly-2026-10-30-0602c5f
    == /home/me/roc-ast/tests/greet.roc
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

The `e-lambda` is node 13, on line 4 from column 9 to column 30 (31 is one
past its last byte). Its parent is node 11, the `s-decl`, and its children
are node 14, the `args`, and node 16, the `e-string`. The last line's value is empty: the string's last part, after
`${name}`, holds no bytes, so the line ends with a tab.
