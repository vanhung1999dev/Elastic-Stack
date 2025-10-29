
<summary>Text Analysis</summary>
<details>
##📚 What Is Text Analysis?

Text analysis is the process of breaking down a piece of text into smaller elements (tokens) and normalizing them for efficient search. <br>
In Elasticsearch, this is done using a combination of: <br>

- Character Filters → Modify the original text before tokenization.
- Tokenizers → Split text into tokens (usually words).
- Token Filters → Modify or remove tokens after tokenization.

#### Pipeline
```
Text → Character Filters → Tokenizer → Token Filters → Indexed Tokens
```

### ⚙️ Built-in Analyzers
| Analyzer                  | Description                                                             |
| ------------------------- | ----------------------------------------------------------------------- |
| `standard`                | The default analyzer. Splits text on word boundaries and lowercases it. |
| `simple`                  | Splits text on non-letter characters and lowercases terms.              |
| `whitespace`              | Splits text only by whitespace.                                         |
| `stop`                    | Same as simple analyzer but removes stop words.                         |
| `keyword`                 | Treats the entire string as a single token (useful for exact matches).  |
| `english`, `french`, etc. | Language-specific analyzers with stemming and stop-word removal.        |

### 🧪 The Analyze API
#### 🔍 Basic Example
```
GET /_analyze
{
  "analyzer": "standard",
  "text": "The Quick Brown Foxes."
}
```

#### Response:
```
{
  "tokens": [
    { "token": "the", "start_offset": 0, "end_offset": 3, "type": "<ALPHANUM>", "position": 0 },
    { "token": "quick", "start_offset": 4, "end_offset": 9, "type": "<ALPHANUM>", "position": 1 },
    { "token": "brown", "start_offset": 10, "end_offset": 15, "type": "<ALPHANUM>", "position": 2 },
    { "token": "foxes", "start_offset": 16, "end_offset": 21, "type": "<ALPHANUM>", "position": 3 }
  ]
}
```
</details>

<summary>Inverted Index</summary>
<details>
  # Inverted Index Internals — Deep Dive into How It Works in Lucene and Elasticsearch

## 1. Overview

An **inverted index** is the core data structure behind Elasticsearch and Apache Lucene.  
It enables **fast full-text search** by mapping **terms (words)** to the **documents** that contain them — essentially the inverse of a traditional forward index (which maps documents → terms).

---

## 2. High-Level Concept

- **Forward index:**  
  `docID → [list of terms]`

- **Inverted index:**  
  `term → [list of docIDs]`

Instead of scanning all documents to find a term, Elasticsearch queries the inverted index directly — allowing sub-linear search performance.

---

## 3. How Text Becomes an Inverted Index

When a document is indexed, its textual fields go through a **text analysis pipeline**.  
Each field is transformed into a stream of tokens, then recorded in the inverted index.

### Simplified flow:
```
Document
↓
Field value ("The quick brown fox")
↓
Analyzer pipeline (character filter → tokenizer → token filters)
↓
Token stream: ["quick", "brown", "fox"]
↓
Inverted index:
quick → [doc1]
brown → [doc1]
fox → [doc1]
```


---

## 4. Internal Components of the Inverted Index

### 4.1. **Term Dictionary (Lexicon)**
- Contains **all unique terms** in a field.
- Each term points to:
  - Its **posting list**
  - Its **term statistics** (e.g., total frequency, document frequency)
- Stored efficiently as a **Finite State Transducer (FST)** — a compressed automaton that maps term → posting list pointer.

> Example:
> ```
> Term Dictionary:
> ├─ apple → pointer to postings block 0x01
> ├─ banana → pointer to postings block 0x0A
> └─ cherry → pointer to postings block 0x10
> ```

---

### 4.2. **Postings List**
- The **heart** of the inverted index.
- For each term, it stores **which documents** contain that term and metadata for scoring and phrase matching.

Each entry is called a **posting** and contains:
| Field | Description |
|--------|--------------|
| `docID` | Numeric ID assigned to the document |
| `termFreq` | Number of times the term appears in that document |
| `positions` | Word positions within the field (for phrase queries) |
| `offsets` | Character start/end offsets (for highlighting) |
| `payloads` | Optional custom bytes per term occurrence |

> Example (term = "quick"):
> ```
> Postings:
>   docID=1, freq=2, positions=[3, 15]
>   docID=3, freq=1, positions=[7]
> ```

---

### 4.3. **Doc Values**
- Stored separately from the inverted index.
- Enable **sorting**, **aggregations**, and **faceting** without scanning the inverted lists.
- Stored in **columnar format** for fast sequential reads.

