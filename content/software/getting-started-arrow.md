+++
title = "A first foray into Arrow (Part 1: Python)"
date = 2025-09-07
updated = 2025-09-07
description = "A wandering through what Arrow is with real code"
draft = false
+++


If we borrow a page from Aristotle, to learn a thing--to understand its causes<sup>1</sup>--we must first accumulate practical experience with it and hone our craft; only after can we analyze effectively.

This post is the first in a short series. We begin with Arrow in Python. In upcoming parts, we’ll look at Arrow in Rust, take a brief historical detour, and close by drawing out some of the key invariants of the specification.

<br>

---

1. [_a look at_ pyarrow](#a-look-at-pyarrow)
  - [_columns and taking data in stride_](#suba)
    - [_example: table schema_](#subaa)
    - [_example: columns_](#subab)
    - [_example: rows_](#subac)
1. [_endnotes_](#endnotes)

---

<br>


## _a look at_ pyarrow

We use SQLite<sup>2</sup> to begin. A minimal configuration surface protects our focus.

Setup for this section:
- _have a Python 3 installation accessible_
- _configure and activate a Python virtual environment_
- `pip install --upgrade adbc-driver-sqlite adbc-driver-manager pyarrow`

First, we can compare the results of a simple select and fetch of a single record.

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
    try:
        connection = db_api.connect(":memory:")
        cursor = connection.cursor()
        cursor.execute(QUERY.strip())
        record = cursor.fetchone()
        return record
    finally:
        cursor.close()
        connection.close()

res1 = hello_db(sqlite_rows)
res2 = hello_db(sqlite_arrow)

print(f"Row-based SQLite: {res1}")
print(f"Arrow Standard:   {res2}")
```

What is the output of this script?

```shell
$ python hello_db.py
Row-based SQLite: ('Hello, world!', 1, 2)
Arrow Standard:   ('Hello, world!', 1, 2)
```

The standard DB-API 2.0 implementation of SQLite hands us a row. Meanwhile, the Arrow standard hands us...a row?

At first glance, both APIs hand us the same row. They are indeed the same. Their origin is different, however. The standard sqlite3 implementation built a Python tuple from a database row. Arrow is doing something subtler.

The difference becomes clear with multiple records.

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

Second, fetch both rows (or more if they existed).

```diff
-        record = cursor.fetchone()
-        return record
+        records = cursor.fetchall()
+        return records
```

Rerun the script.

```shell
$ python hello_db.py
Row-based SQLite: [('Hello, world!', 1, 2), ('Hello, again!', 1, 2)]
Arrow Standard:   [('Hello, world!', 1, 2), ('Hello, again!', 1, 2)]
```

No visible difference. You have not been misled. We need a different method.

```diff
def hello_db(db_api):
    try:
        connection = db_api.connect(":memory:")
        cursor = connection.cursor()
        cursor.execute(QUERY.strip())
-       records = cursor.fetchall()
+       table = cursor.fetchallarrow() # returns a pyarrow.Table
-       return records
+       return table
    finally:
        cursor.close()
        connection.close()

- res1 = hello_db(sqlite_rows)
res2 = hello_db(sqlite_arrow)

- print(f"Row-based SQLite: {res1}")
```

Rerun this latest version.

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

These are the same data as before, but now we see they do not have an identical memory layout. Arrow stores each column in a contiguous, type-homogeneous array<sup>3</sup>. A Table is a logical view that can span multiple **RecordBatch**es. In our example, all fields of the `hello` column are packaged into a String array of length 2; the values of column `one` are contained in an Integer array of length 2 (likewise for column two). Together, these equal-length arrays form a **RecordBatch**. This is the tangible essence of "columnar."

Whereas row-based standards think in records (often tuples of values), Arrow’s fundamental unit is the **RecordBatch**.

| hello           | one | two  |
|-----------------|-----|------|
| Hello, world!   | 1   | 2    |
| Hello, again!   | 1   | 2    |

Simply put, row-based implementations group values together horizontally, in tuples. Columnar standards think in groups of columns. These groups of columns are **RecordBatch**es.


```text
row-oriented layout (tuples in memory)

+---------------------------+
| ("Hello, world!", 1, 2)   |
+---------------------------+
| ("Hello, again!", 1, 2)   |
+---------------------------+
```

Values for different columns are interleaved row by row.


```text
column-oriented layout (contiguous buffers in memory)

hello: ["Hello, world!", "Hello, again!"]
one:   [1, 1]
two:   [2, 2]
```

Each column is stored in its own contiguous buffer.


### columns and taking data in stride {#suba}

This is the what. Dive into bytes to reveal the why.

```python
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

The following examples use this setup.

##### see table schema (type information) {#subaa}
```python
print(table.schema)
```

A table's schema defines its structure: the set of columns, their names, data types, and metadata.
```text
hello: string
one: int64
two: int64
```

##### grab an entire column {#subab}
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

This operation is zero-copy.<sup>4</sup> When your data are columnar (Arrow), each column is already stored as one contiguous array<sup>3b</sup>. To "get the column," you need only return a new view pointing at the same buffer. In Arrow, a column of a Table is a **ChunkedArray** (one or more **Array** chunks). Here it’s a single chunk.

```
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

An Arrow Array is not a true array of values; it is a descriptor (buffer references plus metadata (length, offset, type, etc.). When you iterate or slice the data, Arrow adjusts the metadata of this object without changing the underlying column values. The column values remain unchanged in their shared memory buffers.

In a row-oriented database engine, row fields sit next to each other in memory. To extract a column, you must first allocate a new buffer, then walk each row, access the field, and copy it into that buffer (for larger data, you would build arrays of pointers into those fields). In cases when row count is not known, your target buffer may be too small; at least one dynamic reallocation will occur.<sup>5</sup>

This example teaches us something else: A RecordBatch is also a fat pointer of sorts.

```text
RecordBatch
  schema: {hello: string, one: int64, two: int64}
  columns:
    hello → Arrow Array (length N)
    one   → Arrow Array (length N)
    two   → Arrow Array (length N)
  other_metadata
```

#### grab a row {#subac}
```python
print(table.slice(1))
```

Arrow is quite clever here.

```
hello: [["Hello, again!"]]
one: [[1]]
two: [[2]]
```

Here, Arrow returns another Table. Again, this is a zero-copy operation. Arrow does not copy column values. The Table returned by `table.slice(1)` is just a logical view. Precisely, the starting Table and the second obtained from slicing are collections of fat pointers (Arrow Array = metadata + buffer references). The total number matches the number of columns in the schema.

When dealing with database objects millions of records long, skipping the extra copy (or *copies*) to access a column means significant time and cost savings.

It also practices good systems hygiene: do not move or transform data until you must.


---
<br>

## _endnotes_

1. The traditional English gloss for *αἱ αἰτῐ́αι*.
1. Ordinary CPython distributions include the `sqlite3` module (via pysqlite).
1. type-homogeneous: all values in the collection are of a single, shared type; this is an oversimplification, as columns are stored as contiguous buffers.
1. zero-copy: Zero-copy means accessing or transferring data without creating a new copy of the underlying bytes. I need to write a blog post that celebrates this property one day.
1. There are mitigation strategies like overallocating upfront or maintaining row counts.
