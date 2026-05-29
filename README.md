# Multi-Format Document Q&A with Graph RAG

A production-style document Q&A system that combines **vector search (FAISS)** with **knowledge graph traversal (Neo4j)** for hybrid retrieval over enterprise documents — PDF, Word, PowerPoint, and Excel.

Most RAG systems use vector similarity alone. This system **augments retrieval with entity-relationship context** from a knowledge graph, producing richer, more grounded answers with source citations down to the document and page level.

Built during my AI/ML Engineering Internship at Bugendai Tech (Summer 2025).

---

## Problem

Enterprise document collections sit across mixed formats — PDF, Word, PowerPoint, Excel — and traditional search:
- Reads text without **understanding entity relationships**
- Misses **contextual links** between concepts across documents
- Doesn't retain **deep context** for downstream LLM reasoning

This system solves all three by combining vector similarity with a knowledge graph.

---

## Architecture
<img width="2560" height="1440" alt="Screenshot 2025-06-10 at 3 23 16 PM (2)" src="https://github.com/user-attachments/assets/4446a704-4a58-4ff0-bc0a-6c16a161d8b9" />


### Five-step pipeline

| Step | What happens |
|---|---|
| **1. Document Extraction** | Multi-format parsing — PyMuPDF (PDF), python-pptx (PowerPoint), openpyxl (Excel), python-docx / mammoth (Word) |
| **2. Cache Fingerprinting** | MD5 hash of folder contents (filenames + mtimes) → pickle cache validation |
| **3. Vector + Graph Storage** | FAISS embeddings (500-token recursive splitting with overlap) + Neo4j entity-relationship graph (spaCy NER in 50-chunk batches) |
| **4. Hybrid Retrieval** | Vector similarity (k-NN) + Neo4j graph traversal of related entities → context fusion |
| **5. LLM Generation** | OpenAI / Groq with source citations (document + page references) |

---

## Why Graph RAG?

Vanilla vector RAG retrieves chunks by semantic similarity alone, missing **relational context** between entities.

Example: ask *"How is Llama 2 used in legal document processing?"*
- Vanilla RAG finds chunks containing "Llama 2"
- **Graph RAG also surfaces** related entities (LlamaIndex, fine-tuning approaches, legal-domain models) from the knowledge graph, giving the LLM richer grounding

Measurable improvement in answer relevance vs vector-only RAG.

---

## Caching Strategy (engineering deep dive)

A core design decision: avoid re-embedding documents on every session.

**MD5-based cache validation:**
```python
def get_folder_hash(folder_path):
    hash_md5 = hashlib.md5()
    for filename in sorted(os.listdir(folder_path)):
        if filename.lower().endswith((".pdf", ".docx", ".pptx", ".xlsx")):
            hash_md5.update(filename.encode())
            hash_md5.update(str(os.path.getmtime(file_path)).encode())
    return hash_md5.hexdigest()
```

**Cache lifecycle:**
- **Cold path** (first run, no cache) — full ingestion: extract → chunk → embed → store in FAISS + Neo4j → serialize FAISS to pickle with MD5 hash as filename
- **Hot path** (cache hit) — instant pickle deserialization, skip all embedding/processing → sub-second startup
- **Invalidation** — automatic when folder contents change (hash mismatch); manual via UI

**Performance:**
- Cold start — minutes for large corpora
- Warm start — sub-second
- Cache files typically 10–20% of original document size

This turns enterprise startup from "wait minutes" into "ready immediately" — a meaningful UX improvement.

---

## Tech Stack

| Layer | Tools |
|---|---|
| **AI / ML** | LangChain · Hugging Face · spaCy NLP · FAISS |
| **LLM** | OpenAI / Groq (pluggable backend) |
| **Database** | Neo4j · FAISS vector store · Pickle cache |
| **Document processing** | PyMuPDF · python-pptx · openpyxl · python-docx · mammoth |
| **Web / UI** | Streamlit · Python · Pandas · Groq API |

---

## Features

- **Multi-format support** — PDF, Word, PowerPoint, Excel (live tested with mixed corpora)
- **Sub-2-second response** on cached document collections
- **Citation-backed answers** — every answer traces back to source document + page/section
- **Knowledge graph context shown** alongside answers for transparency
- **MD5-based cache** — skip redundant re-embedding when documents haven't changed
- **Cache and graph management** from the Streamlit UI (Clear Cache, Clear Graph)
- **Real-time processing feedback** during ingestion phases
- **Configurable retrieval parameters** (k, fetch_k, chunk size, overlap)

---

## Demo

![Streamlit UI](docs/streamlit_ui.png)

Real run: 5 mixed-format documents (1 PDF, 1 Word, 2 Excel, 1 PowerPoint) — cached vector store, sub-2-second response with source citations.

---

## Running Locally

### Prerequisites
- Python 3.10+
- Neo4j (Desktop or AuraDB) running locally
- OpenAI or Groq API key

### Install
\`\`\`bash
git clone https://github.com/Bhawna1201/multi-format-document-qa-graphrag.git
cd multi-format-document-qa-graphrag
pip install -r requirements.txt
python -m spacy download en_core_web_sm
\`\`\`

### Configure
Create a `.env` file:
\`\`\`
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password
OPENAI_API_KEY=your_key   # or GROQ_API_KEY
\`\`\`

### Run
\`\`\`bash
streamlit run app.py
\`\`\`

Drop your documents (PDF, Word, PowerPoint, Excel) into the configured folder and query via the UI.

---

## Design Decisions and Tradeoffs

- **FAISS over Pinecone/Chroma** — local, free, no network calls; tradeoff: no managed scaling
- **Neo4j over embedded graph DB** — mature Cypher tooling and visualization; tradeoff: operational overhead
- **OpenAI/Groq over local LLM** — higher answer quality and faster inference; tradeoff: API cost and data leaves the machine (mitigated by enterprise tiers)
- **spaCy NER + dependency parsing over LLM-based extraction** — deterministic, fast, free; tradeoff: occasional missed entities for non-standard text
- **MD5 hash (filename + mtime) over content hash** — sufficient for most workflows, far cheaper than hashing full document content
- **500-token chunks with overlap** — balance between context preservation and embedding precision
- **50-chunk batches for embedding** — throughput optimization without blowing GPU/API rate limits

---

## What I'd Improve

- **Eval framework** — labeled Q&A set with precision@k and answer faithfulness; A/B benchmark vs vector-only RAG to quantify the Graph RAG lift
- **Semantic chunking** — replace fixed-size chunks with embedding-similarity chunks for technical documents
- **Graph schema enrichment** — domain-specific entity types and relationship taxonomies (legal, healthcare, finance)
- **Async ingestion** — for very large document collections
- **Persistent vector store versioning** — track changes over time, support rollback
- **Multi-tenant access control** — partition embeddings and graph nodes by user/team for enterprise deployment

---

## License

MIT