---

### 4.4. **Stored Fields**
- These hold the **original JSON source** or specific fields to return in search hits.
- Not part of the inverted index but stored in the same Lucene segment.

---

## 5. Compression and Storage Efficiency

Lucene optimizes for disk and memory through several compression mechanisms:

### • Delta Encoding (for docIDs)
Stores differences between consecutive docIDs rather than full integers:
```
[100, 105, 107] → [100, +5, +2]
```


### • Variable-Length Integer Encoding (VInt)
Smaller numbers (like docID deltas) use fewer bytes.

### • Block-based Compression
Postings lists are grouped into blocks of 128/256 entries, then compressed using:
- **Frame-of-Reference (FOR)**
- **SIMD-BP128**
- **LZ4** for stored fields

### • FST for Term Dictionary
Highly compressed finite automaton reduces memory footprint while enabling fast prefix/suffix lookups (used in wildcard or fuzzy queries).

---

## 6. Query Execution Using the Inverted Index

When you run a query (e.g., `quick brown`):

1. **Parse query** into tokens → `["quick", "brown"]`
2. **Lookup postings list** for each term
3. **Intersect or union postings** depending on query type:
   - `AND`: intersection of postings lists
   - `OR`: union of postings lists
4. **Score documents** using TF-IDF or BM25 based on:
   - Term frequency (`tf`)
   - Inverse document frequency (`idf`)
   - Field normalization (`norm`)
5. **Return top-K results**

Example:
```
"quick" → [doc1, doc3, doc5]
"brown" → [doc1, doc2, doc5]

AND → intersection = [doc1, doc5]
```


---

## 7. Segments and Merge Process

Lucene writes data in immutable **segments**:
- Each segment is a **mini inverted index** (term dictionary + postings + stored fields).
- New documents are written to new segments.
- Over time, background **merge operations** combine smaller segments into larger ones for performance and space optimization.

### Merge example:
```
Segment_1: docs [1–1000]
Segment_2: docs [1001–2000]
→ merge → Segment_3: docs [1–2000]
```



When merged:
- Duplicate/deleted docs are dropped
- Postings lists are re-written
- FSTs are rebuilt

---

## 8. Handling Updates and Deletes

Since segments are immutable:
- **Updates** = mark old doc as deleted + index a new version in a new segment.
- **Deletes** = maintain a **bitset** (`.del` file) marking which docIDs are invalid.

During searches:
- Deleted docs are skipped using this bitset.
- Deleted docs are physically removed during segment merge.

---

## 9. Example Visualization

Let’s see a simple example with 2 documents:

| docID | Text |
|--------|------|
| 1 | "quick brown fox" |
| 2 | "quick red fox" |

After analysis:

| Term | Postings (docID → positions) |
|------|-------------------------------|
| brown | 1 → [2] |
| fox | 1 → [3], 2 → [3] |
| quick | 1 → [1], 2 → [1] |
| red | 2 → [2] |

---

## 10. Inverted Index vs Forward Index

| Feature | Forward Index | Inverted Index |
|----------|----------------|----------------|
| Structure | docID → terms | term → docIDs |
| Use Case | Document reconstruction | Full-text search |
| Storage | Sequential | Term-based |
| Query Speed | O(n) | O(log n) or better |
| Update Speed | Fast | Moderate (due to segment merges) |

---

## 11. Summary

| Concept | Description |
|----------|-------------|
| **Analyzer** | Converts text into tokens |
| **Term Dictionary (FST)** | Maps terms to posting lists |
| **Postings List** | Stores docIDs, positions, frequencies |
| **Doc Values** | Columnar structure for aggregations/sorting |
| **Segments** | Immutable mini-indices merged periodically |
| **Deletes** | Managed via bitsets until merge |
| **Compression** | Uses delta + variable-length + block encoding |

---

## 12. Key Takeaways

- The inverted index is **optimized for read performance**.
- Lucene achieves **high compression** and **fast lookup** through FSTs and delta encoding.
- **Segment-based immutability** simplifies concurrency, rollback, and merges.
- Query performance depends heavily on **analyzers**, **term statistics**, and **index structure**.

---

## 13. References

- [Lucene Inverted Index Formats](https://lucene.apache.org/core/)
- [Elasticsearch: How Inverted Index Works](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html)
- [Understanding FSTs in Lucene](https://blog.mikemccandless.com/2010/12/using-finite-state-transducers-in.html)

---

</details>
