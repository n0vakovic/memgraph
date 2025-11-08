# RAG Platform Design on Memgraph

**Date:** 2025-11-08
**Status:** Design Proposal
**Purpose:** Design a comprehensive RAG (Retrieval-Augmented Generation) platform built on Memgraph with deep Python integration

---

## Executive Summary

This document outlines the design of a full-featured RAG platform leveraging Memgraph's graph database capabilities combined with enhanced Python runtime integration. The platform aims to be:
- **Easy to use**: Simple APIs for common RAG patterns
- **Fully customizable**: Extensible at every layer
- **Ecosystem-integrated**: Works with popular ML/AI frameworks
- **Production-ready**: Scalable, observable, and maintainable

---

## RAG Platform Architecture

### High-Level Overview

```
┌───────────────────────────────────────────────────────────────────┐
│                       Application Layer                          │
│  ┌────────────┐  ┌────────────┐  ┌──────────┐  ┌─────────────┐  │
│  │ Chat API   │  │ Search API │  │ Admin UI │  │  Notebooks  │  │
│  └──────┬─────┘  └──────┬─────┘  └────┬─────┘  └──────┬──────┘  │
└─────────┼────────────────┼─────────────┼───────────────┼─────────┘
          │                │             │               │
          └────────────────┴─────────────┴───────────────┘
                                 │
┌────────────────────────────────┼─────────────────────────────────┐
│                    RAG Orchestration Layer                       │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Python RAG Framework (Built on Memgraph)                 │   │
│  │  - Query understanding & decomposition                    │   │
│  │  - Multi-hop retrieval strategies                         │   │
│  │  - Context assembly & ranking                             │   │
│  │  - Response generation & streaming                        │   │
│  └───────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
┌────────────────────────────────┼─────────────────────────────────┐
│                         Memgraph Core                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │   Knowledge  │  │   Vector     │  │   Metadata &         │   │
│  │   Graph      │  │   Indices    │  │   Provenance         │   │
│  │              │  │              │  │   Tracking           │   │
│  │  Documents   │  │  Embeddings  │  │                      │   │
│  │  Entities    │  │  Similarity  │  │  Source tracking     │   │
│  │  Relations   │  │  Search      │  │  Versioning          │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │         Python Runtime Integration                       │   │
│  │  • Embedding generation (HuggingFace, OpenAI)           │   │
│  │  • Text processing (chunking, cleaning)                 │   │
│  │  • Custom retrieval algorithms                          │   │
│  │  • LLM integration (OpenAI, Anthropic, local models)    │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
                                 │
┌────────────────────────────────┼─────────────────────────────────┐
│                    Data Ingestion Pipeline                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │   Loaders    │→ │  Processors  │→ │   Graph Builder      │   │
│  │              │  │              │  │                      │   │
│  │  PDFs        │  │  Chunking    │  │  Entity extraction   │   │
│  │  Web pages   │  │  Cleaning    │  │  Relationship        │   │
│  │  APIs        │  │  Embedding   │  │  detection           │   │
│  │  Databases   │  │  Enrichment  │  │  Knowledge fusion    │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Knowledge Graph Schema

#### 1.1 Base Schema

```cypher
// Document nodes
CREATE (d:Document {
    id: "doc_123",
    title: "Introduction to Graph RAG",
    source: "https://example.com/article",
    source_type: "web",
    created_at: datetime(),
    updated_at: datetime(),
    metadata: {author: "Jane Doe", published: "2024-01-15"}
});

// Chunk nodes (text segments)
CREATE (c:Chunk {
    id: "chunk_456",
    content: "Graph RAG combines the power of...",
    position: 0,
    tokens: 150,
    embedding: [0.123, -0.456, ...],  // Vector embedding
    embedding_model: "text-embedding-3-small"
});

// Entity nodes (extracted concepts)
CREATE (e:Entity {
    id: "entity_789",
    name: "Graph RAG",
    type: "Concept",
    canonical_form: "Graph RAG",
    aliases: ["GraphRAG", "Graph-based RAG"],
    description: "A retrieval-augmented generation approach using graphs"
});

// Relationships
CREATE (d)-[:HAS_CHUNK {position: 0}]->(c);
CREATE (c)-[:MENTIONS {confidence: 0.95, context: "definition"}]->(e);
CREATE (e)-[:RELATED_TO {strength: 0.8, type: "enables"}]->(other_entity);
CREATE (c)-[:NEXT_CHUNK]->(next_c);
CREATE (c)-[:SIMILAR_TO {score: 0.87}]->(other_c);
```

#### 1.2 Extended Schema for Advanced RAG

```cypher
// User interaction tracking
CREATE (u:User {id: "user_1", preferences: {...}});
CREATE (q:Query {
    id: "query_1",
    text: "What is Graph RAG?",
    timestamp: datetime(),
    embedding: [...]
});
CREATE (u)-[:ASKED]->(q);
CREATE (q)-[:RETRIEVED]->(c);
CREATE (q)-[:GENERATED {
    response: "Graph RAG is...",
    model: "claude-3-5-sonnet",
    latency_ms: 1234
}]->(r:Response);

// Quality and feedback
CREATE (u)-[:RATED {score: 5, feedback: "Very helpful"}]->(r);

// Provenance and lineage
CREATE (r)-[:CITED {relevance: 0.92}]->(c);
CREATE (c)-[:DERIVED_FROM]->(d);

// Temporal versioning
CREATE (e)-[:VERSION_OF {version: 2, updated_at: datetime()}]->(e_prev);
```

### 2. Python RAG Framework

#### 2.1 Core API

```python
# /home/user/memgraph/python/memgraph_rag/__init__.py

from memgraph import Graph
from typing import List, Dict, Optional, AsyncIterator
import asyncio

class MemgraphRAG:
    """Main RAG orchestration class"""

    def __init__(self, graph: Graph, config: RAGConfig):
        self.graph = graph
        self.config = config
        self.embedder = self._init_embedder()
        self.llm = self._init_llm()
        self.retriever = Retriever(graph, self.embedder)
        self.generator = Generator(self.llm)

    async def query(self,
                   question: str,
                   user_id: Optional[str] = None,
                   strategy: str = "hybrid") -> RAGResponse:
        """
        Main query interface

        Strategies:
        - "vector": Pure vector similarity search
        - "graph": Graph-based traversal
        - "hybrid": Combined vector + graph
        - "adaptive": AI-selected strategy
        """

        # 1. Query understanding
        query_plan = await self.understand_query(question)

        # 2. Retrieval
        context = await self.retriever.retrieve(
            query=question,
            plan=query_plan,
            strategy=strategy,
            top_k=self.config.retrieval_top_k
        )

        # 3. Context ranking and filtering
        ranked_context = await self.rank_context(context, question)

        # 4. Generation
        response = await self.generator.generate(
            question=question,
            context=ranked_context,
            streaming=self.config.streaming
        )

        # 5. Store interaction for learning
        await self.store_interaction(question, context, response, user_id)

        return response

    async def query_stream(self,
                          question: str,
                          **kwargs) -> AsyncIterator[str]:
        """Streaming query interface"""
        async for chunk in self.generator.generate_stream(question, **kwargs):
            yield chunk


class Retriever:
    """Handles multi-strategy retrieval"""

    def __init__(self, graph: Graph, embedder):
        self.graph = graph
        self.embedder = embedder

    async def retrieve(self,
                      query: str,
                      plan: QueryPlan,
                      strategy: str,
                      top_k: int = 5) -> List[Context]:

        if strategy == "vector":
            return await self._vector_search(query, top_k)
        elif strategy == "graph":
            return await self._graph_traverse(query, plan, top_k)
        elif strategy == "hybrid":
            return await self._hybrid_search(query, plan, top_k)
        elif strategy == "adaptive":
            return await self._adaptive_search(query, plan, top_k)

    async def _vector_search(self, query: str, top_k: int) -> List[Context]:
        """Pure vector similarity search"""
        embedding = await self.embedder.encode(query)

        # Using Memgraph stored procedure
        results = await self.graph.call_procedure(
            "rag.vector_search",
            query_embedding=embedding,
            top_k=top_k
        )

        return [Context.from_chunk(r) for r in results]

    async def _graph_traverse(self,
                             query: str,
                             plan: QueryPlan,
                             top_k: int) -> List[Context]:
        """Graph-based traversal retrieval"""

        # Extract entities from query
        entities = await self.extract_entities(query)

        # Multi-hop traversal
        results = await self.graph.query("""
            // Find seed chunks mentioning query entities
            MATCH (e:Entity)
            WHERE e.name IN $entity_names
            MATCH (e)<-[:MENTIONS]-(seed:Chunk)

            // Expand to related chunks
            OPTIONAL MATCH (seed)-[:NEXT_CHUNK*1..3]->(neighbor:Chunk)
            OPTIONAL MATCH (seed)<-[:MENTIONS]-(entity)-[:RELATED_TO*1..2]->
                          (related_entity)-[:MENTIONS]->(related:Chunk)

            // Collect and score
            WITH seed, collect(DISTINCT neighbor) + collect(DISTINCT related) as expanded
            UNWIND [seed] + expanded as chunk

            // Deduplicate and score
            WITH DISTINCT chunk
            MATCH (chunk)-[:MENTIONS]->(e:Entity)
            WHERE e.name IN $entity_names
            WITH chunk, count(e) as entity_overlap

            ORDER BY entity_overlap DESC
            LIMIT $top_k
            RETURN chunk
        """, entity_names=[e.name for e in entities], top_k=top_k)

        return [Context.from_chunk(r['chunk']) for r in results]

    async def _hybrid_search(self,
                            query: str,
                            plan: QueryPlan,
                            top_k: int) -> List[Context]:
        """Combines vector and graph approaches"""

        # Parallel retrieval
        vector_results, graph_results = await asyncio.gather(
            self._vector_search(query, top_k * 2),
            self._graph_traverse(query, plan, top_k * 2)
        )

        # Reciprocal rank fusion
        fused = self._reciprocal_rank_fusion(
            [vector_results, graph_results],
            k=60  # RRF constant
        )

        return fused[:top_k]


class Generator:
    """Handles response generation"""

    def __init__(self, llm):
        self.llm = llm

    async def generate(self,
                      question: str,
                      context: List[Context],
                      streaming: bool = False) -> RAGResponse:
        """Generate response using LLM"""

        # Build prompt
        prompt = self._build_prompt(question, context)

        # Call LLM
        if streaming:
            return await self._generate_stream(prompt)
        else:
            return await self._generate_complete(prompt)

    def _build_prompt(self, question: str, context: List[Context]) -> str:
        """Construct RAG prompt"""
        context_text = "\n\n".join([
            f"[Source {i+1}: {ctx.source}]\n{ctx.content}"
            for i, ctx in enumerate(context)
        ])

        return f"""You are a helpful assistant. Answer the question based on the provided context.

Context:
{context_text}

Question: {question}

Answer: """

    async def _generate_complete(self, prompt: str) -> RAGResponse:
        """Non-streaming generation"""
        response = await self.llm.complete(prompt)
        return RAGResponse(
            text=response.text,
            model=response.model,
            citations=self._extract_citations(response.text)
        )

    async def _generate_stream(self, prompt: str) -> AsyncIterator[str]:
        """Streaming generation"""
        async for chunk in self.llm.stream(prompt):
            yield chunk
```

#### 2.2 Ingestion Pipeline

```python
# /home/user/memgraph/python/memgraph_rag/ingestion.py

class IngestionPipeline:
    """Handles document ingestion and processing"""

    def __init__(self, graph: Graph, config: IngestionConfig):
        self.graph = graph
        self.config = config
        self.chunker = self._init_chunker()
        self.embedder = self._init_embedder()
        self.entity_extractor = self._init_entity_extractor()

    async def ingest_document(self, doc: Document) -> str:
        """
        Full ingestion pipeline:
        1. Load document
        2. Clean and normalize
        3. Chunk into segments
        4. Generate embeddings
        5. Extract entities and relationships
        6. Build knowledge graph
        7. Create indices
        """

        # Step 1-2: Load and clean
        content = await self.load_and_clean(doc)

        # Step 3: Chunk
        chunks = await self.chunker.chunk(content, strategy=self.config.chunking_strategy)

        # Step 4: Embed chunks (batch processing)
        embeddings = await self.embedder.embed_batch([c.content for c in chunks])
        for chunk, embedding in zip(chunks, embeddings):
            chunk.embedding = embedding

        # Step 5: Extract entities
        entities = await self.entity_extractor.extract_batch(chunks)

        # Step 6: Build graph
        doc_id = await self._build_graph(doc, chunks, entities)

        # Step 7: Update indices (automatic via Memgraph)

        return doc_id

    async def _build_graph(self,
                          doc: Document,
                          chunks: List[Chunk],
                          entities: List[Entity]) -> str:
        """Build knowledge graph in Memgraph"""

        # Use Memgraph Python API
        async with self.graph.transaction() as tx:
            # Create document node
            doc_node = await tx.create_vertex(
                labels=["Document"],
                properties={
                    "id": doc.id,
                    "title": doc.title,
                    "source": doc.source,
                    "created_at": datetime.now(),
                    "metadata": doc.metadata
                }
            )

            # Create chunk nodes
            chunk_nodes = []
            for i, chunk in enumerate(chunks):
                chunk_node = await tx.create_vertex(
                    labels=["Chunk"],
                    properties={
                        "id": chunk.id,
                        "content": chunk.content,
                        "position": i,
                        "tokens": chunk.token_count,
                        "embedding": chunk.embedding.tolist(),
                        "embedding_model": self.config.embedding_model
                    }
                )
                chunk_nodes.append(chunk_node)

                # Link document to chunk
                await tx.create_edge(
                    doc_node,
                    chunk_node,
                    "HAS_CHUNK",
                    properties={"position": i}
                )

                # Link chunks sequentially
                if i > 0:
                    await tx.create_edge(
                        chunk_nodes[i-1],
                        chunk_node,
                        "NEXT_CHUNK"
                    )

            # Create entity nodes and relationships
            entity_map = {}
            for entity in entities:
                # Check if entity already exists (deduplication)
                existing = await tx.query("""
                    MATCH (e:Entity {canonical_form: $canonical})
                    RETURN e
                """, canonical=entity.canonical_form)

                if existing:
                    entity_node = existing[0]['e']
                else:
                    entity_node = await tx.create_vertex(
                        labels=["Entity", entity.type],
                        properties={
                            "id": entity.id,
                            "name": entity.name,
                            "type": entity.type,
                            "canonical_form": entity.canonical_form,
                            "description": entity.description
                        }
                    )

                entity_map[entity.id] = entity_node

                # Link chunks to entities
                for mention in entity.mentions:
                    chunk_node = chunk_nodes[mention.chunk_idx]
                    await tx.create_edge(
                        chunk_node,
                        entity_node,
                        "MENTIONS",
                        properties={
                            "confidence": mention.confidence,
                            "context": mention.context
                        }
                    )

            # Create entity relationships
            for relation in self.entity_extractor.extract_relations(entities):
                await tx.create_edge(
                    entity_map[relation.subject_id],
                    entity_map[relation.object_id],
                    relation.predicate.upper(),
                    properties={
                        "strength": relation.confidence,
                        "context": relation.context
                    }
                )

            # Compute chunk similarities (optional, expensive)
            if self.config.compute_chunk_similarities:
                await self._compute_similarities(tx, chunk_nodes, chunks)

        return doc.id

    async def _compute_similarities(self, tx, chunk_nodes, chunks):
        """Compute and store chunk-to-chunk similarities"""
        from sklearn.metrics.pairwise import cosine_similarity
        import numpy as np

        embeddings = np.array([c.embedding for c in chunks])
        similarities = cosine_similarity(embeddings)

        # Only store top-k similarities per chunk
        for i, chunk_node_i in enumerate(chunk_nodes):
            # Get top-k most similar (excluding self)
            similar_indices = np.argsort(similarities[i])[::-1][1:self.config.similarity_top_k+1]

            for j in similar_indices:
                if similarities[i][j] > self.config.similarity_threshold:
                    await tx.create_edge(
                        chunk_node_i,
                        chunk_nodes[j],
                        "SIMILAR_TO",
                        properties={"score": float(similarities[i][j])}
                    )
```

#### 2.3 Stored Procedures (Python/Cypher)

```python
# /home/user/memgraph/query_modules/rag.py

import mgp
import numpy as np
from typing import List

@mgp.read_proc
def vector_search(ctx: mgp.ProcCtx,
                  query_embedding: mgp.List[mgp.Number],
                  top_k: int = 5,
                  filter_labels: mgp.Nullable[mgp.List[str]] = None
                  ) -> mgp.Record(chunk=mgp.Vertex, score=float):
    """
    Vector similarity search using cosine similarity

    Optimized implementation using Memgraph indices
    """
    query_vec = np.array([float(x) for x in query_embedding])
    query_norm = np.linalg.norm(query_vec)

    # Prepare label filter
    label_filter = ""
    if filter_labels:
        label_filter = f":{':'.join(filter_labels)}"

    # Query all chunks with embeddings
    results = ctx.graph.execute(f"""
        MATCH (c:Chunk{label_filter})
        WHERE c.embedding IS NOT NULL
        RETURN c, c.embedding as embedding
    """)

    # Compute similarities
    scored_chunks = []
    for row in results:
        chunk = row['c']
        chunk_embedding = np.array(row['embedding'])
        chunk_norm = np.linalg.norm(chunk_embedding)

        # Cosine similarity
        if chunk_norm > 0 and query_norm > 0:
            similarity = np.dot(query_vec, chunk_embedding) / (query_norm * chunk_norm)
            scored_chunks.append((chunk, float(similarity)))

    # Sort and return top-k
    scored_chunks.sort(key=lambda x: x[1], reverse=True)

    for chunk, score in scored_chunks[:top_k]:
        yield mgp.Record(chunk=chunk, score=score)


@mgp.read_proc
def hybrid_search(ctx: mgp.ProcCtx,
                  query: str,
                  query_embedding: mgp.List[mgp.Number],
                  entities: mgp.List[str],
                  top_k: int = 5,
                  vector_weight: float = 0.5
                  ) -> mgp.Record(chunk=mgp.Vertex, score=float, method=str):
    """
    Hybrid search combining vector similarity and graph traversal

    Uses reciprocal rank fusion to combine results
    """

    # Vector search
    vector_results = list(ctx.call_procedure(
        "rag.vector_search",
        query_embedding=query_embedding,
        top_k=top_k * 2
    ))

    # Graph search via Cypher
    graph_results = ctx.graph.execute("""
        MATCH (e:Entity)
        WHERE e.name IN $entities
        MATCH (e)<-[:MENTIONS]-(c:Chunk)
        WITH c, count(DISTINCT e) as entity_count
        ORDER BY entity_count DESC
        LIMIT $limit
        RETURN c, entity_count
    """, entities=entities, limit=top_k * 2)

    # Reciprocal Rank Fusion
    k = 60
    chunk_scores = {}

    # Add vector scores
    for i, result in enumerate(vector_results):
        chunk_id = result['chunk'].id
        rrf_score = 1.0 / (k + i + 1)
        chunk_scores[chunk_id] = chunk_scores.get(chunk_id, 0) + vector_weight * rrf_score

    # Add graph scores
    for i, result in enumerate(graph_results):
        chunk = result['c']
        chunk_id = chunk.id
        rrf_score = 1.0 / (k + i + 1)
        chunk_scores[chunk_id] = chunk_scores.get(chunk_id, 0) + (1 - vector_weight) * rrf_score

    # Sort and return
    sorted_chunks = sorted(chunk_scores.items(), key=lambda x: x[1], reverse=True)

    chunk_map = {r['chunk'].id: r['chunk'] for r in vector_results}
    chunk_map.update({r['c'].id: r['c'] for r in graph_results})

    for chunk_id, score in sorted_chunks[:top_k]:
        yield mgp.Record(
            chunk=chunk_map[chunk_id],
            score=score,
            method="hybrid"
        )


@mgp.read_proc
def explain_retrieval(ctx: mgp.ProcCtx,
                     chunk_id: str,
                     query: str
                     ) -> mgp.Record(explanation=str, factors=mgp.Map):
    """
    Explain why a chunk was retrieved for a query

    Useful for debugging and improving retrieval
    """
    chunk = ctx.graph.get_vertex_by_id(chunk_id)
    if not chunk:
        return mgp.Record(explanation="Chunk not found", factors={})

    factors = {}

    # Check vector similarity
    # Check entity overlap
    # Check graph proximity
    # Check source authority
    # etc.

    explanation = f"Chunk '{chunk_id}' was retrieved because:\n"
    # Build explanation...

    yield mgp.Record(explanation=explanation, factors=factors)
```

### 3. Integration with ML/AI Ecosystem

#### 3.1 Embedding Providers

```python
# /home/user/memgraph/python/memgraph_rag/embeddings.py

from abc import ABC, abstractmethod
from typing import List
import numpy as np

class EmbeddingProvider(ABC):
    @abstractmethod
    async def encode(self, text: str) -> np.ndarray:
        pass

    @abstractmethod
    async def encode_batch(self, texts: List[str]) -> List[np.ndarray]:
        pass


class OpenAIEmbeddings(EmbeddingProvider):
    """OpenAI embedding API"""
    def __init__(self, model: str = "text-embedding-3-small", api_key: str = None):
        import openai
        self.client = openai.AsyncOpenAI(api_key=api_key)
        self.model = model

    async def encode(self, text: str) -> np.ndarray:
        response = await self.client.embeddings.create(
            model=self.model,
            input=text
        )
        return np.array(response.data[0].embedding)

    async def encode_batch(self, texts: List[str]) -> List[np.ndarray]:
        response = await self.client.embeddings.create(
            model=self.model,
            input=texts
        )
        return [np.array(item.embedding) for item in response.data]


class HuggingFaceEmbeddings(EmbeddingProvider):
    """Local HuggingFace models"""
    def __init__(self, model_name: str = "sentence-transformers/all-MiniLM-L6-v2"):
        from sentence_transformers import SentenceTransformer
        self.model = SentenceTransformer(model_name)

    async def encode(self, text: str) -> np.ndarray:
        return self.model.encode(text)

    async def encode_batch(self, texts: List[str]) -> List[np.ndarray]:
        return self.model.encode(texts)


class CohereEmbeddings(EmbeddingProvider):
    """Cohere embedding API"""
    # Implementation...


class VoyageAIEmbeddings(EmbeddingProvider):
    """Voyage AI embedding API"""
    # Implementation...
```

#### 3.2 LLM Providers

```python
# /home/user/memgraph/python/memgraph_rag/llms.py

class LLMProvider(ABC):
    @abstractmethod
    async def complete(self, prompt: str, **kwargs) -> LLMResponse:
        pass

    @abstractmethod
    async def stream(self, prompt: str, **kwargs) -> AsyncIterator[str]:
        pass


class AnthropicLLM(LLMProvider):
    """Anthropic Claude API"""
    def __init__(self, model: str = "claude-3-5-sonnet-20241022", api_key: str = None):
        import anthropic
        self.client = anthropic.AsyncAnthropic(api_key=api_key)
        self.model = model

    async def complete(self, prompt: str, **kwargs) -> LLMResponse:
        message = await self.client.messages.create(
            model=self.model,
            max_tokens=kwargs.get('max_tokens', 4096),
            messages=[{"role": "user", "content": prompt}]
        )
        return LLMResponse(
            text=message.content[0].text,
            model=self.model,
            usage=message.usage
        )

    async def stream(self, prompt: str, **kwargs) -> AsyncIterator[str]:
        async with self.client.messages.stream(
            model=self.model,
            max_tokens=kwargs.get('max_tokens', 4096),
            messages=[{"role": "user", "content": prompt}]
        ) as stream:
            async for text in stream.text_stream:
                yield text


class OpenAILLM(LLMProvider):
    """OpenAI GPT API"""
    # Similar implementation...


class LocalLLM(LLMProvider):
    """Local models via llama.cpp, vLLM, etc."""
    # Implementation...
```

### 4. Advanced RAG Patterns

#### 4.1 Multi-Hop Reasoning

```python
@mgp.read_proc
def multi_hop_retrieve(ctx: mgp.ProcCtx,
                      question: str,
                      max_hops: int = 3,
                      top_k_per_hop: int = 5
                      ) -> mgp.Record(path=mgp.Path, relevance=float):
    """
    Multi-hop retrieval following entity relationships

    Example: "What did the CEO of Tesla say about AI?"
    Hop 1: Find "CEO of Tesla" → Elon Musk
    Hop 2: Find "Elon Musk" → statements
    Hop 3: Filter statements → about "AI"
    """

    # Initial entity extraction
    entities = extract_entities_from_query(question)

    # Iterative retrieval
    for hop in range(max_hops):
        # Cypher query for multi-hop traversal
        results = ctx.graph.execute("""
            MATCH path = (seed:Chunk)-[:MENTIONS]->(:Entity)-[:RELATED_TO*1..2]->
                        (target:Entity)<-[:MENTIONS]-(related:Chunk)
            WHERE ... // Filter conditions
            RETURN path, score(path) as relevance
            ORDER BY relevance DESC
            LIMIT $top_k
        """, top_k=top_k_per_hop)

        # Yield paths
        for result in results:
            yield mgp.Record(path=result['path'], relevance=result['relevance'])
```

#### 4.2 Adaptive Retrieval

```python
class AdaptiveRetriever:
    """
    Uses LLM to decide retrieval strategy based on query type

    Query types:
    - Factual: Simple vector search
    - Comparison: Multi-entity graph traversal
    - Temporal: Time-based filtering + ranking
    - Analytical: Multi-hop reasoning
    """

    async def retrieve(self, query: str, top_k: int) -> List[Context]:
        # Classify query type
        query_type = await self.classify_query(query)

        # Select strategy
        if query_type == "factual":
            return await self.vector_search(query, top_k)
        elif query_type == "comparison":
            return await self.comparison_search(query, top_k)
        elif query_type == "temporal":
            return await self.temporal_search(query, top_k)
        elif query_type == "analytical":
            return await self.multi_hop_search(query, top_k)
        else:
            return await self.hybrid_search(query, top_k)

    async def classify_query(self, query: str) -> str:
        """Use LLM to classify query type"""
        prompt = f"""Classify this query into one of: factual, comparison, temporal, analytical

Query: {query}

Classification:"""
        response = await self.llm.complete(prompt, max_tokens=10)
        return response.text.strip().lower()
```

#### 4.3 Self-Correcting RAG

```python
class SelfCorrectingRAG:
    """
    RAG system that evaluates its own responses and retrieves more context if needed
    """

    async def query_with_verification(self, question: str) -> RAGResponse:
        max_iterations = 3

        for iteration in range(max_iterations):
            # Retrieve and generate
            context = await self.retrieve(question)
            response = await self.generate(question, context)

            # Self-evaluate
            is_sufficient = await self.evaluate_response(question, response, context)

            if is_sufficient:
                return response

            # Refine retrieval based on gaps
            question = await self.refine_query(question, response)

        return response  # Return best attempt

    async def evaluate_response(self, question: str, response: str, context: List[Context]) -> bool:
        """Check if response adequately answers question"""
        eval_prompt = f"""Does this response adequately answer the question?

Question: {question}
Response: {response}

Answer with YES or NO and explanation."""

        eval_result = await self.llm.complete(eval_prompt, max_tokens=100)
        return "YES" in eval_result.text
```

---

## Ecosystem Integration

### LangChain Integration

```python
# /home/user/memgraph/python/memgraph_rag/integrations/langchain.py

from langchain.vectorstores.base import VectorStore
from langchain.schema import Document

class MemgraphVectorStore(VectorStore):
    """LangChain VectorStore implementation using Memgraph"""

    def __init__(self, graph: Graph, embedder):
        self.graph = graph
        self.embedder = embedder

    def add_texts(self, texts: List[str], metadatas: List[dict] = None) -> List[str]:
        """Add texts to Memgraph"""
        # Implementation using ingestion pipeline
        pass

    def similarity_search(self, query: str, k: int = 4) -> List[Document]:
        """Search for similar documents"""
        embedding = self.embedder.encode(query)
        results = self.graph.call_procedure(
            "rag.vector_search",
            query_embedding=embedding.tolist(),
            top_k=k
        )
        return [Document(page_content=r['chunk'].properties['content']) for r in results]
```

### LlamaIndex Integration

```python
# /home/user/memgraph/python/memgraph_rag/integrations/llamaindex.py

from llama_index.core import VectorStoreIndex
from llama_index.core.vector_stores import VectorStore

class MemgraphVectorStore(VectorStore):
    """LlamaIndex VectorStore implementation"""
    # Similar to LangChain integration
```

### Haystack Integration

```python
# /home/user/memgraph/python/memgraph_rag/integrations/haystack.py

from haystack.document_stores import BaseDocumentStore

class MemgraphDocumentStore(BaseDocumentStore):
    """Haystack DocumentStore implementation"""
    # Implementation
```

---

## Example Use Cases

### Use Case 1: Conversational AI with Memory

```python
# Chat system that remembers conversation history via graph

class ConversationalRAG:
    async def chat(self, user_id: str, message: str) -> str:
        # Retrieve conversation history from graph
        history = await self.get_conversation_history(user_id, last_n=10)

        # Retrieve relevant context
        context = await self.rag.retrieve(message, user_context=history)

        # Generate response with history awareness
        response = await self.rag.generate(
            question=message,
            context=context,
            history=history
        )

        # Store interaction in graph
        await self.store_message(user_id, message, response)

        return response

    async def store_message(self, user_id: str, message: str, response: str):
        """Store conversation in graph for future context"""
        await self.graph.query("""
            MATCH (u:User {id: $user_id})
            CREATE (u)-[:SENT]->(m:Message {
                id: randomUUID(),
                content: $message,
                timestamp: datetime()
            })
            CREATE (m)-[:GOT_RESPONSE]->(r:Response {
                id: randomUUID(),
                content: $response,
                timestamp: datetime()
            })
        """, user_id=user_id, message=message, response=response)
```

### Use Case 2: Code Documentation Q&A

```python
# RAG system for codebase documentation

class CodeRAG(MemgraphRAG):
    async def ingest_codebase(self, repo_path: str):
        """Ingest code files with proper structure"""
        for file_path in discover_code_files(repo_path):
            # Parse code into AST
            ast = parse_code(file_path)

            # Extract functions, classes, imports
            entities = extract_code_entities(ast)

            # Create graph structure
            await self.build_code_graph(file_path, entities)

    async def query_code(self, question: str) -> RAGResponse:
        """Query codebase with code-specific logic"""
        # Identify code entities in question
        code_refs = extract_code_references(question)

        # Retrieve relevant code snippets
        context = await self.retrieve_code_context(question, code_refs)

        # Generate response with code examples
        return await self.generate_with_code(question, context)
```

### Use Case 3: Research Paper Analysis

```python
# RAG for academic papers with citation network

