## Stem-word exclusion
- Imagine, for instance, that users searching for the “World Health Organization” are instead getting results for “organ health.” The reason for this confusion is that both “organ” and “organization” are stemmed to the same root word: organ. Often this isn’t a problem, but in this particular collection of documents, this leads to confusing results. We would like to prevent the words organization and organizations from being stemmed

<br>

## Custom stopwords
- The default list of stopwords used in English are as follows:
```
a, an, and, are, as, at, be, but, by, for, if, in, into, is, it,
no, not, of, on, or, such, that, the, their, then, there, these,
they, this, to, was, will, with
```
- The unusual thing about no and not is that they invert the meaning of the words that follow them. Perhaps we decide that these two words are important and that we shouldn’t treat them as stopwords.

<br>

## Config
```
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_english": {
          "type": "english",
          "stem_exclusion": [ "organization", "organizations" ], 
          "stopwords": [ 
            "a", "an", "and", "are", "as", "at", "be", "but", "by", "for",
            "if", "in", "into", "is", "it", "of", "on", "or", "such", "that",
            "the", "their", "then", "there", "these", "they", "this", "to",
            "was", "will", "with"
          ]
        }
      }
    }
  }
}

GET /my_index/_analyze?analyzer=my_english 
The World Health Organization does not sell organs.
```

<br>

## [One language per Document](https://www.elastic.co/guide/en/elasticsearch/guide/current/one-lang-docs.html)
## [One language per Field](https://www.elastic.co/guide/en/elasticsearch/guide/current/one-lang-fields.html)
## [Mix language Field](https://www.elastic.co/guide/en/elasticsearch/guide/current/mixed-lang-fields.html)