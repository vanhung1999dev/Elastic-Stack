## Standard Tokenize
- A tokenizer accepts a string as input, processes the string to break it into individual words, or tokens (perhaps discarding some characters like punctuation), and emits a token stream as output.

- What is interesting is the algorithm that is used to identify words. The whitespace tokenizer simply breaks on whitespace—​spaces, tabs, line feeds, and so forth—​and assumes that contiguous nonwhitespace characters form a single token. For instance:
```
GET /_analyze?tokenizer=whitespace
You're the 1st runner home!
```
- This request would return the following terms: You're, the, 1st, runner, home!

- The letter tokenizer, on the other hand, breaks on any character that is not a letter, and so would return the following terms: You, re, the, st, runner, home.

- The standard tokenizer uses the Unicode Text Segmentation algorithm (as defined in Unicode Standard Annex #29) to find the boundaries between words, and emits everything in-between. Its knowledge of Unicode allows it to successfully tokenize text containing a mixture of languages.

- **The standard tokenizer is a reasonable starting point for tokenizing most languages, especially Western languages. In fact, it forms the basis of most of the language-specific analyzers like the english, french, and spanish analyzers. Its support for Asian languages, however, is limited, and you should consider using the icu_tokenizer instead, which is available in the ICU plug-in.**