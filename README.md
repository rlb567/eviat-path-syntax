# eviat-path-syntax
A simple path description format to allow selection from a table and subtables.  Very similiar to javascript and other systems, but includes some of our special needs and is LLM/AI friendly.  The path description clearly describes -what- you want, not -how to get it-

Used in our MCP tools as a way for LLM to request rows/columns and subtables in a structured manner. More constrained than SQL, and has special nuances we require. Human readable, but fixed structure for LLM generation.

## Using

Paste the spec into your favorite AI, and ask it to generate a parser in whatever language you want.

## Path examples

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
products[:uservegantickbox; price<:threshold].name
```

Explanations

* products[_sortby=price desc; _index=0]{name, price}
  
Return a grid/table/array of objects which are equivalent to sql "select name,price from products order by price desc limit 1 row"

* products[_key=345].name

select the name for product 345 - where 345 is a primary key.  _key means whatever the primary key is called, the execution engine insert the real name

* products[vegan].stocklevels[_key=401].currentonhand

select all products marked vegan, however that might be implemented, and for that subset of products, load all the matching stocklevels for store=401 and return currentonhand values

* products[:uservegantickbox; price<:threshold].name

This format contains a parameter  :uservegantickbox - In our usage, parameters must be replaced or removed before execution, as our execution engine does not offer parameter passing - they must be in the selector.  You are free to allow/disallow as you require.  We use the parameter versions for design side tooling as it makes explicit where variables should be inserted.
