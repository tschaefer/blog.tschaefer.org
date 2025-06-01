---
title: "A Solid Semantic Search for RSS Feeds"
description: "In this blog post, we explore how to index text information and
perform semantic search on it using Sentence Transformers (SBERT) and SQLite
with a Vector Extension."
images:
    - "https://blog.tschaefer.org/images/semantic-rss-search.png"
author: "Tobias Schäfer"
date: 2025-05-31T07:00:00+02:00
draft: false
toc: false
images:
tags:
  - nlp
  - python
  - rss
  - sbert
  - semantic search
  - sentence transformers
  - sqlite
---

> Retrieval-augmented generation (RAG) is a technique that enables large
language models (LLMs) to retrieve and incorporate new information. Typically,
the data to be referenced is converted into LLM embeddings, numerical
representations in the form of a large vector space.
<sub> -- From the Wikipedia article [Retrieval-augmented generation (RAG)](https://en.wikipedia.org/wiki/Retrieval-augmented_generation)</sub>

The latter part is the focus of this blog post, where we explore how to to
index text information and perform semantic search on it. We use
[Sentence Transformers (SBERT)](https://www.sbert.net) to encode the
information and store it in [SQLite database with a Vector Extension](https://alexgarcia.xyz/sqlite-vec/).
[FastAPI](https://fastapi.tiangolo.com/) is used to create a simple REST API,
allowing to embed and search RSS feeds.

![Semantic Search for RSS Feeds](/images/semantic-rss-search.png)

The crucial part is the transformation of the information into embeddings,
which is done by tokenizing the text and then encoding by interfering with a
pre-trained language model. We use the [all-MiniLM-L6-v2](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2)
model, a sentence or paragraph transformer model which gives us a
384-dimensional vector representation of maximum 256 tokens of text input.

The blog post related project is available on [GitHub](https://github.com/tschaefer/semantic-rss-search/).
The project provides information on how to set up the environment and run the
application, either locally or in a Docker container.

## Parse and Tokenize RSS Feed Entries

The maximum number of tokens has a direct impact on the information we use
from the RSS feed entries to create the embeddings.

We are interested in the `title`, `summary`, `published date` and `link` of
each feed entry. The combination of title and summary is the input for the
vector embeddings and in preparation splitted into sentences. Before this step
the content is cleaned up to avoid wasting tokens on HTML tags and other
non-textual content.

```python
def __build_entries(self, feed):
        entries = []
        for entry in feed.entries:
            title = entry.get('title', 'No title').replace('\n', ' ').strip()
            summary = entry.get('summary', 'No summary').replace('\n', ' ').strip()

            published = time.mktime(entry.get('published_parsed', time.gmtime()))
            link = entry.get('link', None)

            sentences = nltk.tokenize.sent_tokenize(self.__strip_html(title) + ' ' +
                                                    self.__strip_html(summary))
            entries.append(self.__create_feed_entry(title, summary, published, link, sentences))

        self.entries = entries
```

## Vectorize and Store the Information

The SQLite database and the required table `embeddings` are created on service
start if needed. For better performance we enable the Write-Ahead Logging (WAL)
mode, which allows concurrent reads and writes to the database.

```python
def __connect_db(self):
        db = sqlite3.connect(self.dbname, check_same_thread=False)
        db.execute('PRAGMA journal_mode=WAL;')
        db.enable_load_extension(True)
        sqlite_vec.load(db)
        db.enable_load_extension(False)
        return db
```

The vector extension is loaded and provides the posiibilty of storing
embeddings as a 384-dimensional float vector. The further fields and the
number of input tokens are also persisted as metadata in the virtual
`embeddings` table.

```python
 def __init_db(self):
        with self.__connect_db() as db:
            db.execute(f'''
                CREATE VIRTUAL TABLE IF NOT EXISTS embeddings
                USING vec0(
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    title TEXT,
                    summary TEXT,
                    published FLOAT,
                    link TEXT,
                    token INTEGER,
                    vector float[{self.dimension}]
                );
            ''')
            db.commit()
```

The database connection is re-established on each request - embedding or
searching - to grant thread-saftey and avoid issues with concurrent access and
eventual database locks or data corruption.

The common part for embedding or searching is the vectorization of the input.
This is done by encoding a list of sentences using the SBERT model,
and returning the mean of the resulting vectors as a single vector. By doing
this, we ensure that even an input text exceeding the maximum token limit
is handled correctly and storable in the embeddings table with a fixed
dimension size of 384. For sure this has an impact on the quality and accuracy
of the search, but it is a trade-off we accept for the sake of simplicity.
Also we base the embeddings on the title and summary of the feed entries only.

```python
def __vectorization(self, sentences):
        vectors = self.transformer.encode(sentences)
        return np.mean(vectors, axis=0).astype(np.float32)
```

## Semantic Search as K Nearest Neighbours

For the semantic search we use the common K Nearest Neighbours (KNN)
algorithm to match the query with the stored information. KNN is simple
and effective, and finds the `k` closest vectors in the embeddings table
to the query vector. The distance is calculated using the cosine similarity,
which is a common metric for measuring the similarity between two vectors.

```python
def search(self, query, k=5):
        vector = self.__vectorization(nltk.tokenize.sent_tokenize(query))

        with self.__connect_db() as db:
            cursor = db.execute('''
                SELECT title, summary, published, link, distance, token
                FROM embeddings
                WHERE vector MATCH ? AND k = ?
                ORDER BY distance ASC;
            ''', (vector, k))

            results = cursor.fetchall()
```

## Wrap and Expose the Functionality

The REST API is built with FastAPI and wraps the embedding and search
functionality by exposing two endpoints, `/search` and `/embed`. The `/embed`
endpoint is secured with a bearer token authentication. Both endpoints accept
a POST request with a JSON payload including the required parameters for the
operation. Embedding requires a URL of the RSS feed, while search requires a
query string and optional the number of results to return.

For further details on the API, please refer to the OpenAPI schema exposed
by FastAPI at `/docs` and `/redoc` endpoints.


```python
@app.post('/search', response_model=schemas.SearchResponse)
async def search(search: schemas.SearchRequest):
    try:
        query = search.query.strip()
        k = search.k

        results = embeddings.search(query, k=k)

        return {'results': results, 'query': query, 'k': k}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.post('/embed', response_model=schemas.EmbedResponse,
          responses=schemas.authorization_responses(), status_code=201)
async def embed(embed: schemas.EmbedRequest, _: None = Depends(security.authorized)):
    try:
        url = str(embed.url)

        feed = Feed(url)
        feed.parse()
        entries = embeddings.upsert(feed)

        return {'entries': entries}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Conclusion

In this blog post, we have explored how to create a semantic search for RSS
feeds using SBERT and SQLite with a Vector Extension. We have seen how to
parse and tokenize the RSS feed entries, vectorize the information, and
perform a semantic search using KNN. The implementation is simple and
effective, and can be easily extended to support more complex use cases.

## References

- [SBERT](https://www.sbert.net)
- [SQLite with Vector Extension](https://alexgarcia.xyz/sqlite-vec/)
- [FastAPI](https://fastapi.tiangolo.com/)
- [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/)
- [NLTK](https://www.nltk.org/)
- [Hugging Face](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2)
- [Feedparser](https://feedparser.readthedocs.io/en/latest/)
- [semantic-rss-search](https://github.com/tschaefer/semantic-rss-search/)
