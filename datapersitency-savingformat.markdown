---
layout: single
title: Datapersitency & Savingformat
permalink: /datapersitency-savingformat/

toc: true
toc_sticky: true
sidebar:
  nav: "docs"
---

## Overview
The Isenta DBMS saves all data page based in a single databasefile (.db).
The architecture is in 3 clearly separated layers:

| Layer | Description |
|---|---|
| StorageEngine | Lowlevel page-I/O (Read/Write) | 
| Database | Logical fromat: Header, tables, scheme, rows | 
| Write-Ahead Log | Prepared Crahs-recory layer | 


All persistent data gets stored in fixed pages (4096 bytes).

## Page based storage

### Page concept

```rust
pub const PAGE_SIZE: usize = 4096;
```

- Every page is **exactly** 4096 bytes
- Every psge has one logic Page-ID
- Page-ID -> fileoffset:

```rust
file_offset = page_id * PAGE_SIZE
```

### Page structure

```rust
pub struct Page {
    pub id: u64,
    pub data: [u8; 4096],
}
```

- `id`: logic page address 
- `data`: raw page-buffer
- Interpretation only comes from above layers!

### Reading pages

```rust
read_page(page_id)
```

**Process:**

1. Calculation of the offset:

    ```rust
    file_offset = page_id * PAGE_SIZE
    ```
2. If offset ≥ file size
   - Page doesn't exist yet
   - -> returns a zero-page
3. If the page exists:
   - Read up to 4096 bytes
   - Missing bytes get filled wit `0x00`

**Important note:**

> Not existing pages count as **empty**

That ensures:
- easy initialisation
- defensive reparing
- no panics at missing pages

### Writing pages

```rust
write_page(&Page)
```

- Jumps to the page-offset
- Overwrites exactly 4096 bytes
- No partwrites
- No knowledge about schemas or data

**Atomacity note:**

> A page write is **not** atomic! -> that is what the WAL is for.

### Allocation of new pages

```rust
allocate_page()
```

- New page-ID = `file_size / PAGE_SIZE`
- New page gets:
    - initialized with zeros
    - directly written at the end of the file

-> The system is append-only

## Database fileformat

### Page 0: Header page

The first page of the file (page 0) is always the header page.

**Layout of the header page:**

| Offset | Size | Meaning | 
|---|---|---|
| 0-7 | u64 | Magic number (`ISENTADB`) |
| 8-11 | u32 | Database version |
| 12-19 | u64 | Schema-root-page-id |
| 20-23 | u32 | Table count |
| 24-4095 | - | Reserved |

**Magic number:**

```rust
0x4953454E54414442 // "ISENTADB"
```

Only for recognition of the database file validity.

### Initialization of a new database

When opening the database, the following steps are taken:

-   Check file size.
-   If the file is empty:
    -   The Header Page is initialized.
    -   `schema_root` is set to `0`.
    -   `table_count` is set to `0`.
-   If the file exists:
    -   Check the Magic Number.
    -   A warning is issued about possible corruption.
    -   There is no automatic overwriting.

## Schema storage (table definitions)

### Schema as a linked list

All table schemas are stored as a singly linked list of Schema Pages.

```
Header.schema_root
    ↓
[Schema Page 1] → [Schema Page 2] → [Schema Page 3] → NULL
```

-   Each Schema Page describes exactly one table.
-   The order is the order of creation.

### Layout of a Schema Page

The structure is as follows (in order):

1.  Table name
2.  Number of columns
3.  Column definitions
4.  Data-Page-ID
5.  Next-Schema-Page-ID

### Table name

| Field       | Type        |
| :---------- | :---------- |
| Name-Length | u32         |
| Name        | UTF-8 Bytes |

### Columns

| Field             | Type                |
| :---------------- | :------------------ |
| Number of columns | u32                 |
| **For each column:** |                     |
| Name length       | u32                 |
| Name              | UTF-8               |
| Type length       | u32                 |
| Type              | UTF-8 (e.g. INT, TEXT) |

### Data Page Pointer

| Field              | Type |
| :----------------- | :--- |
| First Data Page ID | u64  |

-   Points to the first page with row data.
-   `0` = table empty.

### Chaining

| Field               | Type |
| :------------------ | :--- |
| Next Schema Page ID | u64  |

-   `0` = end of schema list.

## Storage of table data (Rows)

### Data as page chains

Each table has its own chain of Data Pages:

```
Schema.data_page_id
    ↓
[Data Page 1] → [Data Page 2] → NULL
```

### Layout of a Data Page

Structure:

1.  Number of rows
2.  Row data
3.  Next-Data-Page-ID

### Row Count

| Field          | Type |
| :------------- | :--- |
| Number of rows | u32  |

### Row-Encoding

Each row is stored column by column. For each value:

| Type | Meaning       |
| :--- | :------------ |
| 0x00 | NULL          |
| 0x01 | Integer (i64) |
| 0x02 | Text          |

-   **NULL**: `[TYPE_NULL]`, No payload. Currently represented as an empty string.
-   **INT**: `[TYPE_INT][8 Byte i64]`
-   **TEXT**: `[TYPE_TEXT][u32 length][Bytes]`

### Next Data Page Pointer

| Field        | Type |
| :----------- | :--- |
| Next Page ID | u64  |

-   `0` = last page.

### Page Fill Strategy

Rows are written until:

-   no more space is available.
-   Remaining rows → new Data Page.
-   Pages are append-only.
-   No compression or reorganization.

## Write-Ahead Log (wal.rs)

### Purpose of the WAL

The Write-Ahead Log serves as preparation for crash recovery.

Current state:

-   Logging of page changes.
-   No automatic recovery yet.

### WAL file

-   Separate file
-   Append-only
-   Sequential writing

### WAL Record Format

```rust
pub struct WalRecord {
    pub page_id: u64,
    pub offset: u64,
    pub length: u64,
    pub data: Vec<u8>,
}
```

Binary layout:

| Field   | Size    |
| :------ | :------ |
| Page ID | 8 Bytes |
| Offset  | 8 Bytes |
| Length  | 8 Bytes |
| Payload | n Bytes |

### Writing a WAL Record

1.  Write Page-ID
2.  Write offset
3.  Write length
4.  Write data
5.  call flush()

This guarantees persistence of the log.

### Reading the WAL

-   The WAL is read completely.
-   Records are reconstructed sequentially.
-   Invalid or truncated records are ignored.

### Current status of the WAL

**Important**:

-   WAL is not yet coupled with storage.
-   No automatic recovery.
-   No commit markers.
-   Currently a redo-log precursor.

## Consistency & repair mechanisms

### Schema repair

When loading:

-   Circular schema chains are detected.
-   `table_count` is corrected.
-   Invalid pages are ignored.

### Defensive Null-Pages

-   Empty pages = valid state.
-   Prevents crashes on corruption.

## Design decisions & limits

**Deliberate restrictions:**

-   No free list
-   No transactions
-   No real WAL recovery
-   No page checksums
-   No parallel access

**Advantages:**

-   Simple
-   Robust
-   Very extensible

## Summary

The SentinelIS DBMS uses:

-   Page-based storage
-   Chained schema pages
-   Typed row encoding
-   Append-only Data Pages
-   Prepared WAL structure

The architecture is clearly separated, defensive and realistic for a real DB engine.