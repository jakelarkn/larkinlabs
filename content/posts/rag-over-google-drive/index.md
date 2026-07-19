---
title: "RAGtime"
date: 2026-07-15
draft: false
tags: ["rag", "ai", "postgres", "pgvector", "mcp", "ollama"]
---

My wife had a task to improve a few dozen blog posts with cross-links to relevant podcasts. She specs the task into her LLM chat agent. The agent interprets the requirements, and for each blog article searches the business's 400 episode podcast catalog for the single best match, drafts a paragraph on why that episode is the right next listen, generates the HTML for an embedded podcast player, and ships the section into the blog posts directly on the live WordPress site. Forty blog posts are updated in a few hours. She of course created the base content, idea and task. I created a RAG AI agent system that executes it.

Backing up a bit on the background: my wife writes a lot of content. She has run a small niche online education company for about a decade. She's written two books, dozens of course manuals, hundreds of blog posts, emails, sales funnels and marketing copy. We're obviously tech aware, so she's been using AI to augment her writing for the last two years. As a part of my husband duties, I occasionally build out small tech tools to help her out - historically things like API integrations or payment system workflows. Last fall, I wrote some agent configurations to allow her Claude session to update the content of the WordPress website directly. Most recently, I spent a couple days building her a private Retrieval Augmented Generation system over the business's Google Drive. The workspace is 6,000 documents (as I said, she writes a lot!), which have accumulated over a decade: courses, podcasts, marketing emails, alumni testimonials, partnership pitches, manuals, SOPs, market research, and blog posts, plus many stock photos and videos. The old content is always valuable for synthesis of new material. The problem is finding it and using it is too time consuming. When she sat down to write new content, she might half-remember saying exactly the right thing in a years-old document. Finding the correct reference meant scrolling through a byzantine folder tree, re-reading documents and copy & pasting the right sections. So she would spend a lot of time rewriting the same concepts, even though some pretty good versions already existed in the drive archive.

The system I built runs entirely on her laptop. Postgres, ollama, a small Python indexer that pulls from the Drive, and an MCP server that exposes the index as two tools to Claude Code. The document base mostly stays on the machine, no SaaS upload of the entire workspace, no paid hosted embedding API, no third-party indexer crawling the Drive.

## What RAG is, briefly

Retrieval-Augmented Generation (RAG) has a few steps. Break the documents into chunks. Convert each chunk into a vector that encodes meaning (an "embedding"). Store those in a database that can search them effectively. Chunk embeddings are similar to how LLMs encode text and concepts internally, just on a much smaller scale and outside the model itself. It's a bit like giving the LLM a tool in its own language.  We store the embedding vectors in a database that can find the most relevant ones to a new query. Conveniently quite a few database providers realized the importance of vector similarity search a few years ago and such a service is available in a few different varieties.  At prompt time, the RAG empowered LLM agent can query the database for the most similar textual chunks and augment them to the prompt so the model can answer from your stored content rather than just from the chat history or general training data.  

A RAG embedding search API can be exposed to the model by Model Context Protocol (MCP), a standardized way to provide an API and its summary documentation to an LLM agent. The protocol makes it easy to tell the agent 1) that this RAG tool exists 2) what it could be used for, and 3) how to access it. 

## The stack (in brief)

Storage and search are Postgres 17 with the pgvector extension.

The embedding model is `nomic-embed-text`, running under Ollama, locally on the Mac's Apple Silicon.

The indexer is a few hundred lines of Python that walks the Drive over the Google Drive API using an OAuth Desktop credential with read-only scope.

Retrieval is hybrid: vector similarity combined with classical keyword search, merged via Reciprocal Rank Fusion.

On top of that sits a small MCP server that exposes two tools to Claude Code: `search` and `fetch_document`. 


## Low Level Decisions Deep Dive

### DiWHY?

Most "AI for your business" tutorials veer toward uploading everything to some startup's SaaS. That was a non-starter for this task, mostly due to data privacy concerns. The Drive workspace contains sensitive personal records, financial information, and work contracts. Running the embedding model locally with Ollama added essentially no complexity over calling a hosted embedding API, and it made the system viable under a privacy constraint. A second concern was cost; this was a test project, and the local solution dropped the cost to basically nothing.

### Ingestion, Data filtering, and Embedding

The indexer is a small Python script.  Claude can more-or-less write it as a one-shot with minimal corrections. Manually setting up the OAuth to Google in their cloud console was the hardest part. After that the main thing I had to tweak was filtering.  The first indexing runs got clogged on CSVs and spreadsheets containing thousands of rows of data that is useless for semantic search, e.g. 13 thousand line email-list backups or payment transaction data backups. Adding MIME file type allowlists and some custom written filters helped greatly. I also ended up adding a filter that punted files from indexing when a regex found more than a few dozen emails in it.  

