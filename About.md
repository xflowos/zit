# ZIT: Zetabyte Information Tracker

**A Technical Specification**

---

## Executive Summary

ZIT (Zetabyte Information Tracker) is a version control system built on line-level content addressing with BLAKE3 hashing. It eliminates traditional Git's server-side delta compression and pack file generation by storing content as immutable line objects and representing files as ordered lists of line hashes.

**Core mechanism:** Files are stored as `{hash}.lines` files containing ordered sequences of line hashes. All operations reduce to hash comparisons and content assembly.

---

## 1. Architecture Overview

### 1.1 Design Principles

1. **Content addressing at multiple levels:**
   - Lines identified by BLAKE3(content)
   - Files identified by BLAKE3(ordered line hashes)
   - Trees identified by BLAKE3(directory structure)

2. **Hash once, verify once, serve forever:**
   - Client computes hashes
   - Server verifies at ingest
   - Storage integrity handled by backend (ZFS scrub)
   - Reads require no rehashing

3. **Client-side computation:**
   - Line splitting and hashing
   - Tree construction
   - Deduplication detection
   - Server only validates and stores

### 1.2 System Components

**Client:**
- Local ZFS dataset (optional but recommended)
- VFS layer for hash computation
- Working tree appears as normal filesystem

**Server (ZitHub):**
- Content store: `content/{hash}` (line content)
- File store: `{hash}.lines` (ordered list of line hashes)
- Tree store: `{hash}.tree` (directory mappings)
- API layer for ingress/egress
- VFS/9P layer for path-based access

**Distribution:**
- CDN for public access (CloudFront, etc.)
- TailScale overlay for private P2P distribution
- Optional: Traditional HTTPS endpoints

---

## 2. Content Model

### 2.1 Line Objects

**Definition:** A line is a sequence of bytes ending with `\n` (or EOF for final line).

**Storage format:**
```
Path:  content/{BLAKE3_HASH}
Value: Raw bytes of the line
```

**Properties:**
- Immutable
- Stored once regardless of usage count
- No metadata beyond the hash itself
- Size: typically 10-500 bytes per line

**Example:**
```
Line content: "import React from 'react';\n"
BLAKE3 hash:  abc123def456...
Stored at:    content/abc123def456...
```

### 2.2 File Objects

**Definition:** A file is represented as an ordered list of line hashes.

**Storage format:**
```
Path:  {FILE_HASH}.lines
Value: Newline-separated list of BLAKE3 hashes
```

**File hash computation:**
```
file_content = hash1 + "\n" + hash2 + "\n" + hash3 + "\n"
file_hash = BLAKE3(file_content)
```

**Example:**
```
File: src/index.js containing 3 lines

.lines file content:
abc123def456...
def456abc789...
789abc123def...

File hash: xyz789... = BLAKE3(above content)
Stored at: xyz789.lines
```

**Properties:**
- File identity derived from its line sequence
- Two files with identical content have identical hashes
- Changing one line produces new file hash
- Original line hashes are reused

### 2.3 Tree Objects

**Definition:** A tree maps filesystem paths to file hashes within a directory.

**Storage format:**
```
Path:  {TREE_HASH}.tree
Value: Newline-separated entries of "path hash" pairs
```

**Format specification:**
```
Each line: <filepath><TAB><file_hash>
Sorted lexicographically by filepath
```

**Example:**
```
Tree content:
README.md\tabc123...
src/index.js\txyz789...
src/utils.py\tdef456...

Tree hash: tree_abc... = BLAKE3(above content)
Stored at: tree_abc.tree
```

**Properties:**
- Tree identity derived from its structure
- Identical directory structures have identical hashes
- Trees can reference other trees (subdirectories)

### 2.4 References (Branches/Tags)

**Definition:** A reference is a named pointer to a tree hash.

**Storage format:**
```
Path:  refs/{user}/{repo}/{branch}
Value: {TREE_HASH}
```

**Example:**
```
refs/alice/myapp/main → tree_abc123...
refs/alice/myapp/feature-x → tree_def456...
```

**Properties:**
- References are mutable (only mutable component)
- Updates are atomic (compare-and-swap)
- History is implicit via tree relationships

---

## 3. Operations

### 3.1 Write Path (Push)

**Client operations:**

1. **Hash all lines:**
```python
for line in file.readlines():
    hash = blake3(line)
    line_hashes.append(hash)
    if not exists_locally(hash):
        new_content[hash] = line
```

2. **Construct file hash:**
```python
file_content = "\n".join(line_hashes)
file_hash = blake3(file_content)
```

3. **Construct tree:**
```python
tree_entries = []
for path, file_hash in sorted(files.items()):
    tree_entries.append(f"{path}\t{file_hash}")
tree_content = "\n".join(tree_entries)
tree_hash = blake3(tree_content)
```

4. **Query server for missing hashes:**
```http
POST /api/check-hashes
Content-Type: application/json

{
  "hashes": ["abc123...", "def456...", ...]
}

Response:
{
  "missing": ["def456..."]
}
```

5. **Upload missing content:**
```http
PUT /api/content/def456...
Content-Type: application/octet-stream

<raw line bytes>
```

6. **Upload file definitions:**
```http
PUT /api/lines/xyz789...
Content-Type: text/plain

abc123...
def456...
ghi789...
```

7. **Upload tree:**
```http
PUT /api/trees/tree_abc...
Content-Type: text/plain

README.md\tabc123...
src/index.js\txyz789...
```

8. **Update reference:**
```http
POST /api/refs/alice/myapp/main
Content-Type: application/json

{
  "old_hash": "tree_old...",
  "new_hash": "tree_abc...",
  "cas": true
}
```

**Server operations:**

1. Verify hash matches content (one-time)
2. Store objects if verification passes
3. Atomic reference update with CAS
4. Return success/failure

**CPU cost:**
- Client: O(total lines) for hashing
- Server: O(new objects) for verification
- Network: O(new content) for transfer

### 3.2 Read Path (Pull/Clone)

**Client operations:**

1. **Fetch reference:**
```http
GET /api/refs/alice/myapp/main

Response:
tree_abc123...
```

2. **Fetch tree:**
```http
GET /api/trees/tree_abc123...

Response:
README.md\tabc123...
src/index.js\txyz789...
```

3. **Fetch file definitions:**
```http
GET /api/lines/xyz789...

Response:
abc123...
def456...
ghi789...
```

4. **Fetch line content (parallel):**
```http
GET /api/content/abc123...
GET /api/content/def456...
GET /api/content/ghi789...
```

5. **Assemble locally:**
```python
for hash in line_hashes:
    line_content = fetch(f"content/{hash}")
    file_content += line_content

write_to_disk(filepath, file_content)
```

**Server operations:**
- Lookup hash in content store
- Return raw bytes
- Zero computation required

**CPU cost:**
- Client: String concatenation only
- Server: File I/O only (or CDN cache hit)

### 3.3 Diff Operation

**Traditional Git approach:**
```
1. Read both file versions
2. Run Myers diff algorithm: O(n×d) where d=differences
3. Generate patch format
```

**ZIT approach:**
```python
# Load both file hash lists
old_hashes = fetch(f"{old_file_hash}.lines").split("\n")
new_hashes = fetch(f"{new_file_hash}.lines").split("\n")

# Compare arrays
for i, (old_h, new_h) in enumerate(zip(old_hashes, new_hashes)):
    if old_h != new_h:
        old_line = fetch(f"content/{old_h}")
        new_line = fetch(f"content/{new_h}")
        print(f"Line {i}: -{old_line}")
        print(f"Line {i}: +{new_line}")
```

**Properties:**
- O(n) array comparison
- No diff algorithm needed
- Only fetch content for changed lines
- Result is deterministic

### 3.4 Branch Operations

**Create branch (local with ZFS):**
```bash
# ZFS clone: instant, zero-copy
zfs clone pool/repos/alice/myapp/main@snapshot \
          pool/repos/alice/myapp/feature-x
```

**Create branch (remote):**
```http
# Just copy the reference
POST /api/refs/alice/myapp/feature-x
{
  "tree_hash": "tree_abc123..."
}
```

**Cost:**
- Local: Instant (ZFS COW)
- Remote: Single key-value write

**Switch branch (local):**
```bash
# Unmount current, mount new dataset
# Or: just update working tree pointer
```

**Cost:** Pointer update, no file I/O until files accessed

---

## 4. Storage Backend

### 4.1 Local Storage (Client)

**Recommended: ZFS dataset**

Benefits:
- Block-level checksums (integrity)
- Snapshots (instant, COW)
- Clones (zero-cost branching)
- Scrub (detect bit rot)
- Compression (transparent)

**Not required:** Any filesystem works, but loses COW benefits.

**Local structure:**
```
/zpool/repos/alice/myapp/
  main/              # ZFS dataset
    src/
      index.js       # Regular file
    README.md
  .zit/
    refs/
      main → tree_abc123...
    cache/           # Downloaded objects
      content/
      lines/
```

### 4.2 Remote Storage (Server)

**Content Store: S3 or S3-compatible**

```
Bucket: zithub-content
├── content/
│   ├── abc123.../
│   ├── def456.../
│   └── ...
├── lines/
│   ├── xyz789.lines
│   └── ...
└── trees/
    ├── tree_abc.tree
    └── ...
```

**Metadata Store: Key-Value Database**

```
refs/{user}/{repo}/{branch} → tree_hash
users/{user}/repos → [repo_list]
repos/{user}/{repo}/metadata → {description, ...}
```

**Properties:**
- Content store is append-only
- Objects never modified after creation
- Garbage collection is optional (run offline)
- No complex database transactions needed

### 4.3 Deduplication Mechanism

**Implicit through content addressing:**

1. Client hashes line: `abc123...`
2. Checks if `content/abc123...` exists (local then remote)
3. If exists: skip upload, record reference
4. If missing: upload once

**No explicit dedup algorithm needed.**

**Measurement:**
```
Total line references: COUNT(all line hashes in all .lines files)
Unique line objects: COUNT(files in content/)
Dedup ratio: 1 - (unique / total)
```

---

## 5. Network Protocol

### 5.1 HTTP API

**Content operations:**
```
GET  /api/content/{hash}           # Fetch line content
PUT  /api/content/{hash}           # Upload line content
HEAD /api/content/{hash}           # Check existence

GET  /api/lines/{hash}             # Fetch file definition
PUT  /api/lines/{hash}             # Upload file definition

GET  /api/trees/{hash}             # Fetch tree
PUT  /api/trees/{hash}             # Upload tree
```

**Reference operations:**
```
GET  /api/refs/{user}/{repo}/{branch}    # Get tree hash
POST /api/refs/{user}/{repo}/{branch}    # Update reference
  Body: {"old_hash": "...", "new_hash": "...", "cas": true}
```

**Batch operations:**
```
POST /api/check-hashes             # Check which hashes exist
  Body: {"hashes": ["abc...", "def..."]}
  Response: {"missing": ["def..."]}
```

### 5.2 Path-Based Access (VFS Layer)

**GitHub-style URLs:**
```
GET /{user}/{repo}/{branch}/{path}

Example:
GET /alice/myapp/main/src/index.js
```

**Translation:**
```python
def serve_file(user, repo, branch, path):
    # 1. Resolve reference
    tree_hash = db.get(f"refs/{user}/{repo}/{branch}")
    
    # 2. Load tree
    tree = s3.get(f"trees/{tree_hash}.tree")
    
    # 3. Find file hash
    for line in tree.split("\n"):
        file_path, file_hash = line.split("\t")
        if file_path == path:
            break
    
    # 4. Load line hashes
    line_hashes = s3.get(f"lines/{file_hash}.lines").split("\n")
    
    # 5. Assemble content
    lines = [s3.get(f"content/{h}") for h in line_hashes]
    content = "".join(lines)
    
    # 6. Return with caching headers
    return Response(
        content,
        headers={
            "Content-Type": "text/plain",
            "ETag": file_hash,
            "Cache-Control": "public, max-age=31536000, immutable"
        }
    )
```

**Properties:**
- First request: assembles from hashes
- Subsequent requests: CDN cache hit
- Cache key: path + branch + tree_hash
- Invalidation: automatic when tree_hash changes

---

## 6. TailScale Integration

### 6.1 Private Network Overlay

**TailScale provides:**
- WireGuard-based encrypted mesh network
- Automatic NAT traversal
- Identity-based access control
- Zero-configuration peering

**Integration model:**

```
┌─────────────────────────────────────────────┐
│           TailScale Network                 │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │ Client A │  │ Client B │  │ Client C │ │
│  │ (Laptop) │  │(Desktop) │  │ (CI/CD)  │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
│       │             │             │        │
│       └─────────┬───┴─────────────┘        │
│                 │                           │
│           ┌─────▼──────┐                    │
│           │  Gateway   │                    │
│           │    Node    │                    │
│           └─────┬──────┘                    │
└─────────────────┼──────────────────────────┘
                  │
            ┌─────▼──────┐
            │   ZitHub   │
            │  (Public)  │
            └────────────┘
```

### 6.2 Peer-to-Peer Distribution

**Within TailScale network:**

1. Client A pushes content to gateway
2. Gateway uploads to ZitHub
3. Client B requests same content
4. Gateway serves from local cache (or fetches once)
5. Clients C, D, E all fetch from gateway cache

**Benefits:**
- Single external upload per team
- Internal distribution at LAN speeds
- Encrypted by default (WireGuard)
- No VPN configuration needed

**Implementation:**
```python
# Gateway node acts as caching proxy
@app.route('/api/content/<hash>')
def get_content(hash):
    # Check local cache
    if exists(f"cache/content/{hash}"):
        return send_file(f"cache/content/{hash}")
    
    # Fetch from ZitHub
    content = zithub_client.get(f"content/{hash}")
    
    # Cache locally
    write_file(f"cache/content/{hash}", content)
    
    return content
```

### 6.3 Authorization Model

**TailScale ACLs:**
```json
{
  "groups": {
    "group:devs": ["user@example.com", "user2@example.com"]
  },
  "acls": [
    {
      "action": "accept",
      "users": ["group:devs"],
      "ports": ["gateway:443"]
    }
  ]
}
```

**ZIT authorization:**
- Read: Any TailScale network member can read
- Write: Configured per repo/branch
- Admin: Separate permission layer

---

## 7. Performance Characteristics

### 7.1 Operation Costs

**Hash computation (BLAKE3):**
- Single-threaded: ~10 GB/s
- Multi-threaded: ~100+ GB/s
- 1000-line file: <1ms to hash

**Write path:**
```
Hash all lines:           O(total_bytes) at ~10GB/s
Check existence:          O(unique_lines) × 1ms (network RTT)
Upload new content:       O(new_bytes) / bandwidth
Construct tree:           O(files) × 1ms
Update reference:         O(1) × 10ms
```

**Read path (cold):**
```
Fetch reference:          1 × 10ms
Fetch tree:               1 × 10ms
Fetch .lines:             N × 10ms (N = number of files)
Fetch content:            M × 10ms (M = number of unique lines)
Assembly:                 O(lines) string concatenation
```

**Read path (hot - CDN cached):**
```
All objects cached:       <10ms edge latency
Assembly:                 Zero (full file cached)
```

**Diff:**
```
Load two .lines files:    2 × 10ms
Compare arrays:           O(lines) × 10ns per comparison
Fetch changed lines:      K × 10ms (K = number of changes)

Example: 1000-line file, 10 changes
  = 20ms + 10μs + 100ms
  = ~120ms total
```

### 7.2 Caching Strategy

**Immutable objects (forever cache):**
```
content/{hash}
{hash}.lines
{hash}.tree

Cache-Control: public, max-age=31536000, immutable
```

**Mutable references (short cache):**
```
refs/{user}/{repo}/{branch}

Cache-Control: public, max-age=60
```

**Assembled files (conditional):**
```
/{user}/{repo}/{branch}/{path}

Cache-Control: public, max-age=3600
ETag: {file_hash}

If tree changes → ETag changes → cache invalidates
```

### 7.3 Storage Efficiency

**Measurement methodology:**

1. Count total line instances across all files
2. Count unique line hashes in content store
3. Calculate: `dedup_ratio = 1 - (unique / total)`

**Expected patterns:**

Common patterns with high reuse:
- Empty lines
- Brace characters: `{`, `}`, `[`, `]`
- License headers (identical across many repos)
- Common imports: `import React from 'react'`
- Generated code patterns

Unique content:
- Application logic
- Documentation prose
- Configuration values

**Storage breakdown:**
```
Total storage = unique_lines × avg_line_size
              + file_count × avg_file_definition_size
              + tree_count × avg_tree_size

Where:
  unique_lines: measured per system
  avg_line_size: typically 50-100 bytes
  file_definition_size: num_lines × 32 bytes (hash size)
  tree_size: num_files × (path_length + 32 bytes)
```

---

## 8. Security Model

### 8.1 Integrity Guarantees

**Content integrity:**
- Every object verified at ingest: `BLAKE3(content) == filename`
- Collision resistance: 2^256 (computationally infeasible)
- Immutability: Objects never modified after creation

**Storage integrity:**
- Client: ZFS block checksums + scrub
- Server: S3 object checksums
- Transport: TLS 1.3 (or WireGuard for TailScale)

**Trust chain:**
```
Client hashes → Server verifies once → Storage preserves → CDN serves
```

**Properties:**
- Server never re-hashes on read (trusts filename + storage checksum)
- Corruption detected by storage layer
- Tampering impossible (hash mismatch would be detected)

### 8.2 Access Control

**Public repositories:**
- Read: Anyone
- Write: Authenticated users with permission
- No encryption (content is public)

**Private repositories:**

Option 1: TailScale network only
- No public access
- Encrypted WireGuard transport
- Identity-based ACLs

Option 2: Server-side access control
- Read: Authenticated API requests
- Content still stored in plaintext on server
- Trust model: Same as GitHub private repos

**Cannot have:**
- End-to-end encryption + global deduplication
- (Different ciphertexts for same plaintext break dedup)

### 8.3 Authentication

**API authentication:**
```http
Authorization: Bearer <token>

Or:

X-ZIT-User: alice
X-ZIT-Signature: HMAC-SHA256(request, user_secret)
```

**TailScale authentication:**
- Automatic via WireGuard identity
- No additional tokens needed
- ACLs enforced at network layer

---

## 9. Implementation Specification

### 9.1 File Formats

**content/{hash} (line content):**
```
Format: Raw bytes
Encoding: Preserve original (no normalization)
Maximum size: Recommend 32KB limit for individual lines
  (split longer lines into chunks if needed)
```

**{hash}.lines (file definition):**
```
Format: Newline-separated BLAKE3 hashes (hex-encoded)
Line format: [a-f0-9]{64}\n
Example:
abc123...def (64 hex chars)
def456...abc (64 hex chars)
```

**{hash}.tree (directory tree):**
```
Format: Newline-separated "path<TAB>hash" entries
Sorted: Lexicographically by path
Line format: <path>\t[a-f0-9]{64}\n
Example:
README.md\tabc123...def
src/index.js\txyz789...abc
src/utils.py\tdef456...ghi
```

**refs/{user}/{repo}/{branch} (reference):**
```
Format: Single BLAKE3 hash (hex-encoded)
Points to: Tree hash
Example:
tree_abc123def456...
```

### 9.2 Hash Computation

**Line hashing:**
```python
import hashlib

def hash_line(line_bytes: bytes) -> str:
    """
    Hash a line using BLAKE3.
    
    Args:
        line_bytes: Raw bytes including trailing newline
    
    Returns:
        Hex-encoded BLAKE3 hash (64 characters)
    """
    hasher = hashlib.blake2b(digest_size=32)  # BLAKE3 when available
    hasher.update(line_bytes)
    return hasher.hexdigest()
```

**File hashing:**
```python
def hash_file(line_hashes: list[str]) -> str:
    """
    Hash a file by hashing its ordered line hashes.
    
    Args:
        line_hashes: List of line hash strings
    
    Returns:
        Hex-encoded BLAKE3 hash of the .lines content
    """
    lines_content = "\n".join(line_hashes).encode('utf-8')
    hasher = hashlib.blake2b(digest_size=32)
    hasher.update(lines_content)
    return hasher.hexdigest()
```

**Tree hashing:**
```python
def hash_tree(entries: dict[str, str]) -> str:
    """
    Hash a tree by hashing its sorted path→hash mappings.
    
    Args:
        entries: Dict mapping paths to file hashes
    
    Returns:
        Hex-encoded BLAKE3 hash of the .tree content
    """
    lines = [f"{path}\t{file_hash}" 
             for path, file_hash in sorted(entries.items())]
    tree_content = "\n".join(lines).encode('utf-8')
    
    hasher = hashlib.blake2b(digest_size=32)
    hasher.update(tree_content)
    return hasher.hexdigest()
```

### 9.3 Client Implementation Pseudocode

**Push operation:**
```python
def push(repo_path: str, remote_url: str):
    # 1. Scan working tree
    all_files = walk_directory(repo_path)
    
    # 2. Hash all lines
    file_hashes = {}
    new_content = {}
    
    for filepath in all_files:
        line_hashes = []
        with open(filepath, 'rb') as f:
            for line in f:
                h = hash_line(line)
                line_hashes.append(h)
                
                if not exists_locally(h) and h not in new_content:
                    new_content[h] = line
        
        file_hash = hash_file(line_hashes)
        file_hashes[filepath] = file_hash
        
        # Store .lines file locally
        write_lines_file(file_hash, line_hashes)
    
    # 3. Construct tree
    tree_hash = hash_tree(file_hashes)
    write_tree_file(tree_hash, file_hashes)
    
    # 4. Query server for missing hashes
    all_hashes = set(new_content.keys())
    missing = http_post(f"{remote_url}/api/check-hashes", 
                       {"hashes": list(all_hashes)})
    
    # 5. Upload missing content
    for h in missing:
        http_put(f"{remote_url}/api/content/{h}", new_content[h])
    
    # 6. Upload file definitions
    for file_hash in file_hashes.values():
        if not server_has(file_hash):
            lines_content = read_lines_file(file_hash)
            http_put(f"{remote_url}/api/lines/{file_hash}", lines_content)
    
    # 7. Upload tree
    if not server_has(tree_hash):
        tree_content = read_tree_file(tree_hash)
        http_put(f"{remote_url}/api/trees/{tree_hash}", tree_content)
    
    # 8. Update reference
    branch = get_current_branch()
    http_post(f"{remote_url}/api/refs/{user}/{repo}/{branch}",
             {"new_hash": tree_hash, "cas": true})
```

**Pull operation:**
```python
def pull(remote_url: str, repo_path: str):
    # 1. Fetch current reference
    branch = get_current_branch()
    tree_hash = http_get(f"{remote_url}/api/refs/{user}/{repo}/{branch}")
    
    # 2. Fetch tree
    tree_content = http_get(f"{remote_url}/api/trees/{tree_hash}")
    file_entries = parse_tree(tree_content)
    
    # 3. For each file, fetch line hashes
    for filepath, file_hash in file_entries.items():
        lines_content = http_get(f"{remote_url}/api/lines/{file_hash}")
        line_hashes = lines_content.strip().split("\n")
        
        # 4. Fetch missing line content
        lines = []
        for h in line_hashes:
            if not exists_locally(h):
                content = http_get(f"{remote_url}/api/content/{h}")
                cache_locally(h, content)
            lines.append(read_from_cache(h))
        
        # 5. Write file
        full_content = b"".join(lines)
        write_file(os.path.join(repo_path, filepath), full_content)
```

### 9.4 Server Implementation Pseudocode

**Ingest endpoint:**
```python
@app.put('/api/content/<hash>')
def put_content(hash):
    # 1. Read uploaded content
    content = request.get_data()
    
    # 2. Verify hash matches
    computed_hash = blake3(content).hexdigest()
    if computed_hash != hash:
        return {"error": "Hash mismatch"}, 400
    
    # 3. Store in S3
    s3_client.put_object(
        Bucket='zithub-content',
        Key=f'content/{hash}',
        Body=content,
        ContentType='application/octet-stream'
    )
    
    return {"status": "ok"}, 201
```

**Path-based file serving:**
```python
@app.get('/<user>/<repo>/<branch>/<path:filepath>')
@cache_control("public, max-age=3600")
def serve_file(user, repo, branch, filepath):
    # 1. Resolve reference
    tree_hash = db.get(f"refs/{user}/{repo}/{branch}")
    if not tree_hash:
        return {"error": "Branch not found"}, 404
    
    # 2. Load tree
    tree_content = s3_get(f"trees/{tree_hash}.tree")
    file_hash = parse_tree_find_file(tree_content, filepath)
    if not file_hash:
        return {"error": "File not found"}, 404
    
    # 3. Load line hashes
    lines_content = s3_get(f"lines/{file_hash}.lines")
    line_hashes = lines_content.strip().split("\n")
    
    # 4. Fetch all line content (parallel)
    with ThreadPoolExecutor(max_workers=50) as executor:
        lines = list(executor.map(
            lambda h: s3_get(f"content/{h}"),
            line_hashes
        ))
    
    # 5. Assemble and return
    full_content = b"".join(lines)
    
    return Response(
        full_content,
        headers={
            "ETag": file_hash,
            "Content-Type": guess_mime_type(filepath)
        }
    )
```

---

## 10. Comparison with Git

### 10.1 Conceptual Differences

| Aspect | Git | ZIT |
|--------|-----|-----|
| Atomic unit | File blob | Line |
| File identity | Hash of content | Hash of line hash list |
| Dedup scope | Per-repo (pack files) | Global (content-addressed lines) |
| Diff algorithm | Myers algorithm | Array comparison |
| Server CPU | Delta compression | Hash verification only |
| Storage format | Pack files with deltas | Immutable line objects |
| Branch cost | Pointer update | Pointer update (same) |

### 10.2 Operational Differences

**Push operation:**

Git:
```
1. Client sends objects
2. Server receives pack file
3. Server decompresses
4. Server computes deltas (CPU intensive)
5. Server updates pack files
6. Server updates refs
```

ZIT:
```
1. Client hashes lines
2. Client asks which hashes are missing
3. Client uploads only missing hashes
4. Server verifies hashes (one-time)
5. Server stores objects
6. Server updates refs
```

**Clone operation:**

**Clone operation (continued):**

Git:
```
1. Server walks object graph
2. Server computes optimal deltas (CPU intensive)
3. Server generates pack file
4. Server compresses
5. Client receives
6. Client unpacks
7. Client writes working tree
```

ZIT:
```
1. Client fetches reference (tree hash)
2. Client fetches tree (file list)
3. Client fetches .lines files (parallel)
4. Client fetches line content (parallel, from CDN)
5. Client assembles files (string concatenation)
6. Client writes working tree
```

**Diff operation:**

Git:
```
1. Load both blob objects
2. Run Myers diff algorithm: O(n×d)
3. Generate unified diff format
```

ZIT:
```
1. Load both .lines files
2. Compare hash arrays: O(n)
3. Fetch content only for changed lines
4. Display differences
```

### 10.3 Server Resource Comparison

**Theoretical analysis for a large hosting platform:**

Assumptions:
- 10M pushes/day
- 50M clone/fetch operations/day
- Average 1000 lines per file, 100 files per repo

**Git server costs:**
```
Push (per operation):
  - Delta compression: ~500ms CPU
  - Pack file update: ~100ms I/O
  
Clone (per operation):
  - Object graph walk: ~2s CPU
  - Delta computation: ~5s CPU
  - Pack generation: ~3s CPU
  - Total: ~10s CPU per clone

Daily CPU:
  Pushes:  10M × 0.5s = 5M CPU-seconds
  Clones:  50M × 10s = 500M CPU-seconds
  Total:   505M CPU-seconds/day = 5,845 CPU-hours/day
  
Sustained: ~244 cores (assuming 24 hour spread)
Peak (8hr workday): ~732 cores
```

**ZIT server costs:**
```
Push (per operation):
  - Hash verification: ~5ms CPU (only for new content)
  - Object storage: S3 handles
  
Clone (per operation):
  - Reference lookup: ~1ms CPU
  - Object fetching: CDN handles (zero origin CPU)
  
Daily CPU (assuming 40% new content on pushes):
  Pushes:  10M × 0.005s × 0.4 = 20K CPU-seconds
  Clones:  50M × 0.001s = 50K CPU-seconds
  Total:   70K CPU-seconds/day = 19 CPU-hours/day
  
Sustained: ~0.8 cores
Peak: ~2.4 cores
```

**Cost reduction: ~300x less CPU required**

Note: These are theoretical calculations based on operation complexity. Real-world measurements would depend on:
- Actual repository sizes
- Cache hit rates
- Network latency
- Storage backend performance
- Specific implementation efficiency

---

## 11. Deployment Architecture

### 11.1 Single-Server Deployment

**Minimal viable setup:**

```
┌─────────────────────────────────────┐
│         Single EC2 Instance         │
│                                     │
│  ┌──────────────────────────────┐  │
│  │     Nginx (Reverse Proxy)    │  │
│  └────────────┬─────────────────┘  │
│               │                     │
│  ┌────────────▼─────────────────┐  │
│  │   ZIT API Server (Python)    │  │
│  │   - Ingress validation       │  │
│  │   - Reference management     │  │
│  │   - VFS path translation     │  │
│  └────────────┬─────────────────┘  │
│               │                     │
│  ┌────────────▼─────────────────┐  │
│  │     PostgreSQL (Refs)        │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│         Amazon S3 (Content)         │
│  - content/{hash}                   │
│  - {hash}.lines                     │
│  - {hash}.tree                      │
└─────────────────────────────────────┘
```

**Capacity:**
- Handles 1000s of repos
- 100s of concurrent users
- Cost: ~$100-200/month

### 11.2 Production Deployment

**Scalable architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    CloudFront CDN                        │
│  (Caches immutable objects, serves 95%+ of reads)       │
└────────────────────────┬────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
┌────────▼────────┐ ┌───▼────────┐ ┌───▼────────┐
│  API Server 1   │ │ API Server │ │ API Server │
│  (Auto-scaling) │ │     2      │ │     3      │
└────────┬────────┘ └───┬────────┘ └───┬────────┘
         │              │              │
         └──────────────┼──────────────┘
                        │
         ┌──────────────┴──────────────┐
         │                             │
┌────────▼────────┐         ┌─────────▼────────┐
│   PostgreSQL    │         │    Amazon S3     │
│   RDS (Refs)    │         │   (Content)      │
│  (Multi-AZ)     │         │  (Multi-region)  │
└─────────────────┘         └──────────────────┘
```

**Components:**

1. **CloudFront CDN:**
   - Caches all `content/{hash}` objects
   - Caches `.lines` and `.tree` files
   - Caches assembled files at path-based URLs
   - 95%+ cache hit rate expected
   - Zero origin load for hot content

2. **API Servers (Auto-scaling):**
   - Stateless application servers
   - Handle ingress validation
   - Serve reference updates
   - Assemble files on cache miss
   - Scale horizontally based on load

3. **PostgreSQL RDS:**
   - Stores references (branches/tags)
   - Stores user/repo metadata
   - Stores access control lists
   - Multi-AZ for high availability

4. **S3 Storage:**
   - Content store (immutable objects)
   - Multi-region replication optional
   - Lifecycle policies for archival
   - Versioning disabled (objects are immutable)

**Capacity:**
- Millions of repos
- 100K+ concurrent users
- Cost: Variable based on usage

### 11.3 TailScale-Enhanced Deployment

**Enterprise private network:**

```
┌─────────────────────────────────────────────────────────┐
│                  TailScale Network                       │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │Developer │  │Developer │  │  CI/CD   │             │
│  │  Node 1  │  │  Node 2  │  │  Runner  │             │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘             │
│       │             │             │                     │
│       └─────────────┼─────────────┘                     │
│                     │                                   │
│            ┌────────▼────────┐                          │
│            │  Gateway Node   │                          │
│            │  (Team Cache)   │                          │
│            │  - Local cache  │                          │
│            │  - Proxy to hub │                          │
│            └────────┬────────┘                          │
└─────────────────────┼──────────────────────────────────┘
                      │
                      │ (External network)
                      │
              ┌───────▼────────┐
              │     ZitHub     │
              │  (Public/SaaS) │
              └────────────────┘
```

**Benefits:**
- Single external upload per team (at gateway)
- Internal distribution at LAN speeds (100MB/s+)
- Encrypted by default (WireGuard)
- No VPN configuration required
- Works through NAT/firewalls
- Identity-based access control

**Gateway node setup:**
```python
# Simple caching proxy
class GatewayNode:
    def __init__(self, zithub_url, cache_dir):
        self.zithub_url = zithub_url
        self.cache_dir = cache_dir
    
    def get_content(self, hash):
        cache_path = f"{self.cache_dir}/content/{hash}"
        
        # Check cache first
        if os.path.exists(cache_path):
            return open(cache_path, 'rb').read()
        
        # Fetch from ZitHub
        response = requests.get(f"{self.zithub_url}/api/content/{hash}")
        content = response.content
        
        # Cache locally
        os.makedirs(os.path.dirname(cache_path), exist_ok=True)
        with open(cache_path, 'wb') as f:
            f.write(content)
        
        return content
    
    def put_content(self, hash, content):
        # Cache locally
        cache_path = f"{self.cache_dir}/content/{hash}"
        os.makedirs(os.path.dirname(cache_path), exist_ok=True)
        with open(cache_path, 'wb') as f:
            f.write(content)
        
        # Upload to ZitHub
        requests.put(
            f"{self.zithub_url}/api/content/{hash}",
            data=content
        )
```

---

## 12. Migration Strategy

### 12.1 From Git to ZIT

**Conversion process:**

```python
def convert_git_repo_to_zit(git_repo_path):
    # 1. Initialize ZIT repo
    zit_repo = ZitRepo.init()
    
    # 2. For each commit in Git history
    for commit in git_log_all():
        # Checkout commit
        git_checkout(commit.sha)
        
        # Hash all files
        file_hashes = {}
        for filepath in git_ls_files():
            line_hashes = []
            with open(filepath, 'rb') as f:
                for line in f:
                    h = blake3(line).hexdigest()
                    line_hashes.append(h)
                    zit_repo.store_content(h, line)
            
            file_hash = zit_repo.store_file_definition(line_hashes)
            file_hashes[filepath] = file_hash
        
        # Create tree
        tree_hash = zit_repo.store_tree(file_hashes)
        
        # Create commit metadata (optional)
        zit_repo.store_commit_metadata(
            tree_hash=tree_hash,
            message=commit.message,
            author=commit.author,
            timestamp=commit.timestamp,
            parent=previous_tree_hash
        )
        
        previous_tree_hash = tree_hash
    
    # 3. Create branch refs
    for branch in git_branches():
        tree_hash = convert_commit_to_tree(branch.commit)
        zit_repo.update_ref(branch.name, tree_hash)
```

**Properties:**
- One-way conversion (Git → ZIT)
- History preserved (as tree changes)
- Commit metadata stored separately if needed
- Can be done incrementally

### 12.2 Coexistence Strategy

**Dual operation:**

```
Developer workflow:
1. Work in Git locally (familiar tools)
2. Push to Git server (traditional)
3. Automated sync to ZIT (background)

OR

1. Work with ZIT client
2. Automated sync to Git (for compatibility)
```

**Bridge service:**
```python
class GitZitBridge:
    def on_git_push(self, repo, branch, commit):
        # Convert commit to ZIT format
        tree_hash = convert_commit_to_zit(commit)
        
        # Push to ZIT
        zit_client.update_ref(repo, branch, tree_hash)
    
    def on_zit_update(self, repo, branch, tree_hash):
        # Convert ZIT tree to Git commit
        commit = convert_zit_to_git(tree_hash)
        
        # Push to Git
        git_push(repo, branch, commit)
```

---

## 13. Monitoring and Operations

### 13.1 Key Metrics

**Storage metrics:**
```
unique_objects = COUNT(content/* + *.lines + *.tree)
total_size = SUM(object sizes)
dedup_ratio = 1 - (unique_objects / total_references)
```

**Performance metrics:**
```
push_latency_p50 = median(push operation duration)
push_latency_p99 = 99th percentile
clone_latency_p50 = median(clone operation duration)
cdn_hit_rate = cdn_hits / (cdn_hits + cdn_misses)
```

**Server health:**
```
api_requests_per_second
cpu_utilization
memory_utilization
s3_request_count
s3_error_rate
```

### 13.2 Operational Procedures

**Garbage collection:**
```python
def garbage_collect():
    # 1. Find all referenced objects
    referenced = set()
    
    for ref in list_all_refs():
        tree_hash = get_ref_value(ref)
        referenced.add(tree_hash)
        
        # Walk tree
        tree = load_tree(tree_hash)
        for file_hash in tree.values():
            referenced.add(file_hash)
            
            # Walk file
            lines = load_lines_file(file_hash)
            for line_hash in lines:
                referenced.add(line_hash)
    
    # 2. Find unreferenced objects
    all_objects = set(list_all_objects())
    unreferenced = all_objects - referenced
    
    # 3. Delete after grace period
    for obj_hash in unreferenced:
        if object_age(obj_hash) > grace_period:
            delete_object(obj_hash)
```

**Backup:**
```bash
# S3 versioning (optional)
aws s3api put-bucket-versioning \
    --bucket zithub-content \
    --versioning-configuration Status=Enabled

# Cross-region replication
aws s3api put-bucket-replication \
    --bucket zithub-content \
    --replication-configuration file://replication.json

# Reference database backup
pg_dump zithub_refs > refs_backup.sql
```

**Monitoring alerts:**
```yaml
alerts:
  - name: HighPushLatency
    condition: push_latency_p99 > 5s
    action: Scale API servers
  
  - name: LowCDNHitRate
    condition: cdn_hit_rate < 90%
    action: Investigate cache configuration
  
  - name: S3ErrorRate
    condition: s3_error_rate > 0.1%
    action: Check S3 service health
```

---

## 14. Future Enhancements

### 14.1 Possible Optimizations

**Adaptive chunking:**
- Current: Fixed newline boundaries
- Future: Intelligent splitting of long lines
- Benefit: Better dedup for minified code

**Bloom filters:**
- Current: Server checks each hash individually
- Future: Client-side bloom filter for existence checks
- Benefit: Reduce round-trips during push

**Delta encoding (optional):**
- Current: Store full line content always
- Future: Delta-encode similar lines
- Benefit: Further storage reduction
- Trade-off: Added complexity, reduced simplicity

**Prefetching:**
- Current: Fetch lines on demand
- Future: Predict and prefetch likely-needed lines
- Benefit: Reduced latency

### 14.2 Additional Features

**Commit history:**
```
Current: Trees only (no commit graph)
Future: Optional commit objects linking trees
Format:
  tree_hash: abc123...
  parent_hash: def456...
  author: "Alice <alice@example.com>"
  timestamp: 1704067200
  message: "Add feature X"
```

**Merge support:**
```
Current: Manual merge at application level
Future: Three-way merge using hash comparison
Algorithm:
  base_hashes = load_lines(common_ancestor_file)
  ours_hashes = load_lines(our_file)
  theirs_hashes = load_lines(their_file)
  
  for i in range(max(len(base), len(ours), len(theirs))):
      if ours[i] == theirs[i]:
          merged[i] = ours[i]
      elif ours[i] == base[i]:
          merged[i] = theirs[i]  # They changed
      elif theirs[i] == base[i]:
          merged[i] = ours[i]    # We changed
      else:
          conflict[i] = (ours[i], theirs[i])  # Both changed
```

**Signed commits:**
```
Commit object includes signature:
  tree_hash: abc123...
  signature: <GPG signature of tree_hash>
  public_key: <author's public key>
```

**Large file support:**
```
Strategy: Store large files as separate objects
Threshold: > 1MB triggers special handling
Format: Reference in tree points to blob object
  .tree: "large_file.bin\tblob:xyz789..."
```

---

## 15. Reference Implementation

### 15.1 Minimal Client (Python)

```python
#!/usr/bin/env python3
"""
ZIT - Minimal reference client implementation
"""

import os
import hashlib
import requests
from pathlib import Path
from typing import List, Dict, Set

class ZitClient:
    def __init__(self, repo_path: str, remote_url: str):
        self.repo_path = Path(repo_path)
        self.remote_url = remote_url
        self.zit_dir = self.repo_path / '.zit'
        self.cache_dir = self.zit_dir / 'cache'
        
    def init(self):
        """Initialize a new ZIT repository"""
        self.zit_dir.mkdir(exist_ok=True)
        (self.cache_dir / 'content').mkdir(parents=True, exist_ok=True)
        (self.cache_dir / 'lines').mkdir(parents=True, exist_ok=True)
        (self.cache_dir / 'trees').mkdir(parents=True, exist_ok=True)
        
    def hash_line(self, line: bytes) -> str:
        """Compute BLAKE3 hash of a line"""
        # Using BLAKE2b as proxy for BLAKE3
        hasher = hashlib.blake2b(digest_size=32)
        hasher.update(line)
        return hasher.hexdigest()
    
    def hash_content(self, content: bytes) -> str:
        """Compute BLAKE3 hash of content"""
        hasher = hashlib.blake2b(digest_size=32)
        hasher.update(content)
        return hasher.hexdigest()
    
    def store_content_local(self, hash: str, content: bytes):
        """Store content in local cache"""
        path = self.cache_dir / 'content' / hash
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_bytes(content)
    
    def load_content_local(self, hash: str) -> bytes:
        """Load content from local cache"""
        path = self.cache_dir / 'content' / hash
        if path.exists():
            return path.read_bytes()
        return None
    
    def hash_file(self, filepath: Path) -> tuple[str, List[str]]:
        """
        Hash a file and return (file_hash, line_hashes)
        """
        line_hashes = []
        
        with open(filepath, 'rb') as f:
            for line in f:
                h = self.hash_line(line)
                line_hashes.append(h)
                
                # Cache line content
                if not self.load_content_local(h):
                    self.store_content_local(h, line)
        
        # Compute file hash from line hashes
        lines_content = '\n'.join(line_hashes).encode('utf-8')
        file_hash = self.hash_content(lines_content)
        
        # Store .lines file
        lines_path = self.cache_dir / 'lines' / f'{file_hash}.lines'
        lines_path.parent.mkdir(parents=True, exist_ok=True)
        lines_path.write_text('\n'.join(line_hashes))
        
        return file_hash, line_hashes
    
    def scan_tree(self) -> Dict[str, str]:
        """
        Scan working tree and return {filepath: file_hash}
        """
        file_hashes = {}
        
        for filepath in self.repo_path.rglob('*'):
            if filepath.is_file() and '.zit' not in filepath.parts:
                rel_path = filepath.relative_to(self.repo_path)
                file_hash, _ = self.hash_file(filepath)
                file_hashes[str(rel_path)] = file_hash
        
        return file_hashes
    
    def create_tree(self, file_hashes: Dict[str, str]) -> str:
        """Create tree object and return tree hash"""
        # Sort entries
        entries = [f"{path}\t{fhash}" 
                  for path, fhash in sorted(file_hashes.items())]
        tree_content = '\n'.join(entries).encode('utf-8')
        tree_hash = self.hash_content(tree_content)
        
        # Store tree
        tree_path = self.cache_dir / 'trees' / f'{tree_hash}.tree'
        tree_path.parent.mkdir(parents=True, exist_ok=True)
        tree_path.write_bytes(tree_content)
        
        return tree_hash
    
    def check_remote_hashes(self, hashes: Set[str]) -> Set[str]:
        """Check which hashes are missing on remote"""
        response = requests.post(
            f"{self.remote_url}/api/check-hashes",
            json={"hashes": list(hashes)}
        )
        return set(response.json().get('missing', []))
    
    def upload_content(self, hash: str):
        """Upload content object to remote"""
        content = self.load_content_local(hash)
        if content is None:
            raise ValueError(f"Content not found: {hash}")
        
        requests.put(
            f"{self.remote_url}/api/content/{hash}",
            data=content
        )
    
    def upload_lines(self, file_hash: str):
        """Upload .lines file to remote"""
        path = self.cache_dir / 'lines' / f'{file_hash}.lines'
        content = path.read_bytes()
        
        requests.put(
            f"{self.remote_url}/api/lines/{file_hash}",
            data=content
        )
    
    def upload_tree(self, tree_hash: str):
        """Upload tree to remote"""
        path = self.cache_dir / 'trees' / f'{tree_hash}.tree'
        content = path.read_bytes()
        
        requests.put(
            f"{self.remote_url}/api/trees/{tree_hash}",
            data=content
        )
    
    def update_ref(self, branch: str, tree_hash: str):
        """Update branch reference on remote"""
        user = os.environ.get('ZIT_USER', 'default')
        repo = self.repo_path.name
        
        requests.post(
            f"{self.remote_url}/api/refs/{user}/{repo}/{branch}",
            json={"new_hash": tree_hash, "cas": True}
        )
    
    def push(self, branch: str = 'main'):
        """Push current state to remote"""
        print("Scanning working tree...")
        file_hashes = self.scan_tree()
        
        print(f"Found {len(file_hashes)} files")
        
        print("Creating tree...")
        tree_hash = self.create_tree(file_hashes)
        
        print(f"Tree hash: {tree_hash}")
        
        # Collect all hashes
        all_hashes = set()
        for file_hash in file_hashes.values():
            lines_path = self.cache_dir / 'lines' / f'{file_hash}.lines'
            line_hashes = lines_path.read_text().strip().split('\n')
            all_hashes.update(line_hashes)
        
        print(f"Checking {len(all_hashes)} hashes on remote...")
        missing = self.check_remote_hashes(all_hashes)
        
        print(f"Uploading {len(missing)} new objects...")
        for i, hash in enumerate(missing):
            if (i + 1) % 100 == 0:
                print(f"  {i + 1}/{len(missing)}...")
            self.upload_content(hash)
        
        print("Uploading file definitions...")
        for file_hash in file_hashes.values():
            self.upload_lines(file_hash)
        
        print("Uploading tree...")
        self.upload_tree(tree_hash)
        
        print(f"Updating ref {branch}...")
        self.update_ref(branch, tree_hash)
        
        print("Push complete!")
    
    def fetch_content(self, hash: str) -> bytes:
        """Fetch content from remote"""
        # Check cache first
        content = self.load_content_local(hash)
        if content:
            return content
        
        # Fetch from remote
        response = requests.get(f"{self.remote_url}/api/content/{hash}")
        content = response.content
        
        # Cache locally
        self.store_content_local(hash, content)
        
        return content
    
    def pull(self, branch: str = 'main'):
        """Pull from remote"""
        user = os.environ.get('ZIT_USER', 'default')
        repo = self.repo_path.name
        
        print(f"Fetching ref {branch}...")
        response = requests.get(
            f"{self.remote_url}/api/refs/{user}/{repo}/{branch}"
        )
        tree_hash = response.text.strip()
        
        print(f"Tree hash: {tree_hash}")
        
        print("Fetching tree...")
        response = requests.get(f"{self.remote_url}/api/trees/{tree_hash}")
        tree_content = response.text
        
        # Parse tree
        file_entries = {}
        for line in tree_content.strip().split('\n'):
            path, file_hash = line.split('\t')
            file_entries[path] = file_hash
        
        print(f"Fetching {len(file_entries)} files...")
        for i, (filepath, file_hash) in enumerate(file_entries.items()):
            if (i + 1) % 10 == 0:
                print(f"  {i + 1}/{len(file_entries)}...")
            
            # Fetch .lines file
            response = requests.get(f"{self.remote_url}/api/lines/{file_hash}")
            line_hashes = response.text.strip().split('\n')
            
            # Fetch line content
            lines = []
            for line_hash in line_hashes:
                content = self.fetch_content(line_hash)
                lines.append(content)
            
            # Write file
            full_path = self.repo_path / filepath
            full_path.parent.mkdir(parents=True, exist_ok=True)
            full_path.write_bytes(b''.join(lines))
        
        print("Pull complete!")


# CLI interface
if __name__ == '__main__':
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: zit <command> [args]")
        print("Commands: init, push, pull")
        sys.exit(1)
    
    command = sys.argv[1]
    
    if command == 'init':
        repo_path = sys.argv[2] if len(sys.argv) > 2 else '.'
        remote_url = sys.argv[3] if len(sys.argv) > 3 else 'http://localhost:8000'
        
        client = ZitClient(repo_path, remote_url)
        client.init()
        print(f"Initialized ZIT repository in {repo_path}")
    
    elif command == 'push':
        repo_path = '.'
        remote_url = os.environ.get('ZIT_REMOTE', 'http://localhost:8000')
        branch = sys.argv[2] if len(sys.argv) > 2 else 'main'
        
        client = ZitClient(repo_path, remote_url)
        client.push(branch)
    
    elif command == 'pull':
        repo_path = '.'
        remote_url = os.environ.get('ZIT_REMOTE', 'http://localhost:8000')
        branch = sys.argv[2] if len(sys.argv) > 2 else 'main'
        
        client = ZitClient(repo_path, remote_url)
        client.pull(branch)
    
    else:
        print(f"Unknown command: {command}")
        sys.exit(1)
```

---

## 16. Conclusion

### 16.1 Summary

ZIT represents a fundamental reimagining of version control built on three core principles:

1. **Line-level content addressing:** Files are decomposed into individually-addressed lines, enabling maximum deduplication and eliminating the need for delta compression

2. **Immutable objects everywhere:** All content is write-once, enabling aggressive caching, simple replication, and zero-cost branching

3. **Client-side computation:** Heavy work (hashing, tree construction) happens on clients, reducing server CPU requirements to near-zero

The system achieves its performance characteristics through architectural choices rather than implementation optimizations. By storing files as ordered lists of line hashes (`{hash}.lines`), all operations reduce to hash lookups and array comparisons.

### 16.2 Key Benefits

**For operators:**
- Minimal server CPU requirements
- Simple storage model (append-only objects)
- Horizontal scalability through CDN
- Predictable cost structure

**For developers:**
- Familiar Git-like workflow
- Instant branch operations
- Fast diffs (array comparison)
- Optional ZFS benefits (COW, snapshots)

**For enterprises:**
- Private P2P distribution via TailScale
- Encrypted by default
- Fine-grained access control
- Audit trail through immutable objects

### 16.3 Trade-offs

**Benefits gained:**
- Elimination of server-side delta compression
- Global deduplication across repositories
- Near-zero server CPU for reads
- Simple, predictable operations

**Costs accepted:**
- Line granularity less efficient for binary files
- New tooling required (incompatible with Git)
- Storage overhead for hash lists
- Learning curve for operations teams

### 16.4 Suitable Use Cases

**Ideal for:**
- Source code repositories (high line-level duplication)
- Documentation (Markdown, etc.)
- Configuration files
- Large monorepos
- Organizations with multiple similar projects

**Less suitable for:**
- Large binary files (images, videos)
- Frequently-changing binary formats
- Single-line files
- Use cases requiring Git compatibility

### 16.5 Next Steps

**To validate this design:**

1. **Build proof-of-concept implementation**
   - Minimal client (provided in Section 15)
   - Simple server (S3 + lightweight API)
   - Test on real repositories

2. **Measure actual deduplication rates**
   - Scan existing Git repositories
   - Calculate line-level dedup ratios
   
2. **Measure actual deduplication rates** (continued)
   - Scan existing Git repositories
   - Calculate line-level dedup ratios
   - Identify most commonly duplicated lines
   - Compare storage requirements vs Git

3. **Benchmark performance**
   - Push operation latency
   - Pull operation latency
   - Diff operation speed
   - Server CPU utilization under load
   - CDN cache hit rates

4. **Validate assumptions**
   - Hash collision probability (theoretical vs practical)
   - Network overhead (multiple small requests vs single large)
   - Cache effectiveness (CDN hit rates)
   - Client-side hashing performance

5. **Iterate on design**
   - Optimize based on real-world data
   - Refine file formats
   - Improve client implementation
   - Enhance server scalability

---

## Appendix A: File Format Specifications

### A.1 Content Object

**Path:** `content/{BLAKE3_HASH}`

**Format:** Raw bytes

**Constraints:**
- No size limit enforced at format level
- Recommended maximum: 32KB per object
- No encoding normalization
- Preserves exact bytes including line terminators

**Example:**
```
Path: content/abc123def456...
Content: import React from 'react';\n
Size: 27 bytes
```

### A.2 Lines File

**Path:** `{FILE_HASH}.lines`

**Format:** Text file, UTF-8 encoded

**Structure:**
```
<BLAKE3_HASH_1>\n
<BLAKE3_HASH_2>\n
<BLAKE3_HASH_3>\n
...
<BLAKE3_HASH_N>\n
```

**Constraints:**
- Each line contains exactly 64 hexadecimal characters
- Lowercase hex digits [a-f0-9]
- Unix line endings (\n)
- No trailing newline after final hash
- Empty files represented as empty .lines file

**Example:**
```
abc123def456789abc123def456789abc123def456789abc123def456789abcd
def456789abc123def456789abc123def456789abc123def456789abc123def4
789abc123def456789abc123def456789abc123def456789abc123def456789a
```

**Validation:**
```python
def validate_lines_file(content: str) -> bool:
    if not content:  # Empty file is valid
        return True
    
    lines = content.split('\n')
    for line in lines:
        if len(line) != 64:
            return False
        if not all(c in '0123456789abcdef' for c in line):
            return False
    
    return True
```

### A.3 Tree File

**Path:** `{TREE_HASH}.tree`

**Format:** Text file, UTF-8 encoded

**Structure:**
```
<PATH_1>\t<HASH_1>\n
<PATH_2>\t<HASH_2>\n
<PATH_3>\t<HASH_3>\n
...
<PATH_N>\t<HASH_N>\n
```

**Constraints:**
- Entries sorted lexicographically by path
- Paths are relative to repository root
- Path separator: forward slash (/)
- Hash is either:
  - File hash (for files)
  - Tree hash (for subdirectories)
- Tab character (\t) as delimiter
- Unix line endings (\n)
- No trailing newline after final entry

**Example:**
```
.gitignore\tabc123def456789abc123def456789abc123def456789abc123def456789abcd
README.md\tdef456789abc123def456789abc123def456789abc123def456789abc123def4
src\ttree_789abc123def456789abc123def456789abc123def456789abc123def456
src/index.js\t123def456789abc123def456789abc123def456789abc123def456789abc1
src/utils.py\t456789abc123def456789abc123def456789abc123def456789abc123def45
```

**Validation:**
```python
def validate_tree_file(content: str) -> bool:
    if not content:
        return False
    
    lines = content.split('\n')
    previous_path = None
    
    for line in lines:
        parts = line.split('\t')
        if len(parts) != 2:
            return False
        
        path, hash = parts
        
        # Check hash format
        if len(hash) != 64:
            return False
        if not all(c in '0123456789abcdef' for c in hash):
            return False
        
        # Check path doesn't contain invalid characters
        if '\t' in path or '\n' in path:
            return False
        
        # Check lexicographic ordering
        if previous_path is not None and path <= previous_path:
            return False
        
        previous_path = path
    
    return True
```

### A.4 Reference

**Path:** `refs/{user}/{repo}/{branch}`

**Format:** Text file, UTF-8 encoded

**Structure:**
```
<TREE_HASH>\n
```

**Constraints:**
- Single line containing exactly 64 hexadecimal characters
- Must point to a valid tree hash
- Mutable (can be updated atomically)

**Example:**
```
tree_abc123def456789abc123def456789abc123def456789abc123def456789ab
```

---

## Appendix B: API Specification

### B.1 Content Operations

#### Upload Content
```http
PUT /api/content/{hash}
Content-Type: application/octet-stream
Authorization: Bearer {token}

{raw_bytes}

Response 201 Created:
{
  "hash": "abc123...",
  "size": 27
}

Response 400 Bad Request:
{
  "error": "Hash mismatch",
  "expected": "abc123...",
  "computed": "def456..."
}

Response 409 Conflict:
{
  "error": "Object already exists"
}
```

#### Fetch Content
```http
GET /api/content/{hash}

Response 200 OK:
Content-Type: application/octet-stream
ETag: {hash}
Cache-Control: public, max-age=31536000, immutable

{raw_bytes}

Response 404 Not Found:
{
  "error": "Object not found"
}
```

#### Check Content Existence
```http
HEAD /api/content/{hash}

Response 200 OK:
Content-Length: {size}
ETag: {hash}

Response 404 Not Found:
```

### B.2 Batch Operations

#### Check Multiple Hashes
```http
POST /api/check-hashes
Content-Type: application/json
Authorization: Bearer {token}

{
  "hashes": [
    "abc123...",
    "def456...",
    "ghi789..."
  ]
}

Response 200 OK:
{
  "missing": [
    "def456..."
  ],
  "existing": [
    "abc123...",
    "ghi789..."
  ]
}
```

#### Batch Upload
```http
POST /api/batch-upload
Content-Type: multipart/form-data
Authorization: Bearer {token}

--boundary
Content-Disposition: form-data; name="abc123..."; filename="abc123..."

{raw_bytes_1}
--boundary
Content-Disposition: form-data; name="def456..."; filename="def456..."

{raw_bytes_2}
--boundary--

Response 200 OK:
{
  "uploaded": [
    {"hash": "abc123...", "size": 27},
    {"hash": "def456...", "size": 42}
  ],
  "errors": []
}
```

### B.3 File Definition Operations

#### Upload Lines File
```http
PUT /api/lines/{hash}
Content-Type: text/plain
Authorization: Bearer {token}

abc123def456...
def456ghi789...
ghi789jkl012...

Response 201 Created:
{
  "hash": "xyz789...",
  "line_count": 3
}
```

#### Fetch Lines File
```http
GET /api/lines/{hash}

Response 200 OK:
Content-Type: text/plain
ETag: {hash}
Cache-Control: public, max-age=31536000, immutable

abc123def456...
def456ghi789...
ghi789jkl012...
```

### B.4 Tree Operations

#### Upload Tree
```http
PUT /api/trees/{hash}
Content-Type: text/plain
Authorization: Bearer {token}

README.md\tabc123...
src/index.js\txyz789...

Response 201 Created:
{
  "hash": "tree_abc123...",
  "entry_count": 2
}
```

#### Fetch Tree
```http
GET /api/trees/{hash}

Response 200 OK:
Content-Type: text/plain
ETag: {hash}
Cache-Control: public, max-age=31536000, immutable

README.md\tabc123...
src/index.js\txyz789...
```

### B.5 Reference Operations

#### Get Reference
```http
GET /api/refs/{user}/{repo}/{branch}

Response 200 OK:
Content-Type: text/plain
Cache-Control: public, max-age=60

tree_abc123def456...

Response 404 Not Found:
{
  "error": "Reference not found"
}
```

#### Update Reference (Compare-And-Swap)
```http
POST /api/refs/{user}/{repo}/{branch}
Content-Type: application/json
Authorization: Bearer {token}

{
  "old_hash": "tree_abc123...",
  "new_hash": "tree_def456...",
  "cas": true
}

Response 200 OK:
{
  "updated": true,
  "old_hash": "tree_abc123...",
  "new_hash": "tree_def456..."
}

Response 409 Conflict:
{
  "error": "CAS failed",
  "expected": "tree_abc123...",
  "actual": "tree_xyz789..."
}
```

#### Create Reference
```http
POST /api/refs/{user}/{repo}/{branch}
Content-Type: application/json
Authorization: Bearer {token}

{
  "new_hash": "tree_def456..."
}

Response 201 Created:
{
  "created": true,
  "hash": "tree_def456..."
}

Response 409 Conflict:
{
  "error": "Reference already exists"
}
```

#### Delete Reference
```http
DELETE /api/refs/{user}/{repo}/{branch}
Authorization: Bearer {token}

Response 200 OK:
{
  "deleted": true
}

Response 404 Not Found:
{
  "error": "Reference not found"
}
```

### B.6 Path-Based Access (VFS)

#### Get File by Path
```http
GET /{user}/{repo}/{branch}/{path}

Example:
GET /alice/myapp/main/src/index.js

Response 200 OK:
Content-Type: text/javascript
ETag: xyz789...
Cache-Control: public, max-age=3600

import React from 'react';
console.log('hello');

Response 404 Not Found:
{
  "error": "File not found"
}
```

#### List Directory
```http
GET /{user}/{repo}/{branch}/{directory_path}?list=true

Example:
GET /alice/myapp/main/src?list=true

Response 200 OK:
Content-Type: application/json

{
  "path": "src",
  "entries": [
    {
      "name": "index.js",
      "type": "file",
      "hash": "xyz789...",
      "size": 150
    },
    {
      "name": "utils",
      "type": "directory",
      "hash": "tree_abc123..."
    }
  ]
}
```

### B.7 Repository Metadata

#### Get Repository Info
```http
GET /api/repos/{user}/{repo}

Response 200 OK:
{
  "name": "myapp",
  "owner": "alice",
  "description": "My application",
  "branches": ["main", "develop"],
  "default_branch": "main",
  "created_at": "2024-01-01T00:00:00Z"
}
```

#### List User Repositories
```http
GET /api/users/{user}/repos

Response 200 OK:
{
  "repos": [
    {
      "name": "myapp",
      "description": "My application",
      "default_branch": "main"
    },
    {
      "name": "other-project",
      "description": "Another project",
      "default_branch": "main"
    }
  ]
}
```

---

## Appendix C: Configuration Examples

### C.1 Client Configuration

**~/.zitconfig**
```ini
[user]
    name = Alice Developer
    email = alice@example.com

[remote "origin"]
    url = https://zithub.com
    token = zt_abc123def456...

[core]
    backend = zfs
    compression = true
    
[cache]
    max_size = 10G
    location = ~/.zit/cache
```

### C.2 Server Configuration

**zithub.yaml**
```yaml
server:
  host: 0.0.0.0
  port: 8000
  workers: 4

storage:
  backend: s3
  bucket: zithub-content
  region: us-east-1
  
database:
  type: postgresql
  host: localhost
  port: 5432
  database: zithub_refs
  user: zithub
  password: ${DB_PASSWORD}

cdn:
  enabled: true
  provider: cloudfront
  distribution_id: E1234567890ABC

auth:
  enabled: true
  jwt_secret: ${JWT_SECRET}
  token_expiry: 24h

limits:
  max_object_size: 32MB
  max_batch_size: 1000
  rate_limit: 1000/hour

tailscale:
  enabled: false
  network_name: zithub-private
```

### C.3 Nginx Configuration

```nginx
upstream zithub_api {
    least_conn;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}

# Cache for immutable objects
proxy_cache_path /var/cache/nginx/zithub 
                 levels=1:2 
                 keys_zone=zithub_cache:100m 
                 max_size=10g 
                 inactive=365d;

server {
    listen 443 ssl http2;
    server_name zithub.com;
    
    ssl_certificate /etc/ssl/certs/zithub.crt;
    ssl_certificate_key /etc/ssl/private/zithub.key;
    
    # Content objects (immutable, cache forever)
    location ~ ^/api/content/([a-f0-9]{64})$ {
        proxy_pass http://zithub_api;
        proxy_cache zithub_cache;
        proxy_cache_key $uri;
        proxy_cache_valid 200 365d;
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header X-Cache-Status $upstream_cache_status;
    }
    
    # Lines and trees (immutable)
    location ~ ^/api/(lines|trees)/([a-f0-9]{64})(\.lines|\.tree)?$ {
        proxy_pass http://zithub_api;
        proxy_cache zithub_cache;
        proxy_cache_key $uri;
        proxy_cache_valid 200 365d;
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header X-Cache-Status $upstream_cache_status;
    }
    
    # References (mutable, short cache)
    location ~ ^/api/refs/ {
        proxy_pass http://zithub_api;
        proxy_cache zithub_cache;
        proxy_cache_key $uri;
        proxy_cache_valid 200 60s;
        add_header Cache-Control "public, max-age=60";
    }
    
    # All other API requests
    location /api/ {
        proxy_pass http://zithub_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # Path-based file access
    location / {
        proxy_pass http://zithub_api;
        proxy_cache zithub_cache;
        proxy_cache_key $uri;
        proxy_cache_valid 200 1h;
    }
}
```

### C.4 Docker Compose Setup

**docker-compose.yml**
```yaml
version: '3.8'

services:
  zithub-api:
    image: zithub/api:latest
    ports:
      - "8000:8000"
    environment:
      - DB_HOST=postgres
      - DB_PASSWORD=${DB_PASSWORD}
      - JWT_SECRET=${JWT_SECRET}
      - S3_BUCKET=zithub-content
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
    depends_on:
      - postgres
    restart: unless-stopped
    deploy:
      replicas: 3
  
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=zithub_refs
      - POSTGRES_USER=zithub
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
  
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/ssl:ro
      - nginx_cache:/var/cache/nginx
    depends_on:
      - zithub-api
    restart: unless-stopped

volumes:
  postgres_data:
  nginx_cache:
```

---

## Appendix D: Performance Testing

### D.1 Benchmark Suite

```python
#!/usr/bin/env python3
"""
ZIT Performance Benchmark Suite
"""

import time
import os
import random
import string
from pathlib import Path
from zit_client import ZitClient

class ZitBenchmark:
    def __init__(self, repo_path: str, remote_url: str):
        self.client = ZitClient(repo_path, remote_url)
        self.results = {}
    
    def generate_test_file(self, lines: int, avg_line_length: int = 80):
        """Generate a test file with random content"""
        content = []
        for _ in range(lines):
            line_length = random.randint(
                avg_line_length // 2, 
                avg_line_length * 2
            )
            line = ''.join(random.choices(
                string.ascii_letters + string.digits, 
                k=line_length
            ))
            content.append(line)
        return '\n'.join(content)
    
    def benchmark_hash_performance(self, iterations: int = 1000):
        """Benchmark line hashing performance"""
        test_lines = [
            b"import React from 'react';\n",
            b"const x = 42;\n",
            b"// This is a comment\n",
            b"function hello() { return 'world'; }\n"
        ]
        
        start = time.time()
        for _ in range(iterations):
            for line in test_lines:
                self.client.hash_line(line)
        end = time.time()
        
        total_lines = iterations * len(test_lines)
        lines_per_second = total_lines / (end - start)
        
        self.results['hash_performance'] = {
            'lines_per_second': lines_per_second,
            'seconds_per_million_lines': 1_000_000 / lines_per_second
        }
    
    def benchmark_file_hashing(self, file_sizes: list = [100, 1000, 10000]):
        """Benchmark complete file hashing"""
        results = {}
        
        for size in file_sizes:
            content = self.generate_test_file(size)
            test_file = Path(f'/tmp/test_{size}.txt')
            test_file.write_text(content)
            
            start = time.time()
            file_hash, line_hashes = self.client.hash_file(test_file)
            end = time.time()
            
            results[f'{size}_lines'] = {
                'duration_ms': (end - start) * 1000,
                'lines_per_second': size / (end - start)
            }
            
            test_file.unlink()
        
        self.results['file_hashing'] = results
    
    def benchmark_push(self, file_count: int = 100, lines_per_file: int = 500):
        """Benchmark push operation"""
        # Create test repository
        repo_path = Path('/tmp/zit_bench_repo')
        repo_path.mkdir(exist_ok=True)
        
        # Generate test files
        for i in range(file_count):
            test_file = repo_path / f'file_{i}.txt'
            content = self.generate_test_file(lines_per_file)
            test_file.write_text(content)
        
        # Benchmark push
        start = time.time()
        self.client.push('main')
        end = time.time()
        
        total_lines = file_count * lines_per_file
        
        self.results['push'] = {
            'duration_seconds': end - start,
            'files': file_count,
            'total_lines': total_lines,
            'lines_per_second': total_lines / (end - start)
        }
        
        # Cleanup
        import shutil
        shutil.rmtree(repo_path)
    
    def benchmark_pull(self):
        """Benchmark pull operation"""
        repo_path = Path('/tmp/zit_bench_pull')
        repo_path.mkdir(exist_ok=True)
        
        start = time.time()
        self.client.pull('main')
        end = time.time()
        
        self.results['pull'] = {
            'duration_seconds': end - start
        }
        
        # Cleanup
        import shutil
        shutil.rmtree(repo_path)
    
    def benchmark_diff(self, file_lines: int = 1000, change_percentage: float = 0.1):
        """Benchmark diff operation"""
        # Create two versions of a file
        content_v1 = self.generate_test_file(file_lines)
        
        # Modify ~10% of lines
        lines = content_v1.split('\n')
        changes = int(file_lines * change_percentage)
        for _ in range(changes):
            idx = random.randint(0, file_lines - 1)
            lines[idx] = ''.join(random.choices(string.ascii_letters, k=80))
        content_v2 = '\n'.join(lines)
        
        # Hash both versions
        test_file = Path('/tmp/test_diff.txt')
        
        test_file.write_text(content_v1)
        hash_v1, _ = self.client.hash_file(test_file)
        
        test_file.write_text(content_v2)
        hash_v2, _ = self.client.hash_file(test_file)
        
        # Benchmark diff (array comparison)
        lines_v1 = self.client.cache_dir / 'lines' / f'{hash_v1}.lines'
        lines_v2 = self.client.cache_dir / 'lines' / f'{hash_v2}.lines'
        
        hashes_v1 = lines_v1.read_text().split('\n')
        hashes_v2 = lines_v2.read_text().split('\n')
        
        start = time.time()
        differences = []
        for i, (h1, h2) in enumerate(zip(hashes_v1, hashes_v2)):
            if h1 != h2:
                differences.append(i)
        end = time.time()
        
        self.results['diff'] = {
            'duration_ms': (end - start) * 1000,
            'file_lines': file_lines,
            'differences_found': len(differences),
            'lines_per_second': file_lines / (end - start)
        }
        
        test_file.unlink()
    
    def run_all(self):
        """Run all benchmarks"""
        print("Running ZIT benchmarks...")
        
        print("1. Hash performance...")
        self.benchmark_hash_performance()
        
        print("2. File hashing...")
        self.benchmark_file_hashing()
        
        print("3. Diff performance...")
        self.benchmark_diff()
        
        # Note: Push/pull require server setup
        # print("4. Push performance...")
        # self.benchmark_push()
        
        # print("5. Pull performance...")
        # self.benchmark_pull()
        
        return self.results
    
    def print_results(self):
        """Print benchmark results"""
        import json
        print("\n" + "="*60)
        print("BENCHMARK RESULTS")
        print("="*60)
        print(json.dumps(self.results, indent=2))


if __name__ == '__main__':
    benchmark = ZitBenchmark(
        repo_path='/tmp/zit_bench',
        remote_url='http://localhost:8000'
    )
    
    results = benchmark.run_all()
    benchmark.print_results()
```

### D.2 Load Testing

```python
#!/usr/bin/env python3
"""
ZIT Load Testing
"""

import concurrent.futures
import time
import requests
from typing import List, Dict

class ZitLoadTest:
    def __init__(self, base_url: str, num_clients: int = 100):
        self.base_url = base_url
        self.num_clients = num_clients
        self.results = []
    
    def simulate_push(self, client_id: int) -> Dict:
        """Simulate a push operation"""
        start = time.time()
        
        # 1. Check hashes
        hashes = [f"{'0' * 63}{i:x}" for i in range(100)]
        response = requests.post(
            f"{self.base_url}/api/check-hashes",
            json={"hashes": hashes}
        )
        check_time = time.time() - start
        
        # 2. Upload content
        upload_start = time.time()
        for hash in response.json().get('missing', []):
            requests.put(
                f"{self.base_url}/api/content/{hash}",
                data=b"test content"
            )
        upload_time = time.time() - upload_start
        
        # 3. Update ref
        ref_start = time.time()
        requests.post(
            f"{self.base_url}/api/refs/test/repo{client_id}/main",
            json={"new_hash": f"tree_{'0' * 56}{client_id:x}"}
        )
        ref_time = time.time() - ref_start
        
        total_time = time.time() - start
        
        return {
            'client_id': client_id,
            'total_time': total_time,
            'check_time': check_time,
            'upload_time': upload_time,
            'ref_time': ref_time
        }
    
    def simulate_pull(self, client_id: int) -> Dict:
        """Simulate a pull operation"""
        start = time.time()
        
        # 1. Fetch ref
        response = requests.get(
            f"{self.base_url}/api/refs/test/repo{client_id}/main"
        )
        ref_time = time.time() - start
        
        # 2. Fetch tree
        tree_start = time.time()
        tree_hash = response.text.strip()
        requests.get(f"{self.base_url}/api/trees/{tree_hash}")
        tree_time = time.time() - tree_start
        
        # 3. Fetch some content
        content_start = time.time()
        for i in range(10):
            hash = f"{'0' * 63}{i:x}"
            requests.get(f"{self.base_url}/api/content/{hash}")
        content_time = time.time() - content_start
        
        total_time = time.time() - start
        
        return {
            'client_id': client_id,
            'total_time': total_time,
            'ref_time': ref_time,
            'tree_time': tree_time,
            'content_time': content_time
        }
    
    def run_concurrent_pushes(self) -> List[Dict]:
        """Run concurrent push operations"""
        print(f"Running {self.num_clients} concurrent pushes...")
        
        with concurrent.futures.ThreadPoolExecutor(max_workers=self.num_clients) as executor:
            futures = [
                executor.submit(self.simulate_push, i) 
                for i in range(self.num_clients)
            ]
            results = [f.result() for f in concurrent.futures.as_completed(futures)]
        
        return results
    
    def run_concurrent_pulls(self) -> List[Dict]:
        """Run concurrent pull operations"""
        print(f"Running {self.num_clients} concurrent pulls...")
        
        with concurrent.futures.ThreadPoolExecutor(max_workers=self.num_clients) as executor:
            futures = [
                executor.submit(self.simulate_pull, i) 
                for i in range(self.num_clients)
            ]
            results = [f.result() for f in concurrent.futures.as_completed(futures)]
        
        return results
    
    def print_stats(self, results: List[Dict], operation: str):
        """Print statistics from results"""
        total_times = [r['total_time'] for r in results]
        
        print(f"\n{operation} Results ({len(results)} operations):")
        print(f"  Mean: {sum(total_times) / len(total_times):.3f}s")
        print(f"  Min: {min(total_times):.3f}s")
        print(f"  Max: {max(total_times):.3f}s")
        print(f"  P50: {sorted(total_times)[len(total_times)//2]:.3f}s")
        print(f"  P95: {sorted(total_times)[int(len(total_times)*0.95)]:.3f}s")
        print(f"  P99: {sorted(total_times)[int(len(total_times)*0.99)]:.3f}s")


if __name__ == '__main__':
    load_test = ZitLoadTest(
        base_url='http://localhost:8000',
        num_clients=100
    )
    
    push_results = load_test.run_concurrent_pushes()
    load_test.print_stats(push_results, "PUSH")
    
    pull_results = load_test.run_concurrent_pulls()
    load_test.print_stats(pull_results, "PULL")
```

---

## Appendix E: Security Considerations

### E.1 Threat Model

**Assumptions:**
- Attacker cannot break BLAKE3 (find collisions or preimages)
- TLS/WireGuard provides confidentiality and integrity in transit
- Storage backend (S3/ZFS) is trusted
- Server operators are trusted (for non-encrypted deployments)

**Threats considered:**
1. Unauthorized read access to private repositories
2. Unauthorized write access to repositories
3. Denial of service through resource exhaustion
4. Content tampering
5. Reference manipulation
6. Information leakage through timing or deduplication

### E.1 Threat Model (continued)

**Threats considered:**
1. Unauthorized read access to private repositories
2. Unauthorized write access to repositories
3. Denial of service through resource exhaustion
4. Content tampering
5. Reference manipulation
6. Information leakage through timing or deduplication

**Threats explicitly not addressed:**
- Server compromise (trusted server model)
- End-to-end encryption (incompatible with deduplication)
- Side-channel attacks on hash computation
- Quantum computing attacks on BLAKE3

### E.2 Authentication and Authorization

**Token-based authentication:**
```python
def authenticate_request(request):
    """Validate bearer token"""
    auth_header = request.headers.get('Authorization')
    if not auth_header or not auth_header.startswith('Bearer '):
        return None
    
    token = auth_header[7:]  # Remove 'Bearer '
    
    # Verify JWT signature
    try:
        payload = jwt.decode(token, JWT_SECRET, algorithms=['HS256'])
        return payload['user']
    except jwt.InvalidTokenError:
        return None

def authorize_read(user, repo):
    """Check if user can read repository"""
    # Public repos: anyone can read
    if is_public_repo(repo):
        return True
    
    # Private repos: check ACL
    return db.check_permission(user, repo, 'read')

def authorize_write(user, repo, branch):
    """Check if user can write to branch"""
    # Check repository write permission
    if not db.check_permission(user, repo, 'write'):
        return False
    
    # Check branch protection rules
    if is_protected_branch(repo, branch):
        return db.check_permission(user, repo, 'admin')
    
    return True
```

**Permission levels:**
- `read`: Can clone, pull, view files
- `write`: Can push to non-protected branches
- `admin`: Can push to protected branches, modify settings

### E.3 Rate Limiting

**Per-user limits:**
```python
from datetime import datetime, timedelta
from collections import defaultdict

class RateLimiter:
    def __init__(self):
        self.requests = defaultdict(list)
    
    def check_limit(self, user, limit=1000, window_hours=1):
        """Check if user is within rate limit"""
        now = datetime.now()
        cutoff = now - timedelta(hours=window_hours)
        
        # Clean old requests
        self.requests[user] = [
            ts for ts in self.requests[user] 
            if ts > cutoff
        ]
        
        # Check limit
        if len(self.requests[user]) >= limit:
            return False
        
        # Record request
        self.requests[user].append(now)
        return True

@app.before_request
def rate_limit_check():
    user = authenticate_request(request)
    if not user:
        return  # Anonymous requests have lower limit
    
    if not rate_limiter.check_limit(user, limit=1000, window_hours=1):
        return {"error": "Rate limit exceeded"}, 429
```

**Object size limits:**
```python
MAX_CONTENT_SIZE = 32 * 1024 * 1024  # 32MB
MAX_LINES_FILE_SIZE = 10 * 1024 * 1024  # 10MB
MAX_TREE_SIZE = 10 * 1024 * 1024  # 10MB

@app.before_request
def check_content_length():
    if request.content_length and request.content_length > MAX_CONTENT_SIZE:
        return {"error": "Content too large"}, 413
```

### E.4 Input Validation

**Hash validation:**
```python
import re

HASH_PATTERN = re.compile(r'^[a-f0-9]{64}$')

def validate_hash(hash_str):
    """Validate hash format"""
    if not hash_str:
        raise ValueError("Hash cannot be empty")
    
    if not HASH_PATTERN.match(hash_str):
        raise ValueError("Invalid hash format")
    
    return hash_str

def validate_and_normalize_hash(hash_str):
    """Validate and normalize hash"""
    # Convert to lowercase
    hash_str = hash_str.lower()
    
    # Validate format
    return validate_hash(hash_str)
```

**Path validation:**
```python
def validate_path(path):
    """Validate file path"""
    # Reject absolute paths
    if path.startswith('/'):
        raise ValueError("Absolute paths not allowed")
    
    # Reject parent directory references
    if '..' in path.split('/'):
        raise ValueError("Parent directory references not allowed")
    
    # Reject hidden files (optional)
    parts = path.split('/')
    if any(part.startswith('.') for part in parts):
        raise ValueError("Hidden files not allowed")
    
    # Check length
    if len(path) > 4096:
        raise ValueError("Path too long")
    
    return path
```

**Content validation:**
```python
def validate_content_object(hash, content):
    """Validate content matches hash"""
    computed = blake3(content).hexdigest()
    if computed != hash:
        raise ValueError(f"Hash mismatch: expected {hash}, got {computed}")

def validate_lines_file(content):
    """Validate .lines file format"""
    if not content:
        return  # Empty file is valid
    
    lines = content.decode('utf-8').split('\n')
    for i, line in enumerate(lines):
        if not HASH_PATTERN.match(line):
            raise ValueError(f"Invalid hash at line {i}: {line}")

def validate_tree_file(content):
    """Validate .tree file format"""
    if not content:
        raise ValueError("Empty tree not allowed")
    
    lines = content.decode('utf-8').split('\n')
    previous_path = None
    
    for i, line in enumerate(lines):
        parts = line.split('\t')
        if len(parts) != 2:
            raise ValueError(f"Invalid tree entry at line {i}")
        
        path, hash = parts
        
        # Validate path
        validate_path(path)
        
        # Validate hash
        validate_hash(hash)
        
        # Check ordering
        if previous_path and path <= previous_path:
            raise ValueError(f"Tree entries not sorted at line {i}")
        
        previous_path = path
```

### E.5 Denial of Service Protection

**Resource limits:**
```python
# Limit concurrent requests per user
@app.before_request
def check_concurrent_requests():
    user = get_current_user()
    concurrent = get_concurrent_request_count(user)
    
    if concurrent > MAX_CONCURRENT_REQUESTS:
        return {"error": "Too many concurrent requests"}, 429

# Limit batch operation sizes
@app.route('/api/check-hashes', methods=['POST'])
def check_hashes():
    data = request.json
    hashes = data.get('hashes', [])
    
    if len(hashes) > MAX_BATCH_SIZE:
        return {"error": f"Batch size exceeds limit of {MAX_BATCH_SIZE}"}, 400
    
    # Process batch...

# Timeout long-running operations
@app.route('/api/content/<hash>')
@timeout(seconds=30)
def get_content(hash):
    # Fetch with timeout...
```

**Deduplication side-channel protection:**
```python
def check_hash_exists_safe(hash):
    """
    Check if hash exists with timing protection.
    Always takes consistent time regardless of result.
    """
    import time
    start = time.time()
    
    exists = s3_client.head_object(
        Bucket='zithub-content',
        Key=f'content/{hash}'
    )
    
    # Add random delay to prevent timing attacks
    elapsed = time.time() - start
    target_time = 0.1  # 100ms minimum
    if elapsed < target_time:
        time.sleep(target_time - elapsed + random.uniform(0, 0.01))
    
    return exists
```

Note: This timing protection is only relevant if deduplication behavior could leak information about private repository contents. In practice, the public/private separation at the storage level may be preferable.

### E.6 Audit Logging

**Comprehensive audit trail:**
```python
def log_action(user, action, resource, details=None):
    """Log security-relevant actions"""
    log_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'user': user,
        'action': action,
        'resource': resource,
        'details': details or {},
        'ip_address': request.remote_addr,
        'user_agent': request.headers.get('User-Agent')
    }
    
    # Write to audit log
    audit_logger.info(json.dumps(log_entry))
    
    # Store in database for queries
    db.audit_log.insert(log_entry)

# Log all write operations
@app.route('/api/content/<hash>', methods=['PUT'])
def put_content(hash):
    user = get_current_user()
    
    # Verify and store...
    
    log_action(
        user=user,
        action='content.upload',
        resource=f'content/{hash}',
        details={'size': len(request.data)}
    )

# Log authentication failures
@app.before_request
def log_auth():
    user = authenticate_request(request)
    if not user and request.headers.get('Authorization'):
        log_action(
            user='anonymous',
            action='auth.failed',
            resource=request.path
        )
```

### E.7 Secure Deployment Checklist

**Infrastructure:**
- [ ] TLS 1.3 enabled with modern cipher suites
- [ ] HTTPS Strict Transport Security (HSTS) enabled
- [ ] Certificate pinning for API clients
- [ ] DDoS protection (CloudFlare, AWS Shield, etc.)
- [ ] WAF rules for common attacks
- [ ] Network segmentation (API servers, database, storage)

**Application:**
- [ ] All inputs validated and sanitized
- [ ] Rate limiting enabled
- [ ] Authentication required for private operations
- [ ] Authorization checks on all endpoints
- [ ] Audit logging enabled
- [ ] Error messages don't leak sensitive information
- [ ] Secrets stored in environment variables or secret management system

**Storage:**
- [ ] S3 bucket policies restrict public access appropriately
- [ ] S3 versioning enabled for disaster recovery
- [ ] Database backups automated and tested
- [ ] Encryption at rest enabled (S3, RDS)
- [ ] Access logs enabled and monitored

**Monitoring:**
- [ ] Failed authentication attempts monitored
- [ ] Unusual activity patterns detected
- [ ] Resource exhaustion alerts configured
- [ ] Audit logs reviewed regularly
- [ ] Security patches applied promptly

---

## Appendix F: Cost Analysis

### F.1 Storage Costs

**S3 storage pricing (example: AWS us-east-1):**
```
Standard storage: $0.023/GB/month
Requests:
  PUT: $0.005 per 1,000 requests
  GET: $0.0004 per 1,000 requests
Data transfer:
  First 10TB out: $0.09/GB
  Next 40TB out: $0.085/GB
```

**Example calculation for 10TB repository content:**

Assumptions:
- 10TB raw content
- 80% deduplication → 2TB unique content stored
- 100M unique line objects
- 10M file definitions
- 1M tree objects

```
Storage:
  Content: 2TB × $0.023 = $46/month
  File definitions: 320GB × $0.023 = $7.36/month
  Trees: 100GB × $0.023 = $2.30/month
  Total storage: $55.66/month

Requests (estimated monthly):
  Content writes: 1M PUT × $0.005/1000 = $5
  Content reads: 100M GET × $0.0004/1000 = $40
  Total requests: $45/month

Data transfer:
  Egress: 5TB × $0.09 = $450/month

Total monthly cost: $550.66
```

Compare to Git (no deduplication):
```
Storage: 10TB × $0.023 = $230/month
(Plus additional server costs for delta compression)
```

### F.2 Compute Costs

**ZIT server (minimal):**
```
EC2 t3.medium (2 vCPU, 4GB RAM): $30/month
RDS PostgreSQL t3.small: $25/month
Total: $55/month
```

**ZIT server (production):**
```
3× EC2 c6i.2xlarge (8 vCPU, 16GB RAM): $365/month
RDS PostgreSQL m6g.large (Multi-AZ): $280/month
CloudFront CDN: $150/month (varies with traffic)
Total: $795/month
```

**Traditional Git server equivalent:**
```
10× EC2 c6i.4xlarge (16 vCPU, 32GB RAM): $2,440/month
  (More CPU needed for delta compression)
RDS PostgreSQL m6g.xlarge (Multi-AZ): $560/month
Load balancer: $50/month
Total: $3,050/month
```

**Cost reduction: ~74% lower compute costs**

### F.3 Bandwidth Costs

**With CDN (95% cache hit rate):**
```
Total requests: 1 billion/month
Origin requests: 50 million (5%)
CDN requests: 950 million (95%)

Origin bandwidth: 500GB × $0.09 = $45
CDN bandwidth: 9.5TB × $0.02 = $190
  (CloudFront pricing)

Total: $235/month
```

**Without CDN (direct serving):**
```
Total bandwidth: 10TB × $0.09 = $900/month
```

**Savings with CDN: 74% reduction**

### F.4 Total Cost of Ownership Comparison

**ZIT (for 500M repos, 10TB content):**
```
Storage (S3): $556/month
Compute: $795/month
CDN: $235/month
Database: (included in compute)
Total: $1,586/month
```

**Traditional Git hosting:**
```
Storage (EBS): $2,300/month (10TB × $0.10/GB IOPS SSD)
Compute: $3,050/month
Load balancer: $50/month
Total: $5,400/month
```

**Annual savings: $45,768 (71% reduction)**

Note: These are illustrative calculations. Actual costs depend on:
- Traffic patterns
- Geographic distribution
- Repository characteristics
- Deduplication rates achieved
- Reserved instance pricing
- Volume discounts

### F.5 Break-Even Analysis

**When does ZIT become cost-effective?**

Fixed costs:
- Development: $100K - $300K (one-time)
- Migration tooling: $20K - $50K (one-time)

Ongoing savings per month: ~$3,800

Break-even point:
- Conservative: $300K / $3,800 = 79 months (6.6 years)
- Optimistic: $100K / $3,800 = 26 months (2.2 years)

Additional factors favoring ZIT:
- Scales better (savings increase with size)
- Less operational complexity
- Better performance (developer productivity)
- Novel capabilities (vulnerability scanning, etc.)

---

## Appendix G: Migration Guide

### G.1 Pre-Migration Assessment

**Repository analysis:**
```bash
#!/bin/bash
# Analyze repositories for ZIT migration

echo "Analyzing repositories for ZIT migration..."

for repo in $(find . -name ".git" -type d); do
    repo_path=$(dirname "$repo")
    echo "Repository: $repo_path"
    
    # Count commits
    commits=$(cd "$repo_path" && git rev-list --all --count)
    echo "  Commits: $commits"
    
    # Count files
    files=$(cd "$repo_path" && git ls-files | wc -l)
    echo "  Files: $files"
    
    # Calculate size
    size=$(du -sh "$repo_path/.git" | cut -f1)
    echo "  Size: $size"
    
    # Count lines
    lines=$(cd "$repo_path" && git ls-files | xargs wc -l 2>/dev/null | tail -1 | awk '{print $1}')
    echo "  Lines: $lines"
    
    # Estimate deduplication potential
    unique_lines=$(cd "$repo_path" && git ls-files | xargs cat 2>/dev/null | sort -u | wc -l)
    if [ "$lines" -gt 0 ]; then
        dedup=$(echo "scale=2; 100 - ($unique_lines * 100 / $lines)" | bc)
        echo "  Estimated dedup: ${dedup}%"
    fi
    
    echo ""
done
```

### G.2 Migration Process

**Step 1: Setup ZIT infrastructure**
```bash
# Deploy ZitHub server
docker-compose up -d

# Verify server is running
curl http://localhost:8000/api/health

# Create user account
zit auth register --username alice --email alice@example.com
```

**Step 2: Convert repositories**
```python
#!/usr/bin/env python3
"""
Convert Git repository to ZIT format
"""

import subprocess
import tempfile
from pathlib import Path
from zit_client import ZitClient

def convert_git_to_zit(git_repo_path, zit_remote_url):
    """Convert a Git repository to ZIT"""
    
    print(f"Converting {git_repo_path}...")
    
    # Initialize ZIT client
    zit_client = ZitClient(git_repo_path, zit_remote_url)
    zit_client.init()
    
    # Get all branches
    result = subprocess.run(
        ['git', 'branch', '-r'],
        cwd=git_repo_path,
        capture_output=True,
        text=True
    )
    branches = [b.strip().replace('origin/', '') 
                for b in result.stdout.split('\n') if b.strip()]
    
    print(f"Found branches: {branches}")
    
    # Convert each branch
    for branch in branches:
        print(f"  Converting branch: {branch}")
        
        # Checkout branch
        subprocess.run(
            ['git', 'checkout', branch],
            cwd=git_repo_path,
            capture_output=True
        )
        
        # Push to ZIT
        zit_client.push(branch)
        
        print(f"  ✓ Branch {branch} converted")
    
    print(f"✓ Repository converted successfully")

if __name__ == '__main__':
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: convert_git_to_zit.py <git_repo_path> [zit_remote_url]")
        sys.exit(1)
    
    git_repo = sys.argv[1]
    remote_url = sys.argv[2] if len(sys.argv) > 2 else 'http://localhost:8000'
    
    convert_git_to_zit(git_repo, remote_url)
```

**Step 3: Validate conversion**
```bash
# Clone from ZIT
zit clone http://localhost:8000/alice/myapp /tmp/test-clone

# Compare with original
diff -r /path/to/original /tmp/test-clone

# Verify all branches present
cd /tmp/test-clone
zit branch --list
```

**Step 4: Parallel operation**
```bash
# Continue using Git during transition
git push origin main

# Sync to ZIT automatically
zit sync-from-git --source origin --dest zithub
```

**Step 5: Cutover**
```bash
# Update CI/CD to use ZIT
sed -i 's|git clone|zit clone|g' .github/workflows/*.yml

# Update developer documentation
echo "Repository has migrated to ZIT" > MIGRATION.md

# Archive Git repository (read-only)
git config --bool core.bare true
```

### G.3 Rollback Plan

**If migration encounters issues:**

```bash
# Keep Git repository active
# ZIT operates alongside Git

# Developers can continue using Git
git push origin main

# ZIT can be re-synced from Git
zit sync-from-git --force

# Or: abandon ZIT migration
# Delete ZIT repositories
# Continue with Git as before
```

**Data safety:**
- Git repository remains untouched during migration
- ZIT creates new content store (no destructive operations)
- Can operate both systems in parallel indefinitely
- Rollback is simply reverting to Git URLs

---

## Appendix H: Frequently Asked Questions

### H.1 General Questions

**Q: Why not just use Git?**

A: Git is excellent for many use cases. ZIT is designed for scenarios where:
- Global deduplication provides significant storage savings
- Server-side CPU cost is a concern
- Line-level operations are beneficial
- New capabilities (vulnerability scanning, etc.) are desired

**Q: Is ZIT compatible with Git?**

A: No. ZIT uses a different storage model and protocol. However:
- Conversion tools can migrate Git→ZIT
- Workflows are similar (clone, push, pull, branch)
- A bridge could enable interoperability if needed

**Q: What about binary files?**

A: Line-level deduplication works poorly for binary files. Options:
- Store binary files as single objects (no line splitting)
- Use Git LFS-style external storage
- Accept lower deduplication rates for binary-heavy repositories

**Q: Can I self-host?**

A: Yes. ZIT is designed to be self-hostable:
- Reference implementation provided
- Standard components (S3, PostgreSQL)
- Docker deployment available

### H.2 Technical Questions

**Q: What if two lines have the same hash (collision)?**

A: BLAKE3 has 256-bit output. Collision probability is:
- 2^-256 for random data
- Computationally infeasible to find collisions
- More likely: cosmic ray bit flip, hardware failure

If a collision somehow occurred, both lines would reference the same content, causing data corruption. This is detected by ZFS/S3 checksums at storage level.

**Q: How do you handle large files?**

A: Current approach:
- Split on newlines regardless of file size
- Very long lines (>32KB) should be chunked

Future optimization:
- Adaptive chunking for better binary file handling
- Configurable line/chunk boundaries

**Q: What about file permissions and modes?**

A: Current specification:
- Trees store path→hash mappings only
- File metadata not preserved

Future enhancement:
- Extend tree format: `path\tmode\thash`
- Store mode bits (755, 644, etc.)
- Preserve executable flag

**Q: How does branching work without commits?**

A: ZIT branches are just named pointers to tree hashes:
- Branch = reference to a tree
- "Commit" = update reference to new tree
- History is implicit (sequence of tree changes)

Optional future addition:
- Commit objects linking trees
- Store commit messages, authors, timestamps
- Enable traditional commit graph

**Q: Can I sign commits?**

A: Not in current specification. Future enhancement:
- Sign tree hashes with GPG/SSH keys
- Store signatures alongside references
- Verify signatures on pull

**Q: What about merge conflicts?**

A: Current approach:
- Client-side three-way merge using hash comparison
- Conflicts presented to user for resolution

Process:
1. Load base, ours, theirs line hash lists
2. Compare hashes position-by-position
3. Auto-merge where hashes agree
4. Mark conflicts where both changed

### H.3 Operational Questions

**Q: How do I backup a ZIT server?**

A: Two components:
1. Content store (S3): Enable versioning and cross-region replication
2. References (PostgreSQL): Standard database backups

Both can be backed up independently. Content is immutable and easy to replicate.

**Q: What happens if S3 goes down?**

A: Reads fail if content not cached. Mitigation:
- CloudFront CDN provides availability layer
- Multi-region S3 replication
- Local gateway nodes cache content (TailScale mode)

**Q: How do I delete sensitive data?**

A: ZIT uses immutable objects, so deletion is complex:
1. Remove references to sensitive trees
2. Run garbage collection (removes unreferenced objects)
3. Objects remain in S3 for grace period
4. After grace period, permanently deleted

Note: Deduplication means content might be referenced by other repositories.

**Q: Can I migrate back to Git?**

A: Yes, with effort:
1. Clone all repositories from ZIT
2. Initialize Git repositories
3. Commit current state
4. History is lost (ZIT doesn't store commit graph)

Better approach:
- Keep Git repositories during transition
- Sync ZIT→Git automatically
- Full reversibility maintained

**Q: What about CI/CD integration?**

A: Replace `git` commands with `zit`:
```yaml
# Before
- run: git clone https://github.com/user/repo
- run: git checkout main

# After
- run: zit clone https://zithub.com/user/repo
- run: zit checkout main
```

Most CI/CD systems work with any VCS that provides clone/checkout/pull commands.

### H.4 Performance Questions

**Q: Is line hashing fast enough?**

A: Yes. BLAKE3 hashes at ~10GB/s single-threaded:
- 1000-line file: <1ms to hash
- 100,000-line file: <100ms to hash
- Parallelizable across files

Bottleneck is typically network/storage, not hashing.

**Q: What about initial clone of large repositories?**

A: First clone requires downloading all content:
- Same as Git (must transfer all data)
- Subsequent operations are incremental
- CDN caching helps for popular repositories
- TailScale peer caching helps for teams

**Q: How fast is diff?**

A: Very fast:
- O(n) hash array comparison
- No Myers algorithm needed
- Typically <1ms for 1000-line files
- Only fetch content for changed lines

**Q: Can it handle monorepos?**

A: Yes, potentially better than Git:
- Deduplication across entire monorepo
- Fast diffs (hash comparison)
- Instant branching (tree copying)
- No pack file regeneration

Bottleneck would be network bandwidth for initial clone.

---

## Appendix I: Glossary

**BLAKE3:** Fast cryptographic hash function producing 256-bit output. Used for content addressing in ZIT.

**Branch:** Named reference to a tree hash. Represents a line of development.

**Content object:** Immutable storage of a single line's content, addressed by BLAKE3 hash.

**Content-addressed storage:** Storage system where data is retrieved by content hash rather than location.

**Copy-on-write (COW):** Storage technique where data is shared until modified, then copied.

**Deduplication:** Elimination of duplicate data by storing unique content once.

**File definition (.lines file):** Ordered list of line hashes representing a file's structure.

**Hash:** Fixed-size cryptographic fingerprint of data. Used as unique identifier.

**Immutable object:** Data that cannot be modified after creation.

**Line:** Sequence of bytes terminated by newline (or EOF). Atomic unit in ZIT.

**Reference (ref):** Mutable pointer from name (branch/tag) to tree hash.

**Repository:** Collection of files, branches, and history managed by ZIT.

**TailScale:** VPN service providing encrypted peer-to-peer mesh networks.

**Tree object:** Directory structure mapping paths to file/subtree hashes.

**VFS (Virtual File System):** Software layer translating filesystem operations to content-addressed lookups.

**ZFS:** Advanced filesystem with copy-on-write, snapshots, and integrity checking.

---

## Appendix J: References and Further Reading

### J.1 Related Systems

**Git:**
- Official Documentation: https://git-scm.com/doc
- Pro Git Book: https://git-scm.com/book
- Git Internals: https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain

**Content-Addressed Storage Systems:**
- IPFS: https://ipfs.io/
- Perkeep: https://perkeep.org/
- Git-annex: https://git-annex.branchable.com/

**Hash Functions:**
- BLAKE3 Paper: https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf
- BLAKE3 Implementation: https://github.com/BLAKE3-team/BLAKE3

**ZFS:**
- OpenZFS Documentation: https://openzfs.github.io/openzfs-docs/
- ZFS Administration Guide

**Networking:**
- TailScale Documentation: https://tailscale.com/kb/
- WireGuard: https://www.wireguard.com/

### J.2 Academic Papers

1. "Content-Defined Chunking for Deduplication" - Various authors
2. "Approximate Caching for Efficiently Deduplicating Data" - Kulkarni et al.
3. "Fingerprinting by Random Polynomials" - Rabin (1981)
4. "The Git Object Model" - Torvalds et al.

### J.3 Industry Reports

1. Dropbox: "How We've Scaled Dropbox" (deduplication architecture)
2. GitHub: "How We Ship GitHub" (Git hosting at scale)
3. Microsoft: "Git Virtual File System" (GVFS)

---

## Document Revision History

**Version 1.0** - January 2026
- Initial specification
- Core architecture and file formats
- Reference implementation
- Performance analysis
- Security considerations

**Future Revisions:**
- Add commit object specification
- Define merge protocol
- Large file handling strategies
- Distributed operation modes
- Advanced caching strategies

---

## Acknowledgments

This specification was developed through collaborative design discussions exploring line-level content addressing as an alternative to traditional file-based version control.

Key insights:
- Line-level granularity enables maximum deduplication
- Content addressing eliminates server-side delta compression
- Immutable objects enable aggressive caching
- Hash comparison replaces diff algorithms
- VFS layer bridges content store and user expectations

---

**End of Specification**
