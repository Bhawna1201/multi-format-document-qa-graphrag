# Multi-Format Document Q&A with Graph RAG

A production-style document Q&A system that combines **vector search (FAISS)** with **knowledge graph traversal (Neo4j)** for hybrid retrieval over enterprise documents — PDF, Word, PowerPoint, and Excel.

Most RAG systems use vector similarity alone. This system **augments retrieval with entity-relationship context** from a knowledge graph, producing richer, more grounded answers with citations.

Built during my AI/ML Engineering Internship at Bugendai Tech (Summer 2025).

---

## Architecture

![Graph RAG Architecture](docs/architecture.png)

**Ingestion pipeline:**
1. Document upload (PDF, Word, PowerPoint, Excel)
2. MD5 hash check → skip if already processed (caching)
3. Text extraction via PyMuPDF
4. Chunking with LangChain's RecursiveCharacterTextSplitter
5. Dual storage:
   - **FAISS** — vector embeddings for semantic similarity
   - **Neo4j** — entity-relationship graph (entities extracted via spaCy NLP)

**Retrieval pipeline:**
1. User query → vector search in FAISS
2. User query → graph search in Neo4j → related entities
3. Combine vector results + graph-traversed entities → "enhanced query"
4. Pass enhanced context to Llama 3.2 (3B) via Ollama
5. Return citation-backed answer in Streamlit UI

---

## Why Graph RAG?

Standard vector-only RAG misses **relational context** between entities in documents. For example, asking "What does Llama 2 do for legal data?" — vanilla RAG retrieves chunks containing "Llama 2" by similarity, but a knowledge graph also surfaces *related* concepts (LlamaIndex, fine-tuning, legal domain models) the system might otherwise miss.

The hybrid approach measurably improves answer relevance vs vector-only retrieval.

---

## Tech Stack

| Layer | Tools |
|---|---|
| LLM | Llama 3.2 (3B) via Ollama (local inference) |
| Orchestration | LangChain |
| Vector store | FAISS |
| Knowledge graph | Neo4j (Cypher queries) |
| Entity extraction | spaCy NLP |
| Document parsing | PyMuPDF, RecursiveCharacterTextSplitter |
| UI | Streamlit |
| Language | Python |

---

## Features

- Multi-format ingestion (PDF, DOCX, PPTX, XLSX)
- Sub-2-second response times
- Citation-backed answers (source document + section)
- Knowledge graph context shown alongside answers
- MD5-based caching to skip re-ingestion of unchanged docs
- Cache management (clear vector store, clear graph) from the UI

---

## Running Locally

### Prerequisites
- Python 3.10+
- Neo4j (Desktop or AuraDB)
- Ollama with Llama 3.2 (3B) pulled locally

### Install
\`\`\`bash
git clone https://github.com/Bhawna1201/multi-format-document-qa-graphrag.git
cd multi-format-document-qa-graphrag
pip install -r requirements.txt
\`\`\`

### Configure
Create a `.env` file:
\`\`\`
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password
\`\`\`

### Pull the model (Ollama)
\`\`\`bash
ollama pull llama3.2:3b
\`\`\`

### Run
\`\`\`bash
streamlit run app.py
\`\`\`

---

## Demo

![Streamlit UI](docs/streamlit_ui.png)

---

## What I'd Improve

- **Eval framework**: add a structured eval comparing Graph RAG vs vector-only RAG on a labeled QA set (precision@k, answer faithfulness)
- **Chunking strategy**: experiment with semantic chunking vs fixed-size for technical documents
- **LLM**: test with larger Llama variants or hosted models (Claude, GPT-4) to compare answer quality
- **Knowledge graph schema**: enrich entity types and add relationship taxonomies for specific domains (legal, healthcare, finance)
- **Production hardening**: async ingestion, batched embeddings, persistent vector store with versioning

---

## License

MIT
