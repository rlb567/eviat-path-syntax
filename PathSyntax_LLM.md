# Path Syntax Reference for LLMs

This document describes the path syntax used to access data in a nested table system.
Read it carefully. Follow the rules exactly. Examples marked ❌ are parse errors and will be rejected.

---

## Overview

A path is a dot-separated sequence of table and field names, optionally filtered by selector
blocks in square brackets, and optionally terminated with a projection block in curly braces.

```
products
products.description
products[vegan].description
products[vegan].stocklevels[_key=401].currentonhand
products[vegan].stocklevels{qty, lastdate}
```

Paths are evaluated left to right. Each segment narrows the context.
If any segment returns no data, the path returns empty — this is not an error.

---

## Rules That Always Apply

- All names are **case-insensitive**. Always write lowercase.
- Case rules apply to **names only**. String literals and base64() content are
  **case-preserved** — never change the case of a literal value.
- Whether string comparison is case-sensitive depends on the underlying field.
  Do not rely on case to distinguish values — write literals in the data's natural case.
- Names must start with a **letter**, `_`, or `$` — never a digit.
  Normal tables and fields start with a letter. `_` starts meta things
  (`_key`, `._metafield`). `$` starts private/system tables.
- Table/field names: `[a-z0-9_$]` — letters, digits, underscore, dollar.
- Keywords and function names: `[a-z0-9]` — letters and digits only (no `_` or `$`).
- No spaces, hyphens, or other punctuation inside any name.
- Whitespace (spaces, tabs, newlines) is **ignored between tokens inside `[...]` and `{...}`**.
  Whitespace anywhere else in a path is an **error**: `products [vegan]` ❌, `products . name` ❌.
- The underscore prefix `_` is reserved for meta-selectors (`_key`, `_index`, `_sortby`, `_batch`).
- `:` introduces a parameter marker (see section 3.8) — never valid inside a name.

---

## Part 1 — Basic Paths

### Table name only
```
products
sales
stocklevels
```

### Dot notation for fields
```
products.description
products.name
sales.totalamount
```

### Chained dots for nested tables
When a field links to another table, dot notation continues into it.
The nested table is **automatically pre-filtered** to rows relevant to the parent.
You do not write a join condition.
```
products.stocklevels.currentonhand
sales.items.unitprice
```

---

## Part 2 — Selector Blocks `[...]`

Selectors filter which rows are included.
Multiple terms are separated by `;` and are **AND'd in written order**.

```
products[vegan].name
products[vegan; price<10].name
products[vegan; discontinued=0; price<100].name
```

### ❌ Common mistakes
```
products[vegan, price<10].name     ❌  comma is not AND — use semicolon ;
products[vegan price<10].name      ❌  missing semicolon between terms
```

---

## Part 3 — Selector Types

### 3.1 Keywords

A bare word that the table defines as a named filter. Bool fields also become keywords.

```
products[vegan]
products[discontinued]
sales[soldtoday]
```

Keywords are table-specific. Using an unrecognised keyword is a **fatal error**.

### ❌ Invalid keywords
```
products[rboert]       ❌  "rboert" is not a valid keyword for products
products[free items]   ❌  spaces not allowed — no spaces in any name
```

---

### 3.2 Field Expressions

**Syntax:** `fieldname OPERATOR value`

Operators: `=` `!=` `<>` `<` `<=` `>` `>=`

```
products[price=0]
products[price>10]
products[price<=99]
products[price!=0]
products[price<>0]        same as !=
```

**Field-to-field comparison** — unquoted name on RHS means field reference:
```
products[price<ticketprice]
orders[total>=minimumorder]
```

**Dotted field references** — scope qualifier, resolved at runtime:
```
sales.items.product_id[items.eachprice<>ticketprice].name
```
`items.eachprice` looks up the `eachprice` field from the parent `items` context.

**The left side must always be a field name — never a literal.**
```
products[10>price]     ❌  literal on left side — rewrite as products[price<10]
products[0=price]      ❌  rewrite as products[price=0]
```

---

### 3.3 String Literals

String values must be **quoted**. Prefer double quotes. Single quotes also accepted.

```
products[name="widget"]
products[category="food and drink"]
products[code='ABC123']
products[name="Bob's cafe"]        a single quote inside double quotes is fine
```

No escape sequences. Enclosing one quote character in the other kind is the
normal form. Only when a string contains **both** quote characters do you need `base64()`.

**base64() for special strings:**
```
products[name=base64(Qm9iICJ0aGUgbWFuIiBPJ0dyYWR5)]
```

Content inside `base64()` must be valid base64: `[A-Za-z0-9+/=]` only.

String literals are **case-preserved** — write them in the data's natural case.

### ❌ Invalid string syntax
```
products[name=widget]                      ❌  unquoted — treated as field name, not string
products[name="say \"hi\""]               ❌  backslash not supported — use base64()
products[name=<U+201C>widget<U+201D>]      ❌  smart/curly quotes — use ASCII " or '
products[note="a<U+00A0>b"]                ❌  non-breaking space — use a regular space
```
(`<U+201C>` above stands for the actual curly-quote character — shown as a
codepoint so document tooling cannot silently normalise it. Never emit curly
quotes or non-breaking spaces; the parser rejects them with a precise message.)

---

### 3.4 Numeric Literals

```
products[price=0]
products[price>9.99]
products[stockcount>=1]
products[margin>-10]
```

---

### 3.5 Date Arguments in Functions

Dates are validated by the system's date parser. Use ISO format or natural words.

**Valid:**
```
yesterday    today    tomorrow    2026-01-01    2026-06-27
```

**Invalid:**
```
10-04-2026     ❌  ambiguous — use YYYY-MM-DD
04/10/2026     ❌  ambiguous — use YYYY-MM-DD
```

---

### 3.6 Meta-Selectors

Meta-selectors begin with `_` and are structural — they operate on the row set,
not just filter it.

#### `_key` — identity lookup

Selects by primary key. The exact meaning of the key is table-specific.

```
products[_key=345]
stocklevels[_key=401].currentonhand
products[_key="ABC-001"]
```

Supports all operators: `=` `!=` `<` etc. `=` is overwhelmingly the common case.

#### `_index` — positional lookup

0-based. Negative values count from the end. **`=` is the only operator** —
`_index` does not support ranges.

```
sales.items[_index=0].name         first item
sales.items[_index=-1].name        last item
sales.items[_index=-2].name        second to last
```

Out of range returns empty — not an error.
Order is stable within a run but may change if underlying data changes.

#### `_batch` — frozen batch/selection membership

Selects the members of a previously created batch. Batch membership is
**frozen when the batch is created** — it does not drift as underlying data
changes. **`=` is the only operator.** The id may be numeric or a quoted string.

```
invoices[_batch=555].total
invoices[_batch=555]{number, amount}
invoices[_batch="B2026-07"].supplier
```

#### `_sortby` — sort the row set

Comma-separated list of fields with optional direction.
Direction is optional — runtime applies a sensible default per field, falling back to `asc`.

```
products[_sortby=name]
products[_sortby=price desc]
products[_sortby=price desc, name]
products[_sortby=price asc, name desc, stockcount]
```

Combine with `_index` to get a specific row after sorting:
```
products[_sortby=price desc; _index=0].name        most expensive product
products[_sortby=lastdate desc; _index=0].qty       most recently updated stock level
```

Written order of terms matters — `_sortby` before `_index` sorts first, then selects.

### ❌ Invalid meta-selector syntax
```
products[key=345]          ❌  missing underscore
products[index=0]          ❌  missing underscore
products[_key]             ❌  _key requires an operator and value
products[_index=first]     ❌  _index requires an integer
products[_index<5]         ❌  _index supports = only — no ranges
products[_sortby]          ❌  _sortby requires = and at least one field name
products[_batch]           ❌  _batch requires = and a batch id
products[_batch>5]         ❌  _batch supports = only
```

---

### 3.7 Functions

**Syntax:** `functionname(arg1, arg2, ...)`

Arguments separated by commas. Dates validated at evaluation time.

```
sales[soldafter(yesterday)].total
sales[soldbefore(2026-01-01)].total
sales[soldbetween(2026-01-01, 2026-06-27)].total
```

### ❌ Invalid function syntax
```
sales[sold-after(yesterday)]    ❌  hyphens not allowed — use soldafter
sales[soldafter(10-04-2026)]    ❌  ambiguous date — use YYYY-MM-DD or natural words
sales[soldafter()]              ❌  missing argument
```

---

### 3.8 Parameter Markers `:name`

A marker is a **placeholder for a value or filter supplied later by a tool**,
not by you. Use markers only when building a **template** that another tool
will fill in. Never put markers in a path meant to run directly — engines
without binding support reject any path containing a marker.

Marker names: `[a-z0-9_$]`, must start with a letter.

**Two positions, two meanings:**

**1. Whole term — an optional insertion point.** The consuming tool MAY insert
a filter here. If it doesn't, the term simply disappears (no filter applied).

```
products[:uservegantickbox].name
products[:uservegantickbox].salelines[:userdaterange].total
```

**2. Value position — a required value.** Replaces a literal after an operator.
The consuming tool MUST supply a value; an unbound value marker is a fatal error.

```
products[price<:threshold].name
products[_key=:pid].name
invoices[_batch=:batchid].total
```

### ❌ Invalid marker syntax
```
products[:user-vegan].name     ❌  hyphens not allowed in marker names
products[:].name               ❌  marker requires a name
products[:_bad].name           ❌  marker names must start with a letter
products.:segment              ❌  path segments cannot be markers — only selector positions
sales[soldafter(:from)]        ❌  markers not supported in function arguments (yet)
```

---

## Part 4 — Boolean Logic

### AND — semicolon `;`

Terms separated by `;` are AND'd in written order.

```
products[vegan; price<10]
products[vegan; discontinued=0; stockcount>0]
```

### OR — pipe `|` inside parentheses

**Parentheses are required.** A bare `|` outside parens is always a fatal error.

```
products[(vegan|glutenfree)].name
products[(vegan|glutenfree); price<10].name
```

### ❌ OR without parentheses
```
products[vegan|glutenfree].name         ❌  pipe must be inside parens
```

### NOT — `!` or `not`

Both forms are equivalent.

```
products[!discontinued].name
products[not discontinued].name
products[not(vegan|glutenfree)].name
products[!(vegan|glutenfree)].name
```

### Combining AND, OR, NOT

```
products[(vegan|glutenfree); price<10; not discontinued].name
```
Reads as: `(vegan OR glutenfree) AND price<10 AND NOT discontinued`

---

## Part 5 — Projection `{field, field, ...}`

A projection block at the end of a path specifies which fields to return.
**Projection is only valid as the last element of a path.**

```
products[vegan]{name, price}
products{name, description, unitprice}
products[vegan].stocklevels{qty, lastdate}
sales.items.product_id[items.eachprice<>ticketprice]{sequence, ticketprice}
```

Without projection, all fields are returned.

### ❌ Projection not terminal
```
products{name}.stocklevels     ❌  projection must be last — nothing can follow }
```

---

## Part 6 — Nested Table Paths

```
products[_key=345].stocklevels.currentonhand
```
Stock levels for product 345. Nested table is pre-filtered — no join needed.

```
products[vegan].stocklevels[_key=401].currentonhand
```
Step by step:
1. `products[vegan]` — all vegan products
2. `.stocklevels` — stocklevels pre-filtered to those products
3. `[_key=401]` — further filter to store 401
4. `.currentonhand` — read the field

```
products[vegan].stocklevels[_key=401]{qty, lastdate}
```
Same, with projection returning only two fields.

---

## Part 7 — Complete Examples

```
products.name
products[_key=345].name
products[vegan; price<10].name
products[(vegan|glutenfree); not discontinued].name
products[_sortby=price desc; _index=0].name
products[_sortby=price desc; _index=0]{name, price}
products[vegan].stocklevels[_key=401].currentonhand
sales[soldafter(yesterday)].items[_index=-1].productname
sales.items[total=0].productname
sales.items.product_id[items.eachprice<>ticketprice]{sequence, ticketprice}
products[name=base64(Qm9iICJ0aGUgbWFuIiBPJ0dyYWR5)].id
invoices[_batch=555]{number, amount}
products[:uservegantickbox; price<:threshold].name        (template only)
```

---

## Part 8 — Error Messages

| Message contains | What to do |
|---|---|
| `not a valid selector keyword` | Keyword not recognised — check spelling or use a field expression |
| `no validator provided` | System configuration issue — report to developer |
| `OR operator \| found outside parentheses` | Wrap OR in parens: `(a\|b)` |
| `literal on left side` | Swap: `10>price` → `price<10` |
| `Expected ]` | Unclosed `[` — add closing `]` |
| `Expected }` | Unclosed `{` — add closing `}` |
| `Projection {} must be last` | Nothing can follow `}` in a path |
| `Smart quote character detected` | Replace curly quotes with straight ASCII `"` or `'` |
| `Non-breaking space (U+00A0) detected` | Replace with a regular space |
| `Non-ASCII punctuation character detected` | Replace with the plain ASCII equivalent |
| `Backslash escape sequences not supported` | Use `base64()` for special characters |
| `Expected = after _batch` | `_batch` supports `=` only, with a numeric or string id |
| `Expected a parameter name after ':'` | Marker names start with a letter: `:threshold` |
| `ambiguous date format` | Use `YYYY-MM-DD` or natural words: `yesterday`, `today` |
| `nesting exceeds maximum depth` | Simplify — too many nested NOT/OR levels |
| `exceeds maximum` | Simplify — too many terms or OR alternatives |

---

## Quick Reference

```
products                                         table cursor
products.name                                    field access
products[vegan]                                  keyword filter
products[price<10]                               field expression
products[price<ticketprice]                      field-to-field
products[items.eachprice<ticketprice]            dotted field reference
products[name="widget"]                          string literal
products[name=base64(XXXX)]                      string with special chars
products[soldafter(yesterday)]                   function
products[_key=345]                               identity lookup
products[_index=0]                               first item
products[_index=-1]                              last item
invoices[_batch=555]                             frozen batch membership
products[_sortby=price desc]                     sort
products[_sortby=price desc, name]               multi-field sort
products[_sortby=price desc; _index=0]           sort then select first
products[vegan; price<10]                        AND
products[(vegan|glutenfree)]                     OR — parens required
products[!discontinued]                          NOT
products[not discontinued]                       NOT readable form
products[not(vegan|glutenfree)]                  NOT of OR group
products[(vegan|glutenfree); price<10]           OR inside AND
products{name, price}                            projection
products[vegan].stocklevels[_key=401].qoh        nested tables
products[vegan].stocklevels{qty, lastdate}       nested with projection
products[:optionalfilter]                        template: optional insertion point
products[price<:threshold]                       template: required value marker
```
