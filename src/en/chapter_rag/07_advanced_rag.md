# 6.7 Advanced RAG: GraphRAG and Agentic RAG Engineering Practice

> **Goal**: Move beyond the naive "retrieve then generate" pipeline and master two production-grade RAG architectures: knowledge-graph-enhanced retrieval (**GraphRAG**) and Agent-driven adaptive retrieval (**Agentic RAG**).

---

## Why Advanced RAG Is Needed

The core limitation of naive RAG can be summarized in one sentence: **it treats every question as a local question**.

> **Hidden assumption of naive RAG**: the answer must be contained in a few adjacent text chunks.

Real-world questions often break this assumption:

- "Where do all departments overlap in this report?" requires a global view.
- "What indirect partnership exists between company A and company B?" requires relationship reasoning.
- "Why is the final conclusion X? Derive it step by step." requires multi-hop retrieval.
- "Are there contradictions in the user manual?" requires comparison across many locations.

Two advanced architectures address these needs:

| Problem Type | Architecture | Core Idea |
|-------------|--------------|-----------|
| Global relationships / cross-document reasoning | **GraphRAG** | Convert knowledge into a graph and retrieve through graph structure |
| Multi-hop / adaptive / uncertain questions | **Agentic RAG** | Let an Agent dynamically decide the retrieval strategy |

---

## Part I: GraphRAG

### From Chunks to Knowledge Graphs

GraphRAG starts from one key insight: **chunks preserve knowledge but lose relationships**.

Traditional vector search may know that two paragraphs are semantically close, but it does not explicitly represent the relation between entities. GraphRAG extracts entities and relationships, then stores them as nodes and edges.

```text
Text: "Apple acquired Shazam. Apple is a competitor of Google."

Graph:
Apple  --[acquired]-->  Shazam
Apple  --[competitor_of]-->  Google
```

Once relationships are explicit, the retriever can perform graph traversal, community detection, and global summarization.

### Local Search vs Global Search

GraphRAG typically supports two retrieval modes:

```python
"""
GraphRAG retrieval modes

Local Search
  Best for specific questions:
  "What role did Alice play in the project?"
  Flow: find the Alice node → traverse neighbors → assemble related evidence

Global Search
  Best for global questions:
  "Who are the most central collaborators in the whole project?"
  Flow: summarize graph communities → map-reduce over summaries → synthesize answer
"""
```

---

## A Minimal GraphRAG Pipeline

```python
from dataclasses import dataclass, field


@dataclass
class Entity:
    name: str
    type: str
    description: str = ""


@dataclass
class Relation:
    source: str
    target: str
    relation: str
    evidence: str


@dataclass
class KnowledgeGraph:
    entities: dict[str, Entity] = field(default_factory=dict)
    relations: list[Relation] = field(default_factory=list)

    def add_entity(self, entity: Entity) -> None:
        self.entities[entity.name] = entity

    def add_relation(self, relation: Relation) -> None:
        self.relations.append(relation)

    def neighbors(self, entity_name: str) -> list[Relation]:
        return [
            rel for rel in self.relations
            if rel.source == entity_name or rel.target == entity_name
        ]
```

In production, the extraction step is usually performed by an LLM or an information extraction model, then stored in a graph database such as Neo4j or a graph-enabled search system.

---

## Part II: Agentic RAG

Naive RAG always follows the same pipeline. Agentic RAG treats retrieval as a decision-making process.

```text
Question
  ↓
Analyze intent and uncertainty
  ↓
Choose retrieval strategy
  ├── vector search
  ├── keyword search
  ├── graph traversal
  ├── web search
  └── database query
  ↓
Inspect evidence
  ↓
Retrieve more if needed
  ↓
Generate answer with citations
```

```python
class AgenticRAGController:
    def __init__(self, retrievers: dict):
        self.retrievers = retrievers

    def answer(self, question: str) -> dict:
        plan = self.plan_retrieval(question)
        evidence = []

        for step in plan:
            retriever = self.retrievers[step["retriever"]]
            results = retriever.search(step["query"], top_k=step.get("top_k", 5))
            evidence.extend(results)

            if self.has_enough_evidence(question, evidence):
                break

        return self.generate_answer(question, evidence)

    def plan_retrieval(self, question: str) -> list[dict]:
        if "relationship" in question or "connection" in question:
            return [{"retriever": "graph", "query": question, "top_k": 8}]
        if "exact" in question or "quote" in question:
            return [{"retriever": "keyword", "query": question, "top_k": 5}]
        return [{"retriever": "vector", "query": question, "top_k": 5}]

    def has_enough_evidence(self, question: str, evidence: list) -> bool:
        return len(evidence) >= 3

    def generate_answer(self, question: str, evidence: list) -> dict:
        return {"question": question, "evidence": evidence, "answer": "..."}
```

---

## Retrieval Strategy Selection

| Query Type | Recommended Strategy |
|-----------|----------------------|
| Fact lookup | Vector search + keyword search |
| Exact quote | Keyword / BM25 search |
| Relationship reasoning | Graph traversal |
| Global summary | Community summaries + map-reduce |
| Fresh information | Web search |
| Structured facts | SQL / API query |
| Ambiguous question | Ask clarification or run multiple retrievers |

A mature Agentic RAG system should not use a single retriever for everything. It should select and combine retrieval strategies based on the question.

---

## Evaluation Criteria

Advanced RAG should be evaluated on more than answer correctness:

- **Faithfulness**: is the answer supported by evidence?
- **Coverage**: did retrieval include all necessary evidence?
- **Relationship accuracy**: are graph edges and multi-hop chains correct?
- **Citation quality**: can the user trace claims back to sources?
- **Cost and latency**: did the system over-retrieve?
- **Recovery**: can the Agent search again when evidence is insufficient?

---

## Engineering Best Practices

- Use naive RAG as the baseline before adding graph or Agent complexity.
- Keep raw text evidence even when using generated graph summaries.
- Store source references for every entity and relationship.
- Combine vector, keyword, graph, and metadata retrieval.
- Add an evidence sufficiency check before answer generation.
- Cache graph extraction and community summaries.
- Evaluate hallucination and citation accuracy separately.

---

## Chapter Takeaways

- Naive RAG works well for local questions but struggles with global, relational, and multi-hop questions.
- GraphRAG makes relationships explicit through entities, edges, and graph traversal.
- Agentic RAG lets an Agent dynamically choose retrieval strategies and iterate when evidence is insufficient.
- Production RAG should combine multiple retrieval methods and preserve citations.
- Advanced RAG must be evaluated on faithfulness, coverage, relationship accuracy, and cost.

---

*Previous: [6.6 RAG Paper Readings](./06_paper_readings.md)*