class ResearchRAG(MemgraphRAG):
    async def ingest_paper(self, paper: Paper):
        """Ingest paper with citation relationships"""
        # Parse paper sections
        # Extract citations
        # Build citation graph
        # Link to existing papers
        pass

    async def literature_review(self, topic: str) -> ResearchSummary:
        """Generate literature review on topic"""
        # Find seminal papers
        # Traverse citation network
        # Identify trends and gaps
        # Generate summary
        pass
```

---

## Deployment Architecture

### Development Environment

```yaml
# docker-compose.yml for local development

version: '3.8'

services:
  memgraph:
    image: memgraph/memgraph-platform:latest
    ports:
      - "7687:7687"  # Bolt
      - "3000:3000"  # Memgraph Lab
      - "7444:7444"  # HTTP
    volumes:
      - ./query_modules:/usr/lib/memgraph/query_modules
      - ./data:/var/lib/memgraph
    environment:
      - MEMGRAPH_LOG_LEVEL=TRACE

  rag-api:
    build: ./python
    ports:
      - "8000:8000"
    depends_on:
      - memgraph
    environment:
      - MEMGRAPH_HOST=memgraph
      - MEMGRAPH_PORT=7687
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}

  jupyter:
    image: jupyter/scipy-notebook:latest
    ports:
      - "8888:8888"
    volumes:
      - ./notebooks:/home/jovyan/work
    depends_on:
      - memgraph
```

### Production Deployment

```
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer (nginx)                    │
└────────┬────────────────────────────────────────────────────┘
         │
         ├──────────────────┬──────────────────┬───────────────┐
         │                  │                  │               │
         ▼                  ▼                  ▼               ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────┐
│  RAG API     │   │  RAG API     │   │  RAG API     │   │ ...  │
│  Instance 1  │   │  Instance 2  │   │  Instance 3  │   │      │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘   └──┬───┘
       │                  │                  │               │
       └──────────────────┴──────────────────┴───────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Memgraph HA Cluster                            │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐          │
│  │  Main    │─────▶│ Replica  │      │ Replica  │          │
│  │  Node    │      │  Node 1  │      │  Node 2  │          │
│  └──────────┘      └──────────┘      └──────────┘          │
│                            ▲                                │
│                            │                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Coordinator (Raft Consensus)                │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Observability and Monitoring

### Metrics to Track

```python
# Key RAG metrics

class RAGMetrics:
    # Retrieval metrics
    - retrieval_latency_ms: Histogram
    - retrieval_recall_at_k: Gauge (requires labeled data)
    - retrieval_precision_at_k: Gauge
    - average_chunk_relevance: Gauge

    # Generation metrics
    - generation_latency_ms: Histogram
    - tokens_generated: Counter
    - llm_api_errors: Counter

    # End-to-end metrics
    - query_latency_ms: Histogram
    - queries_per_second: Gauge
    - cache_hit_rate: Gauge

    # Quality metrics (user feedback)
    - user_satisfaction_score: Gauge
    - thumbs_up_rate: Gauge
    - response_helpfulness: Histogram
```

### Logging and Tracing

```python
import structlog
from opentelemetry import trace

logger = structlog.get_logger()
tracer = trace.get_tracer(__name__)

class InstrumentedRAG(MemgraphRAG):
    async def query(self, question: str, **kwargs) -> RAGResponse:
        with tracer.start_as_current_span("rag.query") as span:
            span.set_attribute("question.length", len(question))

            logger.info("rag.query.start", question=question[:100])

            try:
                response = await super().query(question, **kwargs)

                span.set_attribute("response.length", len(response.text))
                span.set_attribute("contexts.count", len(response.contexts))

                logger.info("rag.query.success",
                          latency_ms=response.latency_ms,
                          contexts_count=len(response.contexts))

                return response

            except Exception as e:
                span.record_exception(e)
                logger.error("rag.query.error", error=str(e))
                raise
```

---

## Conclusion

This RAG platform design leverages Memgraph's strengths:
- **Graph structure** for complex entity relationships and knowledge fusion
- **Vector search** for semantic similarity
- **Python integration** for ML/AI ecosystem compatibility
- **Triggers and procedures** for extensibility
- **Streaming support** for real-time ingestion

Combined with enhanced Python runtime integration (from Document 01), this creates a powerful, production-ready RAG platform that is:
- Easy to use (simple Python APIs)
- Fully customizable (extensible at every layer)
- Ecosystem-integrated (works with LangChain, LlamaIndex, etc.)
- Production-ready (scalable, observable, maintainable)
