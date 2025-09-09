+++
title = "A first foray into Arrow (Part 1: Python)"
date = 2025-09-07
updated = 2025-09-07
description = "A wandering through what Arrow is with real code. First, a look at Arrow from on-high, using PyArrow."
draft = false
+++


If we borrow a page from Aristotle, to learn a thing--to understand its causes<sup>[1]</sup>--we must first accumulate practical experience with it and hone our craft; only after can we analyze effectively.

This post begins a short series. We embark with Arrow in Python. Next, I will turn to Arrow in Rust, then survey Arrow's history, and finally, close by developing Arrow's key invariants.

<br>

---

1. [_hello columnar_](#hello-columnar)
    - [_one row becomes two_](#one-to-two)
1. [_columns and taking data in stride_](#section_2)
    - [_example: table schema_](#suba)
    - [_example: columns_](#subb)
    - [_example: rows_](#subc)
1. [_records in batches_](#section_3)
1. [_caveat lector_](#section_4)
1. [_parting thoughts_](#section_5)
1. [_endnotes_](#endnotes)

---

<br>


## hello columnar

SQLite<sup>[2]</sup> is a simple row-based runtime. Its minimal configuration keeps Arrow front and center. Arrow is columnar. What does that mean? What does it mean in practice?

Setup for this section:
- _have a Python 3 installation_
- _configure and activate a Python virtual environment_
- `pip install --upgrade adbc-driver-sqlite adbc-driver-manager pyarrow`

First, we compare the results of a simple select-and-fetch of a single record.

```python
# hello_db.py
import sqlite3 as sqlite_rows
from adbc_driver_sqlite import dbapi as sqlite_arrow
#    ^ we'll get to adbc later

QUERY = """
SELECT
    'Hello, world!' as hello,
    1 as one,
    length('hi') as two
"""

def hello_db(db_api):
    with db_api.connect(":memory:") as connection:
        cursor = connection.cursor()
        try:
            cursor.execute(QUERY)
            row = cursor.fetchone()
            return row
        finally:
            cursor.close()

sqlite_res = hello_db(sqlite_rows)
arrow_res  = hello_db(sqlite_arrow)

print(f"Row-based SQLite:             {sqlite_res}")
print(f"Arrow Standard:               {arrow_res}")
```

What is the output of this script?

```shell
$ python hello_db.py
Row-based SQLite:             ('Hello, world!', 1, 2)
Arrow Standard:               ('Hello, world!', 1, 2)
```

We print the data values. The DB-API 2.0 SQLite driver gives a row. In Python, that row is a `Tuple`. Meanwhile, the Arrow standard hands us...a row?

Sometimes, types say more. Log more type information.

```python
print(f"Row-based SQLite row type:    {type(sqlite_res)}")
print(f"Arrow Standard row type:      {type(arrow_res)}")

print()
print("Row-based SQLite data types: ", [type(x) for x in sqlite_res])
print("Arrow Standard data types:   ", [type(x) for x in arrow_res])
```

This tells us almost nothing.<sup>[3]</sup>

```text
Row-based SQLite row type:    <class 'tuple'>
Arrow Standard row type:      <class 'tuple'>

Row-based SQLite data types:  [<class 'str'>, <class 'int'>, <class 'int'>]
Arrow Standard data types:    [<class 'str'>, <class 'int'>, <class 'int'>]
```

On the surface, Arrow looks no different. The values are the same. Their origins are not.

The standard sqlite3 implementation built a Python tuple from a database row. Arrow does something subtler.

This difference is clearer with multiple records.

### one row becomes two {#one-to-two}

First, add a second row to the result set.
```diff
QUERY = """
SELECT
    'Hello, world!' as hello,
    1 as one,
    length('hi') as two
+ union all
+ SELECT
+    'Hello, again!' as hello,
+    1+0 as one,
+    1 << 1 as two
"""
```

Second, fetch both rows (or more if they existed) as a `List[Row]`. This row list represents a database table.

```diff
-        row = cursor.fetchone()
-        return row
+        table = cursor.fetchall()
+        return table
```

Run it back.

```shell
$ python hello_db_redux.py
Row-based SQLite table:       [('Hello, world!', 1, 2), ('Hello, again!', 1, 2)]
Arrow Standard table:         [('Hello, world!', 1, 2), ('Hello, again!', 1, 2)]
```

Again, no visible difference. The solution is to use PyArrow's Arrow-specific functions.

```diff
def hello_db(db_api):
    with db_api.connect(":memory:") as connection:
        cursor = connection.cursor()
        try:
            cursor.execute(QUERY)
-           table = cursor.fetchall()
+           table = cursor.fetchallarrow()
            return table
        finally:
            cursor.close()

-sqlite_res = hello_db(sqlite_rows)
arrow_res  = hello_db(sqlite_arrow)

-print(f"Row-based SQLite table:       {sqlite_res}")
```

Third time's the charm.

```shell
Arrow Standard:   pyarrow.Table
hello: string
one: int64
two: int64
----
hello: [["Hello, world!","Hello, again!"]]
one: [[1,1]]
two: [[2,2]]
```

We are given the table type--now `pyarrow.Table`--along with the table's schema and a columnar view of its data.

Arrow stores each column in a contiguous, type-homogeneous array<sup>[4]</sup>. In our example, all fields of the `hello` column are packaged into a String array of length 2; the values of column `one` are contained in an Integer array of length 2 (likewise for column two). This is the tangible essence of "columnar."


| hello           | one | two  |
|-----------------|-----|------|
| Hello, world!   | 1   | 2    |
| Hello, again!   | 1   | 2    |

Row-based implementations group values together horizontally, in tuples.

```text
row-oriented layout (tuples of references in memory)

+---------------------------+
| ("Hello, world!", 1, 2)   |
+---------------------------+
| ("Hello, again!", 1, 2)   |
+---------------------------+
```

Columnar standards think in groups of columns.

```text
column-oriented layout (contiguous buffers in memory)

hello: ["Hello, world!", "Hello, again!"]
one:   [1, 1]
two:   [2, 2]
```

This is the what. The bytes tell us why.
<br>

## columns and taking data in stride {#section_2}

The following examples use this setup.


```python
# arrow_examples.py
from adbc_driver_sqlite import dbapi as sqlite_arrow

QUERY = """
SELECT
    'Hello, world!' as hello,
    1 as one,
    length('hi') as two
union all
SELECT
    'Hello, again!' as hello,
    1+0 as one,
    1 << 1 as two
"""

connection = sqlite_arrow.connect(":memory:")
cursor = connection.cursor()

cursor.execute(QUERY.strip())
table = cursor.fetchallarrow()
```

#### see table schema (type information) {#suba}
```python
print(table.schema)
```

A table's schema defines its structure. Part of that structure are columns, and each column has a type. In Arrow, types belong to a rich type system optimized for efficient cross-language representation.
```text
hello: string
one: int64
two: int64
```

We see two categories here: primitive scalars (e.g. `i64`) and variable-length types (e.g. `string`).

In creating a formal specification, Arrow gives users a logical _lingua franca_ and _carte blanche_ to optimize types in memory. Contiguous memory cells translate to vectorization (cf. faster loads). Boolean values are bit-packed (1 bit per value!!). I am personally inspired by the engineering in this layer alone.<sup>[5]</sup>

Note other types:
- structured data types (e.g. `list<T>`)
- temporal types (e.g. `date32`, `time64`)
- fixed width decimals
- a full suite of scalars including floats

#### grab an entire column {#subb}
```python
print(table["one"])  # pyarrow.ChunkedArray
```

`pyarrow` can obtain columns by name.
```
 [
  [
    1,
    1
  ]
]
```

This operation is zero-copy.<sup>[6]</sup> In Arrow, each column is stored in contiguous memory blocks. To "get the column," Arrow simply returns a new view pointing at the same buffers—no data are copied. So, why is the type logged above **ChunkedArray**?

Imagine we have an `int64` column. It might exist in memory as several chunks.
```text
ChunkedArray (logical column)
        |
        +-------------------------------+
        |           |           |       |
      Array       Array       Array   ...
     (len=3)     (len=2)     (len=4)

Array #1 → [1, 2, 3]
Array #2 → [4, 5]
Array #3 → [6, 7, 8, 9]

Logical column = [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

A **ChunkedArray** looks like one continuous column. In truth, it may be split into many chunks. This spec is performant by design. Most systems produce data in batches. Arrow keeps those batches in memory as they come, without copying, without merging. You see a single, seamless column.

Now, back to the code at the section heading.

```text
Arrow Array (view object)
+--------------------------------+
| type: int64                    |
| length: 2                      |
| offset: 0                      |
| buffers:                       |
|   [0] validity bitmap -------+ |
|   [1] values buffer pointer -+ |
+--------------------------------+

Elsewhere in memory:
values buffer (contiguous int64):
  01 00 00 00 00 00 00 00
  01 00 00 00 00 00 00 00
# logical view → [1, 1]
```
 The column `one` has only a single Array, so the **ChunkedArray** is just a thin wrapper around that one chunk.

An Arrow Array is not a literal array of values. It is a descriptor: buffers plus metadata like length, offset, and type. When you iterate or slice, Arrow does not have to touch or copy values. The underlying values stay in place. The buffers do not move.

Row-oriented systems are heavier. Each row's fields sit side by side in memory. To pull out a column, you must allocate a new buffer. Then, you walk every row, fetch the field, and copy it. With larger data you may need arrays of pointers. If the row count is unknown, the buffer may overflow and must grow. That means dynamic reallocations and more copying.<sup>[7]</sup>

#### grab a (one) row (table view) {#subc}
```python
print(table.slice(1))
```

Arrow is quite clever here.

```
hello: [["Hello, again!"]]
one: [[1]]
two: [[2]]
```

`Table.slice` returns another Table. Again, this is zero-copy. The Table returned by `table.slice(1)` is just a logical view. Precisely, the starting Table and the second obtained from slicing are collections of fat pointers (Arrow Array = metadata + buffer references). The total number matches the number of columns in the schema.

When dealing with database objects millions of records long, skipping the extra copy (or *copies*) to access a column means significant time and cost savings.

It also practices good systems hygiene: do not move or transform data until you must.


## records in batches {#section_3}

The abstractions so far stand on their own. They are fast and memory-thrifty. They also set the stage for how systems operate. Systems receive data in pieces. Results do not show up all at once. Arrow’s answer to this is the **RecordBatch**.

Earlier, we contrasted rows and columns:

| hello           | one | two  |
|-----------------|-----|------|
| Hello, world!   | 1   | 2    |
| Hello, again!   | 1   | 2    |


A **RecordBatch** is this block viewed columnar & a metadata schema. It is, in Rust parlance, a kind of fat pointer. It points to buffers of data and explains how they fit together as a table.

```text
RecordBatch
  schema: {hello: string, one: int64, two: int64}
  columns:
    hello → Arrow Array (length N)
    one   → Arrow Array (length N)
    two   → Arrow Array (length N)
  other_metadata
```

Like an Array, a RecordBatch does not own data. It points to column arrays and adds just enough metadata to bind them into a batch of rows.

A **Table** in Arrow is then a sequence of **RecordBatches**. Put another way, a Table is a logical view over several **Record Batches**. Each **RecordBatch** contributes one chunk per column. Put together, they form seamless **ChunkedArrays**:

In Arrow, a table is not a naive list of rows. It is a microcosm of abstractions to achieve.
```
Table (logical view)
   ├─ RecordBatch #1 (2 rows)
   │    hello → ["Hello, world!", "Hello, again!"]
   │    one   → [1, 1]
   │    two   → [2, 2]
   │
   └─ RecordBatch #2 (3 rows)
        hello → ["salve", "χαῖρε", "SALAVS"]
        one   → [1, 8, 9]
        two   → [14, 16, 18]

Logical column "one" = [1, 1, 1, 8, 9]
```

When two RecordBatches share the same schema, you can assemble them into one logical dataset.

```
import pyarrow as pa

# rb1, rb2: RecordBatch with identical schemas; zero-copy stitching
t = pa.Table.from_batches([rb1, rb2])
# optional: make each column a single contiguous chunk; requires a copy
t = t.combine_chunks()
```

Note the flexibility. RecordBatch gifts us that. It also makes Arrow streaming-ready (receive chunks asynchronously). And of course, it is analytics-ready, since we have the ability to treat several RecordBatches as one table.

<br>

## caveat lector! {#section_4}

In these examples, we used SQLite as the source engine. SQLite is still a row-based runtime. Arrow comes in only at the hand-off where the driver materializes results. Thus, our examples are _Arrow at the edge_, and not Arrow _all the way down_. The in-memory layout we explored is genuine Arrow. The engine is still building rows, however. Formats like ipc enable smooth columnar-native handoff of payloads between processes.

Arrow is also designed for interoperability. Any Arrow object in Python can be converted to rows for use in `Pandas`, `NumPy`, or similar. Those first `fetchone` prints--where an Arrow cursor returned a plain Python tuple--are examples of that. Arrow data lives in columnar buffers, but PyArrow makes it trivial to expose them as rows when needed.

The catch: conversion to rows means new Python objects. This suits debugging or small workloads. But, the real performance gains of Arrow come from staying columnar as long as possible.

<br />

## parting thoughts {#section_5}

This write-up is a foothold into the Arrow way of thinking. Arrow shines in columnar workloads. Rows are there when you need them, but its real strength is in columns, arrays, and batches. PyArrow makes that model approachable, and the best way to internalize it is to play with the API yourself.

Here, we stayed high-level: what a Table is, how Arrays and ChunkedArrays behave, and why RecordBatch is the core unit. In Part 2, we will switch to Rust and drop closer to the metal.

For now, the takeaway is this: Arrow at first looks familiar, but its engineering is informed by the hard realities of data at scale. This becomes only more clear the deeper we dive.

---
<br>

## _endnotes_

1. The traditional English gloss for *αἱ αἰτῐ́αι*.
1. Ordinary CPython distributions include the `sqlite3` module (via pysqlite).
1. There's a certain kind of engineer who will ask 'why not use `fetchall`.' But even then the returns will be the same, this time, `List` with a `Row` type.
1. type-homogeneous: all values in the collection are of a single, shared type; this is an oversimplification, as columns are stored as "contiguous buffers." We'll talk more about this in Part 2 in Rust.
1. Arrow `BooleanArray` [source](https://github.com/apache/arrow-rs/blob/d9a4b39815de52a15ca84b392a39fdf422361718/arrow-array/src/array/boolean_array.rs#L68) for those immediately curious. This is an accessible implementation detail that speaks to what Arrow does under the hood.
1. zero-copy: Zero-copy means accessing or transferring data without creating a new copy of the underlying bytes. I need to write a blog post that celebrates this property one day. Here in Python, views are zero-copy; materialization (e.g., converting to lists/dataframes) allocates.
1. There are mitigation strategies like overallocating upfront or maintaining row counts.
