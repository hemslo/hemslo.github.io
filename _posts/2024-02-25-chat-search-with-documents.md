---
layout: page
title: Chat Search with Documents
permalink: /chat-search-with-documents/
---

## Introduction

Generative AI and RAG (Retrieval-Augmented Generation) have been a hot topic.
It's very easy now to add a chatbot to your website or application.

I created a simple tool [chat-search](https://github.com/hemslo/chat-search),
and setup a demo for my [blog](https://hemslo.io/chat/).
You can ask questions like "How to chat with documents?"
and it will answer based on this post you are reading.

Now let's walk through the process of building it.

## Flow and Architecture

There is a good explanation in LangChain doc [Q&A with RAG
](https://python.langchain.com/docs/use_cases/question_answering/).

Similar to building full-text search feature, there are 2 stages, index and query.

### Index

`documents -> embeddings -> vector store`

Index is the process to convert documents into a format that can be searched efficiently.
A key difference from full-text search is that we are using an embedding model to generate embeddings
instead of traditional tokenization and inverted index.

### Query

`query -> embeddings -> vector store -> documents -> LLM -> response`

Query is the process to find the most relevant documents for a given query.
It will use the same embedding model to generate query embeddings,
then use knn to find the most relevant documents.
Additionally, instead of just returning documents, 
we can combine retrieved documents and query to generate a final response via a LLM (Large Language Model).

## Tech Stack

* LangChain
* LangServe
* Embedding Model
* Chat Model 
* Redis as vector store

### LangChain

[LangChain](python.langchain.com) is a very popular framework for building LLM applications.

It provides a lot of useful tools, integrations and examples,
can bootstrap an AI project using building blocks quickly.

### LangServe

[LangServe](https://python.langchain.com/docs/langserve) is another great tool from LangChain.
It integrates with [FastAPI](https://fastapi.tiangolo.com/) to build REST API for LLM app.
If you are familiar with web development, this is a good entry point.

### Embedding Model

Embedding model is the core model of the retrieval system.
It's a model that can convert text into embeddings.

The one I used for blog demo is OpenAI [text-embedding-3-small](https://platform.openai.com/docs/guides/embeddings/embedding-models).

Can also use other models like [nomic-embed-text](https://ollama.com/library/nomic-embed-text).

### Chat Model

Chat model is used to rephrase the query and generate a response.

The one I used for blog demo is OpenAI [gpt-3-5-turbo](https://platform.openai.com/docs/models/gpt-3-5-turbo).

Can also use other models like [gemma](https://ollama.com/library/gemma).

### Redis as vector store

There are many [vector store integrations](https://python.langchain.com/docs/integrations/vectorstores) in LangChain,
I chose [Redis](https://python.langchain.com/docs/integrations/vectorstores/redis) because it's easy to use and deploy,
probably already used in many web applications.

## Ingest Documents

This step is a traditional ETL process.
Extract content from different sources,
transform it into a standard document format,
then load it into the system for indexing.

### Crawler

I built a simple crawler, based on [SitemapLoader](https://python.langchain.com/docs/integrations/document_loaders/sitemap)
and [RecursiveUrlLoader](https://python.langchain.com/docs/integrations/document_loaders/recursive_url).
Can extend to more powerful ones like Chromium based, [Web scraping](https://python.langchain.com/docs/use_cases/web_scraping).

### Standard CRUD

LangChain has a new [indexing API](https://python.langchain.com/docs/modules/data_connection/indexing),
but I didn't use it because I want to keep the system simple (no need to have another sql database).

Just use Redis to store the documents. With the source URL hash as document key,
the content hash as value, can be used to avoid recomputation.

The document need to split into multiple ones, as embedding model has a limit on input length.
This can be slow for large documents, can be optimized by doing sync save content and async process embeddings.

### Continuous Ingest

The crawler is bundled together in the full package, published as a docker image, [chat-search](https://github.com/hemslo/chat-search/pkgs/container/chat-search).

To keep the index data up to date with blog data, I setup a github action [crawl.yml](https://github.com/hemslo/chat-search/blob/main/.github/workflows/crawl.yml),
and trigger it after github pages deploy [jekyll.yml](https://github.com/hemslo/hemslo.github.io/blob/master/.github/workflows/jekyll.yml).

## Deployment

### Redis

I use Redis Cloud, the [free tier](https://redis.com/cloud/pricing/) is good enough for the blog demo.

### Google Cloud Run

Deploy the container image to Google Cloud Run is very easy,
and the [free tier](https://cloud.google.com/run/pricing) is also good enough for the blog demo.

There is a full CI/CD example using github actions, [cicd.yml](https://github.com/hemslo/chat-search/blob/main/.github/workflows/cicd.yml).
It will build the container image, push to GitHub packages and Google Cloud Artifact Registry,
then deploy to Google Cloud Run.

## Conclusion

Just a few hundred lines of code, a chatbot with document search is up and running.
Generative AI app development is getting easier and easier,
maybe every product will ship with a chatbot/copilot/assistant in the future,
just like how every product has a web or mobile app now.