For calculating embeddings I used a small local model mounted with Ollama. Ollama is a tool for running AI models locally instead of calling an external service. (It grew out of llama.cpp, a project to run Meta's LLaMA open-source LLM model on consumer hardware). The specific model I ran inside it for embedding is `nomic-embed-text` from Nomic AI. The local model's job is to turn each document chunk into a numerical fingerprint of its meaning. I picked the Nomic Model because it's fully open-weight, it benchmarks competitively against the paid hosted options from OpenAI and Cohere, and runs comfortably on a laptop.

For test chunking, 800-token chunks with 100-token overlap seemed a fine default. Smaller chunks improve precision on narrow questions but lose context. Larger chunks do the inverse. The overlap is the cheap hedge against slicing through the one sentence that mattered. Tuning further wasn't worth the time once the filters were in place.

Pulling all the data and calculating the embeddings for 6,000 documents took about 6 hours in the background.

## VectorDB choices

I briefly surveyed other VectorDB options before settling on Postgres with pgvector. 

Postgres, of course, is a tried-and-true SQL database. Importantly, Postgres has the pgvector extension, which makes vector similarity search just another operator in SQL. Lexical scoring comes from `tsvector` or `pg_trgm`. Hybrid search is a SQL table expression that fuses both with RRF. Joining chunks against the document table, or against any other structured data on the side, is a regular JOIN. So the vector index is one extension on a database I already am familiar with, and any retrieval strategy I think of later is whatever SQL I can write. I'm comfortable running databases locally and I wanted to try pgvector specifically; otherwise, Chroma might have been the friendlier starting point.

Chroma is the obvious open-source alternative, with a purpose-built Python API (and hybrid retrieval since its November 2025 sparse-vector release). It's a fine choice, just one where I'd rather have SQL's flexibility when I can. Vectorize.io turned out to be a managed RAG pipeline rather than a database itself, which puts the data and the embedding step in someone else's cloud and disqualified it for this corpus.

### Hybrid search for Retrieval 

Pure vector search is good at "find content about {a concept expressed in multiple words and phrases}." It can fall short at "find content that mentions {one word} specifically". Embeddings smooth over rare tokens, so proper nouns and niche jargon can land in ways that are hard to find. Combining vector similarity with classical keyword search via Reciprocal Rank Fusion makes the system significantly more versatile. 

Reciprocal Rank Fusion is a trick from search-engine research that lets two different search systems vote on a combined output without having to reconcile their incompatible scoring schemes. One system might be good at conceptual similarity and another at exact word matches, and their raw scores live on different scales that don't necessarily add up. RRF sidesteps the problem by throwing the scores away and keeping only the positions each system produced. A document that came in third on one list and eleventh on the other contributes a small fixed amount for each appearance, with earlier positions counting more than later ones. Because positions are unit-free, the two systems' opinions become directly comparable, and the merged list reflects their consensus without anyone needing to pick a weighting.  With the primitives that Postgres provides it can be implemented with ~20 lines of SQL.

### Exposing to Code Agents via MCP

The retrieval layer is more useful integrated than standalone. Claude Code already runs on my wife's laptop; adding an MCP server lets her ask questions in the same window where she does everything else and gets streaming answers that cite the underlying documents. The server is two tools, both thin wrappers over SQL: `search(query, k, filters)` returns ranked chunks with document IDs and metadata; `fetch_document(id)` returns the full text. Everything else is the model deciding when to call them.

I looked into wiring the MCP into her Claude and OpenAI apps (the desktop UI electron apps), but that was a dead-end.  Those systems have some MCP-like features but they are primarily external, clunky to configure, and they're just a walled garden of approved services.  Thankfully I'd already had my wife using Claude Code for another task (editing her WordPress blog).  Local MCPs are trivially connected to Claude Code with a bit of configuration.


## In practice

There are two examples from the first weeks of her using the system. Both are similar; the information was already in the workspace somewhere, but finding it cost more than redoing the work from scratch, so she'd been redoing the work. RAG dropped the cost of finding it to roughly zero.

In one example task, my wife needed four new blog posts on themes her sales lead had identified. Before creating those, step 1 would be knowing what she and her staff had already written on those topics. Without RAG, that background research alone is a half-day of keyword searches and opening documents one at a time. With the RAG system, the AI agent is able to provide her with relevant passages across all four themes in a couple minutes, and the blog drafts can be stitched together from sentences she had already written, in her own voice, sometimes years apart: a forgotten line from a 2025 promotional document, exact pricing figures from one doc, a graduate's testimonial from another, a budget line from a third, all dropped into the new draft by the agent. Four posts that would have taken a content writer a week or two of research and writing were ready to review in under an hour. 

Back to the cross-linking task from the top of the post. Her blog and her podcast serve the same audience, but the two had almost no cross-linking, which meant every reader who arrived at the blog from Google search was leaving without ever being made aware of the podcast: a huge missed opportunity to drive search traffic into the podcast. She asked the AI agent to take her top forty blog posts by organic traffic and, for each one, find the single best-matching episode (out of four hundred) to link directly in the post. Doing this by hand is effectively impossible. Nobody remembers four hundred episode summaries well enough to pair them confidently against forty articles, and a generic AI without access to her actual catalog would have hallucinated episode titles that don't exist. The agent handled the entire pipeline on its own: it ran a similarity search across the indexed catalog to find each true best match, drafted the intro language explaining to the reader why that specific episode was the right next thing to listen to, generated the HTML markup for the embedded player, and then pushed the finished cross-links straight into the WordPress site, post by post, without my wife or her team having to touch a single page. What would have taken staff weeks of episode-hunting and copy-pasting was done in minutes, every pairing tied to a real episode in her catalog, and along the way the system flagged that only one of her six highest-traffic pages mentioned the podcast at all, surfacing roughly twelve thousand quarterly visitors who were arriving on topic-perfect pages and leaving without ever knowing the episodes existed.

The common pattern across both: each output was hard to distinguish from what a careful human assistant with months on the job and total recall of the archive would have produced. That assistant doesn't exist for any small business. RAG sits in for one.

## What I take from this

The system worked because the corpus existed first. A decade of writing about a specific subject, by one person, in one consistent voice, is a rare and underused asset for any small business that's been at it long enough. Most of these businesses have such an archive sitting in Drive or Notion, and the gap between having it and being able to search it usefully is huge. Closing that gap is what made a 6,000-document workspace flip from "stored" to "operational" overnight.

Wiring the retrieval layer into an agent like Claude Code multiplied the value. The agent can execute on what the retrieval surfaces: draft the post, generate the embed HTML, ship the cross-link to WordPress, all in one task. That is qualitatively different from a smarter search box. A small business with a searchable archive and an agent that can write to its own tooling is doing work that used to take a full-time assistant.

The cost of running this stack on a laptop with open-source pieces was rounding-error small, and the document base never left the machine. There's still a default assumption in this space that anything AI-shaped means uploading the corpus to someone else's cloud. For corpora that include student records, partner contracts, or financial data, that assumption is wrong, and it costs very little to refuse it.

## Appendix: scaling this up

The personal version of this system topped out at 6,000 documents on a laptop, with one user, in one process. The same architecture changes in specific ways once you push it toward something much larger. I've spent the last decade-plus building distributed data systems at Strava, Uber, and AWS, so this appendix sketches the handful of choices I'd think hardest about if I were doing this for a thousand users instead of one.

### Storage and retrieval

Postgres with pgvector and an HNSW index covers most of what heavier-weight vector databases would do. With careful index tuning and read replicas, a sharded Postgres cluster comfortably holds tens of millions of vectors per shard, and you keep SQL for everything that isn't the vector itself. Most of the choices that move beyond this level are focused on other optimizations: QPS, tail latency, or operations over multi-tenant, multi-query systems. When those become the binding constraint, one of the dedicated systems (Pinecone if you want managed, or self-hostable open source like Milvus, Qdrant, Weaviate, Vespa) is the natural next step. They are operationally cheaper per-vector and have richer primitives for vector-specific work. The trade-off is that the joins against your application data move from SQL into code, and you are now running two systems instead of one. 

### Ingestion

The laptop version is a Python script that walks the Drive in one batch. The distributed version is an event-driven pipeline: a change event bus (Kafka or Kinesis) carrying document-changed events from each source (Drive, SharePoint, Confluence, Slack, S3, Notion), and a pool of horizontally scalable workers that pick up those events, calculate the embeddings, and write them to the index. Workers are idempotent so the bus can replay safely, and the pool scales with ingest load rather than with corpus size, so a burst of new documents doesn't back up the rest of the system.

### The embedding pipeline

The laptop version treats embedding as a one-shot cost. At scale it becomes the most expensive part of the system by a wide margin, and most of the engineering attention here is well spent.

The first decision is hosted versus self-hosted. Hosted APIs (OpenAI, Voyage AI, Cohere) are operationally trivial and get expensive fast at the per-token rate. Self-hosted GPU inference (vLLM, Triton, TGI) puts you in the business of running a model server and cuts the marginal cost dramatically. The break-even is somewhere around tens of millions of tokens per day, depending on contract rates.

The second decision is caching. Most documents change rarely; if the embedding cache is keyed by content hash, the steady-state load drops to whatever the daily churn is, which is usually a small fraction of the total corpus. Caching the embedding step is the highest-leverage piece of engineering in this part of the stack, and I would build that before I scaled the inference layer.

### Knowing whether changes help

The hardest part of scaling a retrieval system is knowing whether each change to chunking, embedding, retrieval weighting, or reranking is actually making things better. An eval harness is the honest answer: a few hundred real questions with the documents that should win for each, scored automatically on every change. I haven't built one for a RAG system specifically. The closest analogue from regular software engineering is the integration test suite, and I've spent enough years on systems that depended on dense coverage to safely refactor to recognize the same shape of problem here: tedious to build, easy to let rot, load-bearing once you have it.

I would treat the eval harness as a first-class production system, with its own dashboards and alerts, and I would block any retrieval change from shipping without a green run against it. This feels heavy on day one. It is about the right investment by month six.
