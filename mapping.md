<summary>Mapping and Type</summary>
<details>
# Introduction to the Concept of Mapping in Elasticsearch

## 1. What Is Mapping?

In Elasticsearch, **mapping** defines how a document and its fields are stored and indexed.  
It acts as the **schema** of an index — specifying:

- Which fields exist
- What data types they have (e.g., `text`, `keyword`, `date`, `integer`)
- How each field is analyzed (tokenization, normalization)
- Whether a field is searchable, sortable, or aggregatable

### Analogy:
If an **index** is like a table in a relational database,  
then **mapping** is equivalent to the **table schema** (column definitions).

---

## 2. How Mapping Works

When you create an index, Elasticsearch needs to know **how to interpret** each field.  
You can provide a **custom mapping**, or let Elasticsearch infer one dynamically.

### Example: Explicit Mapping

```json
PUT /users
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "email": { "type": "keyword" },
      "age": { "type": "integer" },
      "address": {
        "properties": {
          "city": { "type": "keyword" },
          "zip": { "type": "integer" }
        }
      }
    }
  }
}
```

### Example: Dynamic Mapping (automatic detection)
If you index a document without an explicit mapping: <br>
```
POST /users/_doc
{
  "name": "Alice",
  "email": "alice@example.com",
  "age": 25
}
```

Elasticsearch automatically infers types: <br>
```
"name" → text
"email" → text/keyword
"age" → long
```

### Two Major Field Categories
| Type                    | Description                                                  | Example                               |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------- |
| **Analyzed fields**     | Used for full-text search; text is tokenized and normalized. | `"name": "John Doe"` (`text`)         |
| **Not analyzed fields** | Stored as-is, for exact matching, sorting, and aggregations. | `"email": "john@doe.com"` (`keyword`) |


## Most Important Data Types

### Keyword Type
#### Use Case:
- Exact matching, sorting, and aggregations.
- Useful for IDs, tags, categories, or email addresses.
#### Behavior:
- Not analyzed → stored as a single token.
- Case-sensitive (unless normalizer is applied).


### Text Type
#### Use Case:
- Full-text search fields like product descriptions, blog posts, or names.
#### Behavior:
- Analyzed using a tokenizer and filters (like stemming, lowercasing).
- Searchable but not sortable by default.

### Object Type
#### Use Case:
- Represents a JSON object (a flat structure) within a document.
- Used for grouping related fields but not for array of objects.
- ```
  "address": {
  "properties": {
    "city": { "type": "keyword" },
    "zip": { "type": "integer" }
  }
  }
  ```
#### Behavior:
- Stored as a flattened structure:
- ```
  address.city → "Hanoi"
  address.zip  → 10000
  ```
- Useful when you only need single-level object fields.
#### Limitation:
- If you index an array of objects, Elasticsearch flattens it, which can cause cross-matching issues.
- ```
  "contacts": [
  { "name": "Alice", "phone": "111" },
  { "name": "Bob", "phone": "222" }
  ]
  ```
  Flattened form: <br>
  ```
  contacts.name: ["Alice", "Bob"]
  contacts.phone: ["111", "222"]
  ```
A query like name = Bob AND phone = 111 → falsely matches because of flattening. <br>

### Nested Type
#### Use Case:
- For arrays of objects where each object should remain isolated.
- Solves the cross-matching issue seen in regular objects.
- ```
  "contacts": {
  "type": "nested",
  "properties": {
    "name": { "type": "keyword" },
    "phone": { "type": "keyword" }
  }
  }
  ```
  
  ```
  GET /users/_search
  {
    "query": {
      "nested": {
        "path": "contacts",
        "query": {
          "bool": {
            "must": [
              { "term": { "contacts.name": "Bob" } },
              { "term": { "contacts.phone": "111" } }
            ]
          }
        }
      }
    }
  }
  ```
### Other Common Data Types
| Type                                 | Description                                           |
| ------------------------------------ | ----------------------------------------------------- |
| `integer`, `long`, `float`, `double` | Numeric values for scoring, sorting, or range queries |
| `boolean`                            | `true` / `false`                                      |
| `date`                               | Date and time (supports custom formats)               |
| `geo_point`                          | Latitude-longitude coordinates                        |
| `ip`                                 | IPv4 and IPv6 addresses                               |
| `flattened`                          | JSON objects with unknown or dynamic keys             |
| `range`                              | Range queries for numeric/date/IP values              |


### Chosing the right type
| Goal                 | Best Type                       |
| -------------------- | ------------------------------- |
| Full-text search     | `text`                          |
| Exact match          | `keyword`                       |
| Nested objects       | `nested`                        |
| Simple JSON object   | `object`                        |
| Sorting/Aggregations | `keyword`, `numeric`, or `date` |

</details>

<summary>How Keyword work</summary>
<details>
# Deep Dive: How the `keyword` Data Type Works in Elasticsearch

## 1. Introduction

The **`keyword`** data type in Elasticsearch is one of the most fundamental and widely used field types.  
It is designed for **structured content** — values that are meant to be **searched, filtered, sorted, or aggregated** **exactly as stored**, without text analysis or tokenization.

Common examples include:
- Usernames
- Email addresses
- Tags
- Product IDs
- Country codes
- Status fields (`"active"`, `"pending"`, `"archived"`)

---

## 2. Core Behavior

When a field is mapped as `keyword`, Elasticsearch **stores the field’s value exactly as it is** — without breaking it into tokens.

### Example:

```json
PUT /users
{
  "mappings": {
    "properties": {
      "email": { "type": "keyword" }
    }
  }
}

POST /users/_doc
{
  "email": "John.Doe@example.com"
}
```

Internally: <br>
```
Term Dictionary:
└─ "John.Doe@example.com" → [doc1]
```

✅ The value "John.Doe@example.com" is treated as a single token. <br>

No lowercasing, no stemming, no splitting — unlike the text type <br>

### Comparison: keyword vs text
| Feature          | `keyword`             | `text`             |
| ---------------- | --------------------- | ------------------ |
| **Analysis**     | ❌ Not analyzed        | ✅ Tokenized        |
| **Search Type**  | Exact match           | Full-text          |
| **Sorting**      | ✅ Supported           | ❌ Not supported    |
| **Aggregations** | ✅ Supported           | ❌ Not recommended  |
| **Storage**      | Single term           | Multiple tokens    |
| **Use Case**     | IDs, tags, categories | Paragraphs, titles |

</details>

<summary>Type corehin</summary>
<details>
# Understanding Type Coercion in Elasticsearch

## 1. Introduction

**Type coercion** in Elasticsearch refers to the process where Elasticsearch **automatically converts** a field’s value from one data type to another during indexing or querying — if it can do so safely.

This feature helps maintain **flexibility and fault tolerance** when indexing heterogeneous or semi-structured JSON data, where values might not always match the expected type perfectly.

---

## 2. Why Type Coercion Exists

In the real world, data isn’t always clean or consistent.  
For example:
- A numeric field may sometimes arrive as a string (`"42"` instead of `42`)
- A boolean field may come as `"true"` or `"1"`
- A date may be provided as an epoch number or formatted string

Without coercion, these mismatches would cause indexing errors.

Elasticsearch automatically converts compatible types to ensure documents are still indexed successfully.

---

## 3. How Type Coercion Works

Each field in a mapping has an expected **data type** (`integer`, `float`, `date`, `boolean`, `keyword`, etc.).  
When a new value is ingested:
1. Elasticsearch checks the field’s declared type.
2. If the incoming value is of a different type but **can be safely converted**, it performs the conversion automatically.
3. If conversion fails (e.g., string `"abc"` into integer), it throws a **mapper_parsing_exception**.

---

## 4. Example: Numeric Coercion

### Mapping:
```json
PUT /products
{
  "mappings": {
    "properties": {
      "price": { "type": "float" }
    }
  }
}
```

#### Indexing Document:
```
POST /products/_doc
{
  "price": "19.99"
}
```

### Type Coercion in Queries
```
GET /products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": "10",   // string
        "lte": "50"
      }
    }
  }
}
```
Even though "10" and "50" are strings, Elasticsearch coerces them to floats during query evaluation. <br>

### When Coercion Is Disabled
You can turn off automatic coercion by setting "coerce": false in your mapping.
```
PUT /products
{
  "mappings": {
    "properties": {
      "price": {
        "type": "float",
        "coerce": false
      }
    }
  }
}

POST /products/_doc
{ "price": "19.99" }
```

### Coercion Behavior by Type
| Type                       | Example of Coercion                     | Default   |
| -------------------------- | --------------------------------------- | --------- |
| `integer`, `float`, `long` | `"10"` → `10`, `"3.14"` → `3.14`        | ✅ Enabled |
| `boolean`                  | `"true"` / `"1"` → `true`               | ✅ Enabled |
| `date`                     | `"2025-10-29"` / `1730207100000` → date | ✅ Enabled |
| `keyword`                  | `"123"` → `"123"` (no coercion needed)  | ✅ N/A     |
| `object`                   | `{ "a": "1" }` → `{ "a": "1" }`         | ✅ Enabled |
| `geo_point`                | `"40.12,-71.34"` → `[40.12, -71.34]`    | ✅ Enabled |

</details>

<summary>Array Type</summary>
<details>
# Understanding Arrays in Elasticsearch

## 1. Introduction

In Elasticsearch, arrays are **first-class citizens** — you don’t need to declare them explicitly. Any field can hold **one or multiple values** of the same data type, and Elasticsearch automatically treats it as an array.

This makes it easy to store data like tags, categories, or multiple email addresses without creating a separate data structure.

---

## 2. How Arrays Are Indexed

When you index a document in Elasticsearch, and one of its fields contains multiple values, Elasticsearch **treats each value as a separate term** within the inverted index.

Example document:

```json
{
  "name": "Alice",
  "tags": ["developer", "backend", "elasticsearch"]
}
```

Internally, the tags field is stored as if it had three separate entries for the same field: <br>
| Field | Value         |
| ----- | ------------- |
| tags  | developer     |
| tags  | backend       |
| tags  | elasticsearch |


So the field tags has multiple terms in the inverted index, allowing Elasticsearch to match any of them during searches. <br>

### Arrays of Objects and Their Limitations
Arrays of primitive types (e.g., strings, numbers, booleans) are simple. <br>
However, arrays of objects behave differently and can be misleading if you’re not careful. <br>

```
{
  "comments": [
    { "author": "Alice", "likes": 10 },
    { "author": "Bob", "likes": 5 }
  ]
}
```

Elasticsearch flattens objects internally, so it stores: <br>
| Field           | Values     |
| --------------- | ---------- |
| comments.author | Alice, Bob |
| comments.likes  | 10, 5      |


⚠️ This means that the relationship between author and likes is lost! <br>
A query searching for { "author": "Alice", "likes": 5 } could incorrectly match the above document — because Alice and 5 both exist, even though not in the same object. <br>

```
comments.author → { "Alice": [doc1], "Bob": [doc1] }
comments.likes  → { 10: [doc1], 5: [doc1] }
```


There’s no linkage between the two fields at the Lucene level. <br>
Elasticsearch can only tell that the document contains both values somewhere — not that they appear together in the same object. <br>

### The Solution: The nested Data Type
To preserve relationships within arrays of objects, Elasticsearch provides the nested data type. <br>
Nested fields are indexed separately but remain logically linked to their parent document <br>

```
{
  "mappings": {
    "properties": {
      "comments": {
        "type": "nested",
        "properties": {
          "author": { "type": "keyword" },
          "likes": { "type": "integer" }
        }
      }
    }
  }
}
```

Now, a query for { "author": "Alice", "likes": 5 } won’t match incorrectly — Elasticsearch ensures both conditions apply to the same object in the array. <br>
Elastic search will understand you want to search in nested object. <br>

### Summary
| Concept              | Description                                           |
| -------------------- | ----------------------------------------------------- |
| Arrays are implicit  | Any field can have multiple values                    |
| Same type required   | Mixed types cause mapping conflicts                   |
| Flattened indexing   | Each array element becomes a separate term            |
| Arrays of objects    | Lose field relationships unless `nested` is used      |
| Sorting/Aggregations | Use `mode` to control which array value is considered |

</details>

<summary>How Date work</summary>
<details>
  
</details>
