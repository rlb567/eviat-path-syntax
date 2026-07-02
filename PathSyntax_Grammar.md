# Path Syntax — Formal Grammar and Worked Examples

This document is for developers and maintainers of the path parser.
For the LLM-facing reference see `PathSyntax_LLM.md`.
For the machine documentation patch see `MachineDocs_PathPatch.md`.

---

## EBNF Grammar

```ebnf
(* ============================================================ *)
(*  Top-level path                                              *)
(* ============================================================ *)

path            ::= segment ( segment_tail )* projection?

segment_tail    ::= ( '.' segment )
                  | selector_block

segment         ::= name_segment

name_segment    ::= ( letter | '_' | '$' ) name_char*
                    (* '_' start = meta things (_key selector, ._metafield access) *)
                    (* '$' start = private/system tables                            *)
                    (* digits can never start a name                                *)

selector_block  ::= '[' selector_list ']'

selector_list   ::= selector_term ( ';' selector_term )*

projection      ::= '{' proj_field ( ',' proj_field )* '}'

proj_field      ::= name_segment


(* ============================================================ *)
(*  Selector terms - AND'd across the list                      *)
(*  processed in written order (sequential, not set-at-once)   *)
(* ============================================================ *)

selector_term   ::= factor

factor          ::= negation
                  | or_group
                  | meta_selector
                  | field_expr
                  | function_call
                  | keyword
                  | param_marker    (* bare factor = optional insertion point *)

negation        ::= ( '!' | 'not' ) factor

or_group        ::= '(' factor ( '|' factor )+ ')'


(* ============================================================ *)
(*  Meta-selectors  (underscore prefix reserved)               *)
(* ============================================================ *)

meta_selector   ::= key_lookup
                  | index_lookup
                  | sort_by
                  | batch_lookup
                  | limit          (* undocumented - security use only *)

key_lookup      ::= '_key' op ( literal_value | param_marker )

index_lookup    ::= '_index' '=' integer   (* 0-based, negative from end *)
                    (* '=' is the only operator - no ranges *)

sort_by         ::= '_sortby' '=' sort_field ( ',' sort_field )*

sort_field      ::= dotted_field_ref ( 'asc' | 'desc' )?
                    (* direction is tristate: not-set / asc / desc *)
                    (* runtime applies sensible default when not-set, falls back to asc *)

batch_lookup    ::= '_batch' '=' ( literal_value | param_marker )
                    (* members of a persisted frozen batch/selection *)
                    (* membership captured at batch creation - does not drift *)
                    (* '=' is the only operator; id numeric or string *)

limit           ::= '_limit' '=' positive_integer
                    (* always emits ParseWarning when seen in input *)
                    (* multiple _limit terms: effective limit is min() of all *)
                    (* present - user input can tighten but never loosen a *)
                    (* security-injected limit *)


(* ============================================================ *)
(*  Field expressions                                           *)
(* ============================================================ *)

field_expr      ::= dotted_field_ref op rhs_value

rhs_value       ::= literal_value       (* quoted string, base64, or number *)
                  | dotted_field_ref    (* unquoted = field reference *)
                  | param_marker        (* ':' prefix = parameter marker *)
                    (* distinction: quoted = literal, unquoted = field ref, ':' prefix = param *)
                    (* literal on LHS is always a parse error *)

dotted_field_ref ::= name_segment ( '.' name_segment )*
                    (* dots are scope qualifiers resolved at runtime *)
                    (* NOT path navigation - cannot embed [_index] etc *)
                    (* runtime searches up execution context to resolve *)

param_marker    ::= ':' letter marker_char*
                    (* IMPLEMENTED - parser extracts and classifies only *)
                    (* marker names must start with a letter; then [a-z0-9_$] *)
                    (* valid positions: bare factor, rhs_value, _key value, _batch value *)
                    (* NOT valid: path segments, function args, _index, _sortby, projection *)
                    (* bare factor  -> SelectorExprType::ParamMarker, name = marker *)
                    (* value slot   -> paramName set on FieldOp/KeyLookup/BatchLookup *)
                    (* binding is the consumer's job - see Parameter Markers section *)

marker_char     ::= letter | digit | '_' | '$'


(* ============================================================ *)
(*  Function calls                                              *)
(* ============================================================ *)

function_call   ::= keyword '(' arg_list? ')'

arg_list        ::= arg ( ',' arg )*

arg             ::= literal_value
                  | keyword           (* bare word eg yesterday, today *)
                  | date_word         (* eg 2026-01-01 - hyphen allowed in
                                         function args ONLY; a numeric that
                                         runs into '-' re-parses as date_word *)

date_word       ::= ( letter | digit ) ( letter | digit | '-' )*


(* ============================================================ *)
(*  Operators                                                   *)
(* ============================================================ *)

op              ::= '=' | '!=' | '<>' | '<' | '<=' | '>' | '>='


(* ============================================================ *)
(*  Literals                                                    *)
(* ============================================================ *)

literal_value   ::= quoted_string
                  | base64_string
                  | number

quoted_string   ::= '"' [^"\n\r\\]* '"'
                  | "'" [^'\n\r\\]* "'"
                    (* no escape sequences - use base64 for special chars *)
                    (* smart quotes U+201C/201D/2018/2019 are rejected *)
                    (* backslash is always rejected *)

base64_string   ::= 'base64(' [A-Za-z0-9+/=]* ')'
                    (* decode to string at parse time *)
                    (* use for strings containing " or ' characters *)

number          ::= '-'? digit+ ( '.' digit+ )?
                    (* stored as __int64 if no decimal, double if decimal *)


(* ============================================================ *)
(*  Keywords                                                    *)
(* ============================================================ *)

keyword         ::= letter keyword_char*
                    (* table-defined named filters, bool fields become keywords *)

keyword_char    ::= letter | digit
                    (* stricter than name_char - no _ or $ in keywords *)
                    (* _ prefix is reserved for meta-selectors *)


(* ============================================================ *)
(*  Character classes                                           *)
(* ============================================================ *)

name_char       ::= letter | digit | '_' | '$'
                    (* valid in table names and field names *)
                    (* $ valid for private/system tables: bob$products *)

letter          ::= [a-z] | [A-Z]   (* lowercased at parse time *)

digit           ::= [0-9]

integer         ::= '-'? digit+

positive_integer ::= digit+

(* Characters confirmed NEVER valid in any identifier:           *)
(*   whitespace  ( ) [ ] { } | ; , = ! < > " ' @ # % ^ & * +   *)
(*   - / \ ? ~ ` :                                               *)
(*   (extend this list as additional characters are confirmed)   *)
(*                                                                 *)
(* ':' introduces a parameter marker - never valid inside a       *)
(* name/keyword/literal. Only appears as the prefix of a          *)
(* param_marker.                                                   *)
(*                                                                 *)
(* Whitespace: space, tab, CR, LF are skipped between tokens      *)
(* INSIDE selector blocks [...] and projection blocks {...}.      *)
(* Whitespace at path level (around dots, before '[') is a fatal  *)
(* error. Unicode whitespace (eg NBSP U+00A0) is never skipped -  *)
(* it is rejected with a targeted error message.                  *)
```

---

## Precedence and Evaluation Order

Within a selector block `[a; b; c]`:

- Terms are separated by `;` and are **AND'd**
- Terms are evaluated in **written order** — this is intentional and allows sequential processing effects (sort before limit, filter before sort, etc.)
- `|` (OR) must be wrapped in `()` — bare `|` at term level is a parse error
- `not` / `!` bind tightly to the immediately following factor only

Within an `or_group` `(a | b | c)`:

- Alternatives are evaluated left to right, short-circuits on first true

Precedence summary (tightest to loosest):
```
1. Literals, field references, keywords, function calls   (leaf)
2. not / !                                                (prefix unary)
3. ( ... )                                               (grouping / OR group)
4. ;                                                     (AND, implicit, sequential)
```

---

## Worked Examples

Each example shows the input, what the parser produces conceptually, and any notes.

---

### Example 1 — Simple field access

```
products.description
```

Three path segments: `products`, then `.`, then `description`.
No selectors. No projection.
Returns the `description` field for all rows in `products`.

---

### Example 2 — Identity lookup

```
products[_key=345].name
```

Path: `products` → selector `_key=345` → `.name`

Selector is a `KeyLookup` node: op=Eq, value=345 (stored as __int64).
Table defines what "key 345" means internally — caller does not need to know.
Returns the `name` field of the single matching product.

---

### Example 3 — Keyword filter

```
products[vegan].name
```

Selector is a `Keyword` node: name="vegan".
Validator is called at parse time with "vegan".
If validator returns false, this is a fatal parse error.
If no validator is provided, all keywords are rejected.

---

### Example 4 — AND chaining

```
products[vegan; price<10; discontinued=0].name
```

Selector block contains three terms, all AND'd in written order:
1. `Keyword` — "vegan"
2. `FieldOp` — name="price", op=Lt, value=10 (integer)
3. `FieldOp` — name="discontinued", op=Eq, value=0 (integer)

All three must be true for a row to pass.

---

### Example 5 — OR group

```
products[(vegan|glutenfree); price<10].name
```

Two AND terms:
1. `Or` node with two children: Keyword("vegan"), Keyword("glutenfree")
2. `FieldOp` — name="price", op=Lt, value=10

Parens around the OR are mandatory. `[vegan|glutenfree]` is a parse error.

---

### Example 6 — Negation

```
products[not(vegan|glutenfree); price<10].name
```

Two AND terms:
1. `Not` node, child is `Or`(Keyword("vegan"), Keyword("glutenfree"))
2. `FieldOp` — name="price", op=Lt, value=10

Equivalent: rows that are neither vegan nor glutenfree, and cost less than 10.
`!` form: `products[!(vegan|glutenfree); price<10].name` — identical.

---

### Example 7 — Field-to-field comparison

```
sales.items.product_id[items.eachprice<>ticketprice].name
```

Path: `sales` → `.items` → `.product_id` → selector → `.name`

Selector is a `FieldOp`:
- name (LHS) = "items.eachprice"  (dotted field reference - scope resolved at runtime)
- op = Ne  (from `<>`)
- rhsName = "ticketprice"  (unquoted RHS = field reference, not literal)
- pValue = null

Runtime walks up the execution context to find `items.eachprice`.
`items` is a parent level in the current path — resolver finds it there.
`ticketprice` is found on the current table (product_id's target).

---

### Example 8 — Sort and positional

```
products[_sortby=price desc; _index=0].name
```

Two AND terms in written order:
1. `SortBy` — sortFields: [{fieldName="price", direction=Desc}]
2. `IndexLookup` — value=0 (first after sort)

Written order matters here: sort is applied first, then index selects from sorted result.
Result: the cheapest product's name.

---

### Example 9 — Multi-field sort

```
products[_sortby=price asc, name].name
```

`SortBy` with two sort fields:
- {fieldName="price", direction=Asc}
- {fieldName="name",  direction=NotSet}  → runtime applies default (asc)

---

### Example 10 — Projection

```
products[vegan].stocklevels{qty, lastdate}
```

Path: `products` → selector[vegan] → `.stocklevels` → projection{qty, lastdate}

Last segment (`stocklevels`) has Projection = ["qty", "lastdate"].
Projection is only valid as the terminal element of a path.
Caller receives only these two fields, not all stocklevels fields.

---

### Example 11 — Nested table with projection

```
sales.items.product_id[items.eachprice<>ticketprice]{sequence, ticketprice}
```

Path: `sales` → `.items` → `.product_id` → selector → projection

Selector filters rows where the item's each-price differs from ticket price.
Projection returns only `sequence` and `ticketprice` fields.
`product_id` here navigates through the link to the products table;
the runtime knows the link from items.product_id → products internally.

---

### Example 12 — Function call

```
sales[soldafter(yesterday)].total
```

Selector is a `Function` node:
- name = "soldafter"
- args = [RSValueClass2_StdFieldVariant("yesterday")]

"yesterday" is passed as a string. `RDateTime::IsValid("yesterday")` validates at eval time.
`soldafter(2026-01-01)` is equally valid — date format validated by RDateTime, not the parser.

---

### Example 13 — base64 string literal

```
products[name=base64(Qm9iICJ0aGUgbWFuIiBPJ0dyYWR5)].id
```

FieldOp RHS is decoded from base64 at parse time to the string value.
Use base64 when the string contains `"` or `'` characters.
base64 content is validated to `[A-Za-z0-9+/=]` only — safe from injection.

---

### Example 14 — Security injection (_limit)

```
products[vegan; _limit=100].name
```

The `_limit` meta-selector is not documented to users or LLMs.
If it appears in user-supplied input, parser emits a `ParseWarning`:
  "Warning - _limit is a system-level meta-selector. Review query for unexpected use."

It still parses successfully. Security layer can inject `_limit` terms into
`Selectors` at the SymbolPathPart level without going through the string parser at all.
When both user and security supply `_limit`, the effective limit is the **min()**
of all present.

---

### Example 15 — Frozen batch membership

```
invoices[_batch=555]{number, amount}
```

Selector is a `BatchLookup` node: value=555 (stored as __int64).
The batch store defines the membership - captured when batch 555 was created,
it does not drift as invoice data changes. This is what makes bulk writes
addressed via `_batch` safe: the write set is exactly what was previewed.
`=` is the only operator.

---

### Example 16 — Parameter markers (template)

```
products[:uservegantickbox; price<:threshold].name
```

Two AND terms:
1. `ParamMarker` node - name="uservegantickbox". An OPTIONAL insertion point:
   a consuming tool may bind a filter term here; unbound, the term collapses.
2. `FieldOp` - name="price", op=Lt, paramName="threshold", value IsNull().
   A REQUIRED value: unbound at bind/eval time is fatal.

The parser only extracts markers into the AST. Consumers without binding
support abort when any marker is present.

---

## Error Reference

| Situation | Error type | Message pattern |
|---|---|---|
| Unknown keyword | Fatal | `'X' is not a valid selector keyword for this table` |
| No keyword validator | Fatal | `'X' cannot be validated - no validator provided` |
| Bare `\|` outside parens | Fatal | `OR operator \| found outside parentheses` |
| Literal on LHS | Fatal | `Unexpected character at position N` (digit/quote seen where ident expected) |
| Unclosed `[` | Fatal | `Expected ] to close selector block` |
| Unclosed `(` | Fatal | `Expected ) to close OR group` |
| Unclosed `{` | Fatal | `Expected } to close projection block` |
| Projection not terminal | Fatal | `Projection {} must be the last element of a path` |
| Unknown `_xxx` | Fatal | `Unknown meta-selector '_xxx'` |
| Smart quote | Fatal | `Smart quote character detected (U+201C...)` |
| Backslash in string | Fatal | `Backslash escape sequences not supported` |
| `_limit` in input | Warning | `_limit is a system-level meta-selector` |
| `_batch` without `=` | Fatal | `Expected = after _batch` |
| `_batch` with range op | Fatal | `Expected = after _batch ... _batch supports = only` |
| Marker without name | Fatal | `Expected a parameter name after ':'` |
| Marker starts with non-letter | Fatal | `marker names must start with a letter` |
| NBSP in input | Fatal | `Non-breaking space (U+00A0) detected` |
| Non-quote unicode punctuation | Fatal | `Non-ASCII punctuation character detected` |
| Empty projection `{}` | Warning | `Empty projection {} - no fields specified` |
| Depth limit exceeded | Fatal | `Selector expression nesting exceeds maximum depth of N` |
| Term count exceeded | Fatal | `Selector block exceeds maximum of N terms` |
| OR width exceeded | Fatal | `OR group exceeds maximum width of N` |

---

## Security Notes

- **No validator = reject all keywords.** Safe default for hostile input. Never accept keywords without providing a validator tied to MetaDef.
- **`_limit` is always warned** when it appears in parsed input. Security can inject it directly to `Selectors` without parsing.
- **`_limit` collision semantics: min() wins.** When multiple `_limit` terms are present (eg user-supplied plus security-injected), the effective limit is the minimum of all present. User input can tighten a limit, never loosen one.
- **base64 content is character-validated** — `[A-Za-z0-9+/=]` only. No injection risk through base64.
- **Backslash is unconditionally rejected** in string literals — no escape sequence attack surface.
- **Smart quotes produce a clear error** with Unicode codepoint — LLM can self-correct.
- **Depth and term limits** block runaway expressions. Defaults: depth=20, terms=20, OR width=20. Override per instance before decode.
- **`_` prefix in meta-selectors** is structurally reserved. Path segment names may start with `_` (meta-field access, eg `._iszero`), but inside a selector block a leading `_` always routes to meta-selector handling — an unrecognised `_xxx` there is a fatal error, so a field whose name starts with `_` can never be used as a bare keyword. Use a field expression instead.
- **Smart-quote detection is byte-precise**: only the exact sequences for U+2018/2019/201C/201D report as smart quotes; other `E2`-lead punctuation (em-dash, ellipsis) reports as non-ASCII punctuation; `C2 A0` reports as a non-breaking space. All are fatal; the precise message lets an LLM self-correct.

---

## Parameter Markers `:name` — IMPLEMENTED

**Status:** implemented in this parser. The parser **extracts and classifies
only** — it never substitutes values. One parser serves all consumers; the run
engine / binding tool decides validity. Run engines with no binding support
must abort when any marker is present in the AST.

**Where markers are accepted (parser level):**
- Bare factor: `products[:uservegantickbox]` → `SelectorExprType::ParamMarker`, `name` = marker
- RHS of a field expression: `products[price<:threshold]` → `FieldOp` with `paramName` set
- `_key` value: `products[_key=:pid]` → `KeyLookup` with `paramName` set
- `_batch` value: `invoices[_batch=:batchid]` → `BatchLookup` with `paramName` set

**Where markers are rejected (deliberately not implemented):**
- Table/field path segments — those stay fixed, per design
- Function arguments (`soldafter(:datefrom)`) — likely first candidate if extended
- `_index`, `_sortby` values — parameterizing structure, not values; not planned
- Projection `{field, field}` — under consideration, not settled

**Marker name character class:** must start with a letter, then `[a-z0-9_$]`.
Lowercased at parse time. Hyphens are not valid.

**Binding semantics (consumer contract, not enforced by parser):**
- **Bare-factor marker = optional insertion point.** The template author is
  explicitly declaring "a filter MAY go here". Unbound → the term collapses to
  nothing (no filter). `products[:x]` with `:x` unbound behaves as `products[]`.
- **Value-position marker = required.** There is no valid "nothing" for a value
  slot. Unbound at bind/eval time → **fatal error, never an empty result**.
  Fail-open is only permitted where the template author opted in via a
  bare-factor marker.
- Binding is AST substitution into the parsed `SelectorExpr` (fill `value` /
  replace the `ParamMarker` node), never string splicing.
