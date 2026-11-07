# Inia 0.2

This is the English edition. `SPEC.zh-Hant.md` and `SPEC.ja.md` are parallel
editions of the same specification; no edition is declared controlling.

## 1. The idea

An Inia document is an ordered series of writes. Evaluation starts with an
empty root object, applies the writes in document order, then resolves
deferred selectors, defaults, and references. The result is a tree of
strings, arrays, and objects. Inia does not assign meanings such as number or
boolean to strings.

Inia keeps the familiar INI shape:

```ini
name = value

[section]
name = value
```

It uses `.ini` files and `;` comments, but it is not compatible with the
Windows profile APIs.

## 2. Text

Input is UTF-8. A leading BOM is ignored. CRLF and CR become LF. An LF inside
a quoted span remains part of that span.

Whitespace means SP and HTAB. Outside quotes, leading and trailing whitespace
is removed from names and values. Other Unicode whitespace is content.

A comment line begins with `;` after optional whitespace. An end-of-line
comment begins at a `;` outside quotes when a whitespace character precedes
it. `a;b` therefore contains a literal semicolon. `#` is not a comment mark.

Quoting is the only escape mechanism. A `"` at the start of a token opens a
quoted span; inside it, `""` is one literal quote and the next single `"`
closes the span. Everything else is literal. A quote elsewhere is literal.

```ini
empty = ""
quote = """"
text = """a, b"
literal = a"b
```

These values are respectively empty, `"`, `"a, b`, and `a"b`. Quoted text
does not become a path separator, index, operator, reference, or delimiter.

Names compare by Unicode simple case folding (CaseFolding status C and S).
No Unicode normalization is applied. The first spelling is retained.

## 3. Values

The value form is determined by its own text, not by the surrounding node.

- **Single values** contain one token: `a`, `"a"`, and `a"b` are examples.
- **Aggregate values** are complete values written as `(...)` or `[...]`.
- Single values and aggregate values together are **whole values**.
- **Empty values** and **column values** together are **zero values**. "Zero"
  describes a scattered form, not the number of items: an empty value is one
  zero item, while a column can combine zero items.

The output still has only three node types: string, array, and object. A single
value normally evaluates to a string, but the syntactic category "single value"
is not the same thing as the output type "string".

`key =` is a no-value. `""` is a single value containing an explicit empty
string. `(...)` is an aggregate array or object-array value, and `[...]` is an
aggregate object value.

An unquoted comma is an optional terminator. A logical line end also
terminates a column item. There is no second ordinary separator.

```ini
a = (1,2,3)
b = (1,2,3,)
c = (1,2,3,,)
literal = "1,2,3"
pair = k=v
```

`a` and `b` contain three items. `c` contains three values and one no-value.
`"1,2,3"` is one single value. `k=v` inside a column is a pair item; in
`pair = k=v`, the first `=` separates the address from the single-value RHS,
so the value is the literal `k=v`. `k=v,` and `k=v` are equivalent as column
items.

`(...)` and `[...]` are complete values. An ordinary write of a complete
value replaces the previous complete value; it does not perform an implicit
deep merge. A column can contribute items to an enclosing array or object.

## 4. Documents and sections

A document contains blank lines, comments, headers, key-value lines, and the
standalone `###` line. Use `###` mainly to leave a section and open the top
level again; lines before the first header already write to the root object.

```ini
[S]
a = 1
###
b = 2
```

These headers are defined:

```ini
[P]                 object section
[P()]               array section
[P(=)]              object-array section without a primary key
[P(k1=k2)]          object-array section with a mapping
[P::element]        addressed element section
[P::(condition)]    selector section
```

`P` is a path. A header opens the addressed node and gives a newly created
node the declared type. Reopening a node with another type is an error.

A section lasts until the next header, `###`, or end of document. `###` ends
the current section and returns to the root; it does not close an explicit
key-side block.

An object section contains ordinary key-value and operation lines. An array
section addresses positions or appends through columns and operations. An
object-array section contains element lines of the form `slot = value`; with
a mapping, `slot` and the first item of `value` fill the declared first-pair
fields. A selector section applies each body line to every hit in document
order.

## 5. Paths

A path joins segments with `::`, without whitespace touching the separator.

| Segment | Meaning |
|---|---|
| `name` | object key, or primary-key lookup on an object array |
| `"name"` | always a name |
| `123` | position on an array or selection; name on an object |
| `~123` | position counted from the end; `~1` is last |
| `(k=v, ...)` | condition selector |
| `(name)` | selector containing a path |
| `()` | all-elements selector |
| `((sel))` | immediate selector |
| `::name` | hidden name |

Positions start at 1. Position 0 and an out-of-range reverse position are
missing. A numeric primary-key value is addressed with a quoted segment.
After an empty segment, the next segment is always a name, so `P::::1` is
the hidden name `1`.

`(k=v, ...)` selects every element satisfying all conditions. A condition
value is a string or a reference yielding a string. Missing fields do not
match; object and array fields are invalid condition values.

`(name)` is the selector form of a path. Thus `path::(sub::name)` first
resolves `path`, then applies the selector `sub::name`. A failed selector may
be deferred. `((sel))` makes the complete selector immediate.

Operations are prefixes on the address, before its first segment:

```text
!x       delete x
<x       insert before x
>x       insert after x
<        insert at the beginning
>        insert at the end
?P::x    skip a missing target
```

Write `?` before the address, as in `?P::x = 1`. It must be the first prefix.
It suppresses an applicable missing-target error immediately; it never creates
a deferred selector.

## 6. Blocks and aggregation

Sections are implicit blocks. Explicit key-side blocks use markers belonging
to the addressed key:

```ini
[S]
list::( = a,
list::1 = b
list::) = c,
```

`::(` opens an array block and `::)` closes it. Object blocks use `::[` and
`::]`. Opening and closing lines may carry content.

Outside an aggregate or operation column, an ordinary assignment overwrites.
Inside an aggregate, column, or key-side block, repeated keys and positions
accumulate when their value form has a defined column meaning:

```ini
::1 = a
::1 = b
```

contributes `a` and `b` at successive positions. Complete strings and
complete aggregates have no general merge operation; a collision requiring
one is invalid.

`<` and `>` are insertion operations, not object merges. Repeated operations
with the same target and direction are folded into one operation column and
execute once. A selector target applies the operation to every hit.

## 7. Mappings and defaults

A mapping before an aggregate declares the first-pair slots of an object
array:

```text
(name=key)   slot -> name, first value -> key
(=key)       discard slot, first value -> key
(name=)      slot -> name
(=)          discard both
```

A default prefix is local to the following aggregate text:

`|default,k=v,k2=v2(...)`

`|v` is the default value; `|` is the empty default value. Default pairs fill
missing pairs in that aggregate text. They do not recurse into nested
objects. A nested `|w` replaces the inherited default value; a nested `|`
resets it to the empty string. Explicit values, explicit `""`, and tombstones
always beat defaults.

When no default applies, a no-value resolves to the empty string.

## 8. References

`@path` reads from the final tree and applies the expansion rule of the result
value. A path with one hit returns that value. A path with multiple hits
returns an aggregate array. An array value expands by one level; its elements
remain elements and are not expanded recursively. Thus a multiple selection
keeps its selected elements, while a referenced array contributes its own
elements.

On a selection, an index chooses one element, `~n` counts from the end, a
selector filters again, and a name projects that field from every element that
has it. `(@ref)` and `[@ref]` are ordinary array and object aggregate syntax.

File references are distinct from string values:

```ini
whole = @[file]
part = @[file]::section
text = @["file"]
```

The first two parse a file as Inia; the last reads the file as one string.
Relative paths are resolved by the parser. Missing files, invalid files, and
file cycles are errors.

## 9. Selectors and evaluation

Resolve every address from left to right. A missing ordinary name, index, or
primary-key lookup fails immediately. When a selector has no hit, defer the
entire addressed write instead of applying it. Retry deferred writes in
document order after later writes have been applied. If a selector still has
no hit after the final retry, report E-NOMATCH. `((sel))` requires its selector
to succeed immediately. A `?` address skips its applicable missing-target
failure immediately and never enters the deferred queue. Multiple selector
hits receive the write in document order.

## 10. Primary keys

The first name in `(k1=k2)` is the primary-key field of the array. All
declarations for one array must use the same `k1`. Primary-key values are
unique in the final array.

Defaults are applied before selector conditions are compared. References read
the final tree after defaults and deferred selectors have settled. Hidden keys
and tombstones are removed from the output.

## 11. Errors

An implementation must reject input with undefined syntax or semantics. The
following names describe the main failures:

| Code | Meaning |
|---|---|
| E-SYNTAX | malformed text, value, header, quote, or block |
| E-TYPE | incompatible node type or operation |
| E-INDEX | missing left-hand position |
| E-NOMATCH | missing ordinary or selector target |
| E-NOPK | name lookup on an array without a primary key |
| E-PKDECL / E-PKDUP | inconsistent or duplicate primary-key declaration |
| E-CYCLE / E-DANGLING | reference or deferred-address cycle / missing target |
| E-DUPKEY | duplicate item with no legal column aggregation |
| E-DELVAL | deletion written with a value |

## 12. Small complete example

```ini
title = demo

[services(name=id)]
A = url-a, env=prod
B = url-b, env=dev
C = url-c, env=prod

[services::(env=prod)]
enabled = yes

[result]
names = (@services::()::name)
```

The selector edits `A` and `C`. The final `names` value is
`["A", "B", "C"]`. All values remain strings unless an application assigns
its own interpretation.
