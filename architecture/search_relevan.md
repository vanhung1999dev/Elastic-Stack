## True Positive
- **Relevant Document that are return to user**
<br>

## False Positive
- **irrelevant Document that are return to user**
<br>
<br>

- ![false_positive](/imgage/Screen%20Shot%202024-02-19%20at%2021.20.59.png)

<br>

## True negative
- **relevant Document that not return to user**
<br>

## False negative
- **irrelevant Document that not return to user**
<br>

- ![negative](/imgage/Screen%20Shot%202024-02-19%20at%2021.25.14.png)

<br>

## Precise
- Precision is calculated by true positives divided by the sum of true positives and false positives. Precision tells you what portion of the retrieved data is actually relevant to the search query.
<br>

![precise](/imgage/Screen%20Shot%202024-02-19%20at%2021.27.48.png)
<br>

## Recall
-  is calculated by true positives divided by the sum of true positives and false negatives. It tells you what portion of relevant data is being returned as search results.
<br>

![recall](/imgage/Screen%20Shot%202024-02-19%20at%2021.29.58.png)
<br>

## What is a score?
- Score is a value that represents how relevant a document is to that specific query. A score is computed for each document that is a hit, and hits are search results that are sent to the user.
- The higher the score a document has, more relevant the document is to the query, and it’s going to end up higher in the order!
<br>

### How is a score calculated?
- There are multiple factors that are used to compute a document's score. This blog will only focus on term frequency(TF) and inverse document frequency(IDF).
- ![score](/imgage/Screen%20Shot%202024-02-19%20at%2021.32.08.png)
<br>

### What is term frequency?
- When the relevant documents are retrieved, Elasticsearch looks through these documents and calculates how many times each search term appears in a document.
- ![TF](/imgage/Screen%20Shot%202024-02-19%20at%2021.34.24.png)
<br>

### What is Inverse Document Frequency(IDF)?
- When we calculate a score based on term frequency alone, this will not give us the most relevant documents. This happens because term frequency considers all search terms to be equally important when assessing the relevance of a document.
- ![IDF](/imgage/Screen%20Shot%202024-02-19%20at%2021.36.52.png)
<br>

- We have search terms, "how" and "to" and "form" and "good" and "habits". Not all of these search terms will help you determine the relevance of a document.

- For example, the first four search terms are commonly seen in many, if not all documents. Take a look at the hits(blue box), then at the documents highlighted with an orange box.

- The documents like "how to form a meetup group" or "good chicken recipes" do contain some of the search terms. But these documents are completely irrelevant to what we are looking for!

- But because of term frequency, if these commonly found search terms were found in high frequency in any of these documents, these documents will end up with high scores, even though these are irrelevant to the query.

- So Elasticsearch offsets this with inverse document frequency(IDF). With Elasticsearch, if certain search terms are found in many documents in the result set, it knows that these terms are not useful at determining relevance.

- When Elasticsearch goes through all the hits, it will reduce the score for documents with unimportant search terms. It will also increase the score for documents with important search term like habits.

<br>

[Source Reference](https://dev.to/lisahjung/beginner-s-guide-to-understanding-the-relevance-of-your-search-with-elasticsearch-and-kibana-29n6)