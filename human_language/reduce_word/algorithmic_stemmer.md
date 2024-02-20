## Algorithmic Stemmers

- Most of the stemmers available in Elasticsearch are algorithmic in that they apply a series of rules to a word in order to reduce it to its root form, such as stripping the final s or es from plurals. They don’t have to know anything about individual words in order to stem them.

- These algorithmic stemmers have the advantage that they are available out of the box, are fast, use little memory, and work well for regular words. The downside is that they don’t cope well with irregular words like be, are, and am, or mice and mouse.

- One of the earliest stemming algorithms is the Porter stemmer for English, which is still the recommended English stemmer today. Martin Porter subsequently went on to create the Snowball language for creating stemming algorithms, and a number of the stemmers available in Elasticsearch are written in Snowball.

## Using an Algorithmic Stemmer
- While you can use the porter_stem or kstem token filter directly, or create a language-specific Snowball stemmer with the snowball token filter, all of the algorithmic stemmers are exposed via a single unified interface: the stemmer token filter, which accepts the language parameter.

- For instance, perhaps you find the default stemmer used by the english analyzer to be too aggressive and you want to make it less aggressive. The first step is to look up the configuration for the english analyzer in the language analyzers documentation, which shows the following:
```
{
  "settings": {
    "analysis": {
      "filter": {
        "english_stop": {
          "type":       "stop",
          "stopwords":  "_english_"
        },
        "english_keywords": {
          "type":       "keyword_marker", 
          "keywords":   []
        },
        "english_stemmer": {
          "type":       "stemmer",
          "language":   "english" 
        },
        "english_possessive_stemmer": {
          "type":       "stemmer",
          "language":   "possessive_english" 
        }
      },
      "analyzer": {
        "english": {
          "tokenizer":  "standard",
          "filter": [
            "english_possessive_stemmer",
            "lowercase",
            "english_stop",
            "english_keywords",
            "english_stemmer"
          ]
        }
      }
    }
  }
}
```