Sure. Let’s walk through an **end-to-end upload of a 1 GB file** in the Google Drive style design from the page.

The key idea is that the file is **not uploaded as one giant blob**. It is split into smaller chunks, each chunk is uploaded independently, and the backend stores only the changed blocks plus metadata.

## Big picture

For a 1 GB file, the client usually:

1. asks the server to start an upload,
2. gets back an `upload_id` and a chunk size,
3. splits the file into chunks,
4. uploads the chunks one by one or in parallel,
5. tells the server the upload is complete,
6. the server stores file metadata and makes the file visible in Drive.

The page recommends **4 MB chunks**, so let’s use that.

### How many chunks for 1 GB?

1 GB = 1024 MB  
Chunk size = 4 MB

So:

```text
1024 / 4 = 256 chunks
```

So your 1 GB file becomes **256 chunks**.

---

## End-to-end flow with an example

Imagine Alice uploads a file called `holiday-video.mp4`, size = **1 GB**.

### 1) Client starts the upload

Alice clicks Upload in the Drive app.

The client sends a request like:

```json
POST /files/upload
{
  "file_name": "holiday-video.mp4",
  "file_size": 1073741824,
  "parent_folder_id": "folder_abc123",
  "checksum": "file_sha256_hash"
}
```

### What the server does here

The API server:

- checks Alice’s authentication
- checks quota
- validates the request
- creates an **upload session**
- returns an `upload_id`
- tells the client the chunk size

Response:

```json
{
  "upload_id": "up_xyz789",
  "upload_url": "https://storage.example.com/upload/up_xyz789",
  "chunk_size": 4194304,
  "total_chunks": 256
}
```

At this point, the file is **not yet visible** in Drive as a completed file. It is just an active upload session.

---

### 2) Client splits the file into chunks

The client reads the file from disk and splits it into 4 MB blocks:

- Chunk 1
- Chunk 2
- Chunk 3
- ...
- Chunk 256

Each chunk is hashed locally. The hash is used to verify integrity and possibly deduplicate blocks.

For example:

- Chunk 1 hash = `hash_a1`
- Chunk 2 hash = `hash_b2`
- Chunk 3 hash = `hash_c3`

If the same chunk already exists in storage, the system can avoid storing it again.

---

### 3) Client uploads chunks

The client uploads chunks to the block upload service or to storage through a pre-signed URL, depending on implementation.

A chunk upload might look like:

```http
PUT /files/upload/up_xyz789/chunk/1
Content-MD5: ...
```

The body contains 4 MB of binary data.

This happens for all 256 chunks. Often the client uploads several chunks in parallel to speed things up.

### What happens per chunk on the server

For each chunk:

- server receives chunk data
- server computes checksum
- server verifies it matches what the client sent
- server stores the chunk in object storage
- server marks that chunk as received for this `upload_id`

If chunk 17 fails because of a network problem, only chunk 17 is retried, not the whole 1 GB file.

---

### 4) Temporary state is tracked during upload

The backend keeps an upload session record like:

```json
{
  "upload_id": "up_xyz789",
  "file_name": "holiday-video.mp4",
  "total_chunks": 256,
  "received_chunks": [1, 2, 3, 4, 5, ...],
  "status": "in_progress"
}
```

This is what makes the upload **resumable**.

If Alice loses internet after chunk 180, the app can later ask:

```http
GET /files/upload/up_xyz789/status
```

And get back something like:

```json
{
  "received_chunks": [1, 2, 3, ..., 180],
  "missing_chunks": [181, 182, ..., 256]
}
```

Then it resumes only the missing chunks.

---

### 5) Client completes the upload

Once all 256 chunks are uploaded, the client calls:

```http
POST /files/upload/up_xyz789/complete
```

This tells the server, “all chunks are here, please finalize the file.”

### What the server does now

The server:

- verifies all chunks arrived
- verifies their hashes
- creates the final file record in the metadata database
- stores the ordered list of block hashes
- assigns a `file_id`
- sets the file status to complete

Example metadata saved:

```json
{
  "file_id": "file_abc123",
  "name": "holiday-video.mp4",
  "owner_id": "user_123",
  "size": 1073741824,
  "version": 1,
  "block_list": ["hash_1", "hash_2", "...", "hash_256"]
}
```

Now the file becomes visible in Alice’s Drive.

---

## Why this design is good

### Resumable uploads
If the network fails halfway, the user does not restart from zero. Only missing chunks are retried.

