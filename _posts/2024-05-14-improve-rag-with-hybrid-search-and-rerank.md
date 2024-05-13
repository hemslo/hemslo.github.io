---
layout: page
title: Improve RAG with Hybrid Search and Rerank
permalink: /improve-rag-with-hybrid-search-and-rerank/
description: |
  Explore how to enhance RAG for question answering by integrating hybrid search with reranking,
  addressing the limitations of vector search through diversity and precision.
---

## Introduction

RAG (Retrieval-Augmented Generation) is a popular model for question answering.
However, a simple flow of vector search and generation may not be enough.
In this post, we will explore how to improve RAG with hybrid search and rerank.

## Vector Search Problems

Vector search can return semantically similar results.
It's good at semantic search, such as finding related documents based on the meaning,
even the input format or language is different from the documents.

But there are 3 problems with vector search:

1. The search results may not be diverse. Too many similar results are not helpful.
2. No filtering based on the input question.
3. It's not good at matching exact phrases like abbreviations.

Let's see how to solve these problems.

## Maximal Marginal Relevance

To ensure diversity, we can use a rerank method called Maximal Marginal Relevance
([MMR](https://www.cs.cmu.edu/~jgc/publication/The_Use_MMR_Diversity_Based_LTMIR_1998.pdf)).

The idea is to select the most relevant document first, then skip the similar ones.
So the end results are diverse, and the user can see different perspectives.

Example usage for Redis retriever in [LangChain](https://python.langchain.com/v0.1/docs/integrations/vectorstores/redis/#redis-as-retriever):

```python
retriever = rds.as_retriever(
    search_type="mmr", search_kwargs={"fetch_k": 20, "k": 4, "lambda_mult": 0.1}
)
```

## Self Query

To filter the search results based on the input question, we can use self query.

Pass user query to LLM, let it generate a structured result, including `query`, `filter` and `limit`(optional).

Here is an example from [LangChain doc](https://python.langchain.com/v0.1/docs/modules/data_connection/retrievers/self_query/)

```text
User Query: What are songs by Taylor Swift or Katy Perry about teenage romance under 3 minutes long in the dance pop genre

Structured Request:
{
    "query": "teenager love",
    "filter": "and(or(eq(\"artist\", \"Taylor Swift\"), eq(\"artist\", \"Katy Perry\")), lt(\"length\", 180), eq(\"genre\", \"pop\"))"
}
```

Then the request will be parsed and converted to different vector store queries, [Redis example](https://python.langchain.com/v0.1/docs/integrations/retrievers/self_query/redis_self_query/#creating-our-self-querying-retriever).

Narrowing down search results from metadata can help to improve the quality of the final answer.

## Keyword Search

Keyword search is the most common way to match exact phrases.
We have used it for decades. It's still effective for many cases.

Keyword search is done by inverted index, key is the keyword, value is document ids.
It's a sparse index, compare to dense index for vector search.
On search time, there are many algorithms to rank the results, such as TF-IDF, BM25, etc.

Full-text search engine like ElasticSearch is usually used for keyword search.

## Hybrid Search

To get the best of both worlds, we can combine vector search and keyword search.
For the same input question, we can run both searches, then merge the results.

It's also possible to add more search sources, such as database search, web search,
or even multiple vector stores.

## Rerank

Now we have multiple search results, we can rerank them based on the input question.

One simple way is to use Reciprocal Rank Fusion ([RRF](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf)).
The idea is if a document is ranked higher in multiple search results, it should be more relevant.
It only relies on the rank in each source result, not the score, so scoring methods can be different in each source.

In LangChain, there is [Ensemble Retriever](https://python.langchain.com/v0.1/docs/modules/data_connection/retrievers/ensemble/)
to combine multiple retrievers. See the full hybrid retriever chain example [here](https://github.com/hemslo/chat-search/blob/377e41f961625564bbed1e2e93c619107e0f8bb7/app/chains/retriever.py).

A more advanced way is to use a rerank model to sort the results.
Such as [Cohere reranker](https://python.langchain.com/v0.1/docs/integrations/retrievers/cohere-reranker/),
it will rerank the results based on the input question and the search results.

## Conclusion

In this post, we have explored how to improve RAG with hybrid search and rerank.
By combining vector search, keyword search, and rerank, we can get better results for question answering.
