# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Jupyter Notebook demo comparing Elasticsearch chunking strategies for providing context to agentic AI: native `semantic_text` (automatic and pre-chunked) vs. manual external chunking with `dense_vector` fields. Three scenarios progress from simplest (semantic_text auto-chunking with semantic highlighting) to most flexible (one-doc-per-chunk with hybrid BM25+vector search and reranking). The narrative arc shows the tradeoffs — `semantic_text` retrieves relevant chunks with zero pipeline overhead, while manual chunking unlocks per-chunk metadata, independent chunk lifecycle, and hybrid retrieval with reranking.

## Architecture

- **demo.ipynb** — Single notebook containing all 5 scenarios, run sequentially
- **terraform/** — Elastic Cloud Serverless provisioning (outputs API key and Cloud ID to `.env`)
- **assets/plan.md** — Detailed implementation plan for each notebook scenario

### Scenario Flow

1. **Scenario 1** — `semantic_text` automatic chunking (ES handles chunking, embedding, and retrieval via semantic highlighting)
2. **Scenario 2** — `semantic_text` with pre-chunked arrays (regex split on markdown headings, ES handles embedding)
3. **Scenario 3** — `semchunk` library → one ES doc per chunk with `dense_vector` + field collapsing for dedup
4. **Scenario 4** — Hybrid BM25+vector search with reranking (queries Scenario 3's index — no reindexing needed)
5. **Scenario 5** — LangChain `RecursiveCharacterTextSplitter` → nested chunk objects (niche pattern illustrating nested kNN tradeoffs)

### Key Patterns

- Scenarios 1-2 use `semantic_text` with ES-managed inference; Scenarios 3-5 use `dense_vector` with the `.jina-embeddings-v5-text-small` inference endpoint
- Manual chunking scenarios use `dense_vector` with `int8_hnsw` quantization
- Scenario 3 demonstrates field collapsing on `parent_id` for native dedup
- Scenario 4 demonstrates hybrid retrieval with `text_similarity_reranker` — a query-level capability on any index with `text` + `dense_vector` fields
- Scenario 5 illustrates nested kNN querying with `inner_hits`

## Commands

```bash
# Activate virtual environment
source .venv/bin/activate

# Run the notebook
jupyter notebook demo.ipynb

# Provision infrastructure
terraform -chdir=terraform init -upgrade
terraform -chdir=terraform apply -auto-approve

# Tear down infrastructure
terraform -chdir=terraform destroy -auto-approve
```

## Dependencies

Python packages (installed via `requirements.txt` in the notebook):
- `elasticsearch` — ES Python client
- `semchunk` — semantic-aware text chunking (Scenario 2)
- `langchain-text-splitters` — text splitting (Scenario 5)

Infrastructure: Terraform with Elastic Cloud provider.