### Lower failure risk
A single 1 GB request is fragile. 256 smaller chunk uploads are much more reliable.

### Parallelism
Chunks can be uploaded at the same time, which improves speed.

### Deduplication
If some chunks already exist, they do not need to be stored again.

### Atomic completion
The file appears only after all chunks are uploaded, so users do not see half-finished files.

---

## What happens if upload is interrupted?

Suppose Alice reaches chunk 180, then her laptop sleeps.

Later, when she reopens the app:

1. The client asks the server for upload status.
2. The server replies with received and missing chunks.
3. The client uploads only chunks 181 to 256.
4. The client sends `complete`.
5. The file is finalized.

So the file upload can survive failures without wasting all the progress.

---

## End-to-end summary in one flow

```text
User selects 1 GB file
→ Client sends upload init request
→ Server returns upload_id + chunk_size
→ Client splits file into 256 chunks of 4 MB each
→ Client uploads chunks, possibly in parallel
→ Server stores each chunk and tracks progress
→ If connection fails, client resumes missing chunks
→ Client sends complete request
→ Server writes file metadata and block list
→ File becomes available in Drive
```

---

## A quick mental model

Think of it like shipping a huge book as **256 numbered pages in separate envelopes** instead of one giant box.

If envelope 173 gets lost, you resend only envelope 173, not the whole book.

---

If you want, I can also show this as a **sequence diagram** or explain the **same 1 GB upload with exact API calls and database writes step by step**.



##
This sentence is describing the **metadata layer** of the storage system, not the file bytes themselves.

## What it means

When a user uploads a file, the system splits that file into blocks, for example 4 MB chunks. After all chunks are uploaded, the API server writes a **file record** into the metadata database. That record contains:

- the file name
- owner
- folder path
- size
- permissions
- version info
- and, most importantly, the **ordered list of block hashes**

That ordered list is the recipe for rebuilding the file later.

---

## Why the order matters

A file is not just a bag of chunks. The chunks must be reassembled in the correct sequence.

For example, suppose a file is split into 3 blocks:

- Block A
- Block B
- Block C

The metadata record stores something like:

```json
{
  "file_id": "file_123",
  "name": "report.pdf",
  "block_hashes": ["hash_A", "hash_B", "hash_C"]
}
```

When someone downloads the file, the system reads this list in order:

1. fetch block `hash_A`
2. fetch block `hash_B`
3. fetch block `hash_C`
4. concatenate them

That produces the original file again.

If the order were wrong, the file would be corrupted.

---

## End-to-end example

Let’s say Alice uploads a 1 GB video.

The client splits it into 4 MB blocks. That gives 256 blocks.

After upload, the system stores metadata like this:

```json
{
  "file_id": "file_987",
  "owner_id": "alice",
  "name": "holiday.mp4",
  "size": 1073741824,
  "block_hashes": [
    "h1",
    "h2",
    "h3",
    "...",
    "h256"
  ]
}
```

Each hash points to a block stored in object storage.

Later, when Bob downloads the file, the server does not search for the whole 1 GB blob. Instead, it:

1. looks up `file_987` in the metadata DB
2. gets the ordered block list
3. retrieves each block from storage by hash
4. joins them in order
5. returns the reconstructed file

So the metadata DB acts like a **map** from file ID to the exact pieces needed to rebuild the file.

---

## Why store hashes instead of raw block pointers?

Using hashes gives several benefits:

### 1. Deduplication
If another file already contains the same block, the system can reuse it instead of storing it again.

### 2. Integrity checking
The hash can also verify that the block contents have not changed or become corrupted.

### 3. Content-addressable storage
The block can be fetched by its content identity, not just by a random location.

---

## Why not store the whole file in metadata DB?

Because the metadata database is meant for **small, frequently accessed information**, not huge file contents.

Storing the full file there would be:

- expensive
- slow
- hard to scale
- bad for large files like videos or archives

So the design splits responsibility:

- **Metadata DB** stores the structure and pointers
- **Object storage** stores the actual bytes

---

## Simple analogy

Think of a book stored in a library:

- The **metadata record** is the catalog card that says:
  - title
  - author
  - shelf location of each volume
  - order of the volumes
- The **blocks** are the pages or volumes themselves

If you want to read the book later, you use the catalog card to collect the pages in the right order.

---

## In one sentence

That text means the system stores a file as a **sequence of block references in metadata**, and later reconstructs the file by fetching those blocks in that exact order and concatenating them.

If you want, I can also show how the **database schema** for `files`, `file_versions`, and `version_blocks` works together for this.

##

Great question. There are **two separate concerns** here:

1. **How the order of blocks is decided**
2. **What happens if the upload is incomplete or corrupted**

## 1) Is the order decided by chunk number or arrival time?

It should be decided by **chunk number / block index**, not arrival time.

### Why not arrival time?
Chunks can arrive out of order because:

- the client uploads in parallel
- retries happen
- network delays differ
- some chunks fail and are resent later

If the server used arrival time, the reconstructed file could end up scrambled.

### Correct approach
The client assigns each chunk a stable position:

- chunk 0
- chunk 1
- chunk 2
- ...

The metadata stores the block hashes in that exact order.

Example:

```json
{
  "file_id": "file_123",
  "block_hashes": ["h0", "h1", "h2", "h3"]
}
```

Even if chunk 3 arrives before chunk 1, the final file is still reconstructed as:

```text
chunk 0 + chunk 1 + chunk 2 + chunk 3
```

So the order is based on the **logical chunk index**, not upload arrival order.

---

## 2) What if some chunk has not arrived yet?

Then the file should **not be finalized or shown as complete**.

There are usually two stages:

### Stage A: Upload in progress
The file is still an **upload session**.  
The system may track:

- received chunks
- missing chunks
- checksums
- upload status

But it should **not** appear as a finished file in the user’s Drive.

### Stage B: Upload complete
Only after all chunks are present and verified does the API server create the final file record in the metadata database.

That is the moment when the file becomes visible as a normal file.

So if chunk 17 is missing, the system keeps the upload as **incomplete**, not downloadable as a regular file.

---

## 3) What if a chunk is corrupted?

Then the server rejects that chunk.

### How corruption is detected
Each chunk usually has a checksum or hash, for example SHA-256 or MD5.

When the chunk arrives, the server computes the hash of the received bytes and compares it with the expected value.

- If it matches, store the chunk.
- If it does not match, reject it and ask the client to resend.

### Important point
A corrupted chunk should not be counted as valid, so the file still remains incomplete until a correct version arrives.

---

## 4) Will the user see the file while it is incomplete?

Usually **no**, at least not as a normal downloadable file.

The common behavior is:

- **Upload session exists**, but is invisible or marked as “uploading”
- **Only when all chunks are received**, the system commits the file metadata
- Then the file appears in the user’s file list

This atomic publish behavior avoids a bad experience where the user sees a half-uploaded file.

---

## 5) What if the user refreshes the page or switches devices?

The system can show the upload as:

- uploading
- paused
- failed
- resumable

But it should not expose it as a fully available file until completion.

The backend can keep temporary session state like:

```json
{
  "upload_id": "up_123",
  "status": "in_progress",
  "received_chunks": [0, 1, 2, 4],
  "missing_chunks": [3]
}
```

Once chunk 3 arrives and all chunks are verified, the system creates the final file record.

---

## 6) Why this is safer

This design prevents:

- scrambled file reconstruction
- partial files appearing as complete
- corrupted uploads becoming visible
- duplicate or out-of-order retries causing inconsistency

---

## Short answer

- **Order is determined by chunk number / block index, not arrival time.**
- **Missing or corrupted chunks keep the upload incomplete.**
- **The file should not be shown as a completed downloadable file until all chunks are received and verified.**

If you want, I can draw the **exact upload state machine** for this, like `in_progress -> incomplete -> complete -> visible`.


##

Good question. The key point is: **the client does not invent a new chunk size during modification**. It usually uses the **same chunking scheme** that was used before, so the file can be compared consistently.

## 1) How does the client decide chunk size on modification?

Usually one of these happens:

### Option A: Fixed chunk size
The system uses a fixed size, like **4 MB**, for all versions of the file.

So if `document.docx` was previously stored as:

- chunk 0
- chunk 1
- chunk 2
- chunk 3

then after modification, the client re-splits the new file using the **same 4 MB chunk size** and compares chunk by chunk.

This is the simplest approach, and it is what the page is describing.

### Option B: Content-defined chunking
For better delta sync, the client can use content-defined boundaries. In that case, the chunk size is not strictly fixed, but the system still uses the **same chunking algorithm** every time.

The client does not decide a different size each time. It follows the same rule so identical or mostly identical content produces mostly identical chunk hashes.

---

## 2) What if a new chunk appears that was not in the previous version?

That is completely normal.

A modified file is not required to have the exact same chunk list as the previous version. The new version might have:

- the same chunks in the same order
- some changed chunks
- extra chunks
- fewer chunks

The server does **not** try to “guess” where the new chunk belongs based on upload arrival time.

Instead, the client sends the **new block list in logical order**.

For example:

### Old version
```text
[file_v1] = [A, B, C]
```

### New version after edit
```text
[file_v2] = [A, X, C, D]
```

Here:

- `A` stayed the same
- `B` changed into `X`
- `D` is a new extra chunk

The client tells the server the new ordered list:

```json
["hash_A", "hash_X", "hash_C", "hash_D"]
```

The server stores this as a **new version** of the file.

It does **not** overwrite the old version in place.

---

## 3) How does the server know where to put the new chunk?

This is the most important part.

The server knows the order from the **chunk position / block index** provided by the client, or from the **ordered list of hashes** in the upload completion request.

For example, the client may upload:

- chunk 0
- chunk 1
- chunk 2
- chunk 3

or it may upload block references like:

```json
{
  "file_id": "file_123",
  "version": 2,
  "blocks": [
    { "index": 0, "hash": "hash_A" },
    { "index": 1, "hash": "hash_X" },
    { "index": 2, "hash": "hash_C" },
    { "index": 3, "hash": "hash_D" }
  ]
}
```

So the server does not infer order from time.  
It stores the block list exactly as an ordered sequence.

---

## 4) What if only one new chunk is inserted in the middle?

Then the new file version might look like:

### Before
```text
[A, B, C]
```

### After edit
```text
[A, B, X, C]
```

The client uploads only `X` if `A`, `B`, and `C` are unchanged and already exist.

The server creates a new version record:

```text
version 1 -> [A, B, C]
version 2 -> [A, B, X, C]
```

The old version stays intact for history or rollback.

---

## 5) Why this works well

This design lets the system:

- preserve old versions
- reconstruct any version later
- upload only changed pieces
- avoid depending on arrival order
- support retries safely

---

## 6) Important distinction: upload chunks vs file version blocks

There are really two related but different things:

### Upload chunks
These are transport pieces used to send data reliably over the network.

Example:

- file is 1 GB
- split into 4 MB upload chunks
- upload chunk 1, 2, 3...

### File version blocks
These are the logical pieces that define the file content and order.

Example:

- version 2 = `[hash_A, hash_X, hash_C]`

A new upload chunk may become one logical block in the file version, but the server stores the final file using the **logical order**, not the network arrival order.

---

## 7) Short answer

- The client usually uses the **same chunking strategy** as before, not a new one every time.
- A new or extra chunk is treated as part of the **new file version**.
- The server knows where it belongs from the **chunk index / ordered block list**, not from when it arrived.
- The server stores a **new version**, it does not modify the old one in place.

If you want, I can show this with a **fully worked example of a 10 MB document before and after a small edit**, including the exact chunk lists.


##

Great question. The key thing to understand is:

**The cursor is not based on where inside the file the user edited.**  
It is based on **the order of sync events in the system’s change log**.

So even if the user changes the middle of a file, the cursor still just means:

> “This device has processed changes up to event number N.”

It does **not** mean “the edit happened at byte 500” or “the edit was in the middle of the file.”

---

# 1) Two different ideas: file position vs sync cursor

These are easy to mix up.

### A. File position
This is where the edit happened inside the file.

Example:

- changed bytes 4 MB to 8 MB
- changed block 2 out of 3
- inserted text in the middle of paragraph 5

### B. Sync cursor
This is the position in the **change log**, not the file.

Example:

- event 120
- event 121
- event 122

The cursor is about **which changes the device has seen**, not where the change happened in the file.

---

# 2) Example: editing the middle of a file

Suppose `document.docx` is split into 3 blocks:

```text
Version 1 = [A, B, C]
```

Now Alice changes something in the middle block only.

So the new version becomes:

```text
Version 2 = [A, X, C]
```

Here:

- `A` unchanged
- `B` changed to `X`
- `C` unchanged

The Sync Service records this as a **new change event**.

---

## Change log example

Let’s say the global change log looks like this:

- 200 = `photo.jpg` created
- 201 = `document.docx` modified
- 202 = `notes.txt` deleted

The change to the middle of `document.docx` is just **event 201**.

That event might internally describe:

- file id
- version number
- changed blocks
- maybe old version and new version references

But the cursor itself is simply:

- after processing event 200, cursor = 200
- after processing event 201, cursor = 201

---

# 3) What exactly is stored in the cursor?

Usually the cursor is just an **opaque token** or a **sequence number**.

For example:

```json
{
  "cursor": "201"
}
```

or

```json
{
  "cursor": "cur_9f82ab"
}
```

The client does not need to understand how it is built. It just stores it and sends it back later.

---

# 4) End-to-end example

## Step 1: Device B is synced up to event 200
Device B’s last cursor is 200.

```json
{
  "cursor": 200
}
```

## Step 2: Alice edits the middle of the file on Device A
The client detects the changed block.

Old file:

```text
[A, B, C]
```

New file:

```text
[A, X, C]
```

## Step 3: Sync Service records a new event
It appends this to the change log:

```text
201: document.docx changed
```

Now the change log is:

- 200
- 201
- 202
- ...

## Step 4: Device B asks for changes since 200
Device B sends:

```http
GET /sync/changes?cursor=200
```

## Step 5: Server responds
The server returns:

```json
{
  "changes": [
    {
      "position": 201,
      "file_id": "document.docx",
      "change_type": "modified",
      "changed_blocks": [1]
    }
  ],
  "cursor": 201
}
```

Now Device B updates its saved cursor to 201.

---

# 5) What if the change is in the middle but also causes extra chunks?

This is very common.

Suppose the file is block-based and Alice inserts text in the middle.

### Before edit
```text
[A, B, C]
```

### After edit
```text
[A, X, C]
```

or if the edit shifts boundaries:

```text
[A, X, Y, C]
```

The sync system still creates **one change event** for that upload or edit operation, even though internally multiple blocks may change.

The cursor still advances by **event order**, not by file offset.

So the change log might say:

- event 301: `document.docx` updated from version 7 to version 8

That one event may reference multiple changed blocks.

---

# 6) Does the cursor ever depend on the file contents?

No, not directly.

The cursor is usually assigned by the sync backend when the change is committed.

A common implementation is:

- the database or log system assigns an increasing sequence number
- or the event stream system assigns an offset
- or the server generates an opaque token tied to the log position

The file contents only determine **what changed**, not the cursor value itself.

---

# 7) Why this design is good

This separation is important because the system can now answer two different questions:

### “What changed in the file?”
That is answered by block hashes, version records, and changed block lists.

### “What changes has this device already seen?”
That is answered by the cursor.

So even if one edit changes a middle block, the cursor still works cleanly.

---

# 8) Simple analogy

Imagine a newspaper delivery system.

- The **file edit** is like a story being updated on page 8.
- The **cursor** is like saying which newspaper edition you last read.

If you last read edition 100, the next one you need is edition 101.

The fact that the story changed on page 8 does not affect the edition number.

---

# Final answer

If the user changes the middle of a file, the **cursor does not point to the middle of the file**.  
The cursor is assigned to the **sync event** in the change log.

### So:
- file position = where inside the file the bytes changed
- cursor = which change event the device has already processed

### Example:
- Device B last saw cursor `200`
- Alice edits the middle of `document.docx`
- Sync Service records event `201`
- Device B asks for changes since `200`
- Server returns event `201`
- Device B updates cursor to `201`

If you want, I can next explain **how the server generates the change log position in a distributed system** and what happens when there are **multiple sync servers**.

##

Sure. Let’s explain **how content-defined chunking works**, and how a block is decided.

## Big idea

Instead of cutting a file into blocks at fixed byte counts, the system scans the file and **chooses chunk boundaries based on the content**.

That means the boundary is not at “every 4 MB exactly.”  
It is at a place where the data “looks right” according to a rolling hash rule.

---

# 1) Fixed-size chunking first

Suppose you have this data:

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

If we use fixed-size chunking with size 5, we split like this:

```text
ABCDE | FGHIJ | KLMNO | PQRST | UVWXY | Z
```

These boundaries are purely based on position.

If you insert `X` at the start:

```text
XABCDEFGHIJKLMNOPQRSTUVWXYZ
```

Now the chunks become:

```text
XABCD | EFGHI | JKLMN | OPQRS | TUVWX | YZ
```

Everything shifted. That is why fixed-size chunking is bad for inserts near the start.

---

# 2) Content-defined chunking idea

With content-defined chunking, the system scans byte by byte and looks for a pattern in a **rolling hash**.

When the rolling hash matches a rule, it says:

> “Okay, end the chunk here.”

So chunk boundaries depend on the **content**, not the absolute position.

---

# 3) A simple example of boundary selection

Imagine we scan bytes and compute a rolling hash over the last few bytes.

The system says:

- keep extending the chunk
- when the rolling hash ends in `0000` in binary, cut a chunk

This is just an example rule. Real systems use more sophisticated rules, but the idea is the same.

### Example stream

Suppose we scan:

```text
A B C D E F G H I J K L M N O P ...
```

As we move forward, the rolling hash changes.

At some point, after `E`, the hash matches the boundary rule, so the first chunk becomes:

```text
ABCDE
```

Then scanning continues from `F`.

At some later point, after `J`, the hash matches again:

```text
FGHIJ
```

And so on.

So the chunk sizes become variable:

- 5 bytes
- 5 bytes
- 6 bytes
- 4 bytes
- etc.

---

# 4) Why this helps with inserts

Suppose the original file is:

```text
HELLO_WORLD_THIS_IS_A_FILE
```

CDC may split it at content-based points like:

```text
HELLO_ | WORLD_ | THIS_IS_ | A_FILE
```

Now suppose you insert `NEW_` near the beginning:

```text
HELLO_NEW_WORLD_THIS_IS_A_FILE
```

With fixed-size chunking, everything after the insertion shifts.

With CDC, the rolling hash keeps scanning content. The insertion may affect the chunk boundary near that location, but later parts may still produce the same boundaries, so many chunks stay unchanged.

That is why CDC is good for sync and deduplication.

---

# 5) Very small practical example

Let’s use a toy rule.

### Rule
Cut a chunk whenever the last 2 bytes are `AA`.

### File 1
```text
P Q R A A S T A A U V
```

The chunks might be:

```text
PQRAA | STA A | UV
```

More realistically, the scanner keeps moving until it sees `AA`.

Now insert a byte near the beginning:

```text
X P Q R A A S T A A U V
```

The first boundary may shift a little, but the later `AA` patterns still exist, so the chunks after that may remain similar.

That is the whole trick, boundaries depend on content patterns, not just byte offsets.

---

# 6) How real CDC works

Real systems use a **rolling hash**, such as Rabin fingerprinting.

### Rolling hash means:
When you move one byte forward:

- remove the old byte from the hash window
- add the new byte
- update the hash quickly

This lets the system scan the file efficiently.

### Boundary rule
A chunk boundary is chosen when the hash matches some condition, for example:

- lower 13 bits are zero
- hash modulo a number equals a target
- hash falls in a specific range

That makes average chunk size predictable, like around 4 MB, but the exact boundaries still depend on the content.

---

# 7) How a block is decided, step by step

Here is the practical process:

1. Start at the beginning of the file
2. Keep reading bytes into the current chunk
3. Compute rolling hash as you go
4. If the hash matches the boundary rule:
   - end the chunk
   - store its hash
5. Start a new chunk from the next byte
6. Repeat until end of file

So a block is decided by the **content pattern match**, not by a fixed counter.

---

# 8) Example with file edit

Suppose original file is:

```text
AAAAABBBBBCCCCCDDDDDEEEEE
```

CDC might split it as:

```text
[AAAAA][BBBBB][CCCCC][DDDDD][EEEEE]
```

Now insert `XX` near the front:

```text
AAXXAAABBBBBCCCCCDDDDDEEEEE
```

With fixed-size chunking, all later cuts may change.

With CDC, maybe the boundaries become:

```text
[AAXXAAA][BBBBB][CCCCC][DDDDD][EEEEE]
```

Only the first chunk changed, the rest still match.

That means only one new block may need uploading.

---

# 9) Why this is better for Drive-like systems

If a user edits one paragraph in a document:

- fixed-size chunking may cause many chunks to change
- CDC often preserves later chunk boundaries
- so the system uploads fewer blocks

This reduces:

- upload time
- network usage
- storage duplication

---

# 10) Important trade-off

CDC is better for dedup and sync, but it is a little more complex than fixed-size chunking.

It needs:

- rolling hash computation
- boundary rule logic
- more CPU than naive fixed splitting

But for cloud file sync systems, the savings are usually worth it.

---

# Final answer

A block is decided in content-defined chunking by **scanning the file and cutting when a rolling hash matches a boundary rule**.  
So the chunk boundary depends on the **data pattern**, not on a fixed byte count.

### Example:
- fixed-size: cut every 4 MB
- CDC: cut when rolling hash says “boundary here”

That is why when you insert bytes near the start, CDC usually changes only nearby chunks, while fixed-size chunking may shift all later chunks.

If you want, I can now show a **tiny 30-byte worked example with rolling-hash-style boundaries** so you can see exactly how blocks are formed.
