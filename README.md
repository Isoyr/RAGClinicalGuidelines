# RAGClinicalGuidelines

# RAG-Powered Clinical Conversational Agent for Obstetric Decision Support

A Retrieval-Augmented Generation (RAG) conversational agent that provides rapid, natural-language access to obstetric clinical practice guidelines. The system is built entirely on no-/low-code tooling (n8n) and compares two retrieval strategies — **standard RAG** and **HyDE-enhanced RAG** — evaluated with RAGAS-inspired metrics via an LLM-as-a-judge pipeline.

This repository accompanies the paper *"RAG-Powered Clinical Conversational Agent for Real-Time Clinical Decision Support"* (ColCACI 2026), by Isabel Sofía Tovar Sánchez, Rubén Manrique, Nathalia Ortega, and Luis Felipe Giraldo — Universidad de los Andes & Instituto Roosevelt, Bogotá, Colombia.

> **Note on credentials:** The workflow JSON files in this repo are **sanitized**. API keys are redacted, and project-specific identifiers (Supabase project ref, Google Drive folder ID, Google Docs/Sheets IDs, n8n instance subdomain) are replaced with placeholders such as `YOUR_PROJECT`, `YOUR_DRIVE_FOLDER_ID`, and `YOUR_GOOGLE_DOC_ID`. You must supply your own credentials and values before running them.

## Architecture

The agent operates in four stages: (1) the physician submits a clinical question in natural language; (2) the query is converted into a semantic vector and used for similarity search over the vector database; (3) the most relevant guideline chunks are retrieved; and (4) GPT-4o generates a response grounded in the retrieved context, with traceability back to the source documents.

**Stack:**
- **Orchestration:** n8n (visual workflow automation)
- **Vector store:** Supabase (pgvector)
- **Generation:** GPT-4o (agent), GPT-4o-mini (LLM judge)
- **Embeddings:** OpenAI `text-embedding-3-small` (1536 dimensions)
- **PDF parsing:** LlamaParse
- **Document source:** Google Drive
- **Evaluation I/O:** Google Sheets

## Repository contents

| File | Description |
|------|-------------|
| `DataIngestion.clean.json` | Ingestion pipeline: lists PDFs from Google Drive, parses them with LlamaParse, chunks the text, embeds it with OpenAI, and inserts it into the Supabase vector store. Includes a job-tracking branch that resumes pending parsing jobs. |
| `NormalRAGgit.clean.json` | Standard RAG agent: chat trigger → GPT-4o agent querying the Supabase vector store via OpenAI embeddings. |
| `HYDEgit.clean.json` | HyDE-enhanced RAG agent: generates a hypothetical answer before retrieval, embeds it, queries Supabase via an RPC/edge function, formats the top contexts, and passes them to the GPT-4o agent. |
| `EVALUACIONRAGASgit.clean.json` | LLM-as-a-judge evaluation pipeline: pulls questions and expert ground truth from Google Sheets, queries each agent, retrieves contexts, scores responses with GPT-4o-mini on RAGAS metrics, and appends results back to Google Sheets. Contains both the standard-RAG and HyDE evaluation flows. |
| `LICENSE` | License for this repository. |

## Data ingestion

Ingestion runs as a two-stage n8n workflow. In the first stage, PDFs are listed from Google Drive, downloaded individually, and uploaded to LlamaParse for structured text extraction (chosen for its handling of tables, multi-column layouts, and embedded figures). Each parse job ID is stored in Supabase. In the second stage, a separate workflow retrieves pending jobs, fetches the extracted text, chunks it with a recursive character text splitter, embeds the chunks with `text-embedding-3-small` (1536 dimensions), and inserts them into the Supabase vector store. Wait nodes mitigate rate-limiting during batch processing, and the workflow is re-executable for documents that fail.

The Supabase vector table must match the embedding dimensionality (1536); dimension mismatches between the embedding model and the table schema cause silent partial failures.

## Evaluation

Performance is assessed with four RAGAS-inspired metrics, computed by a GPT-4o-mini judge:

- **Faithfulness** — whether the response is factually supported by the retrieved context (hallucination detector).
- **Answer Relevancy** — how directly the answer addresses the question.
- **Context Precision** — the proportion of retrieved chunks that are relevant.
- **Context Recall** — whether retrieval surfaces all information needed to answer.

Ground truth was built by concatenating independent answers from multiple obstetric specialists (answering without access to the guideline documents), capturing the range of acceptable clinical answers.

## Results

Average RAGAS scores across the full question set:

| Metric | Standard RAG | HyDE-Enhanced RAG |
|--------|:---:|:---:|
| Faithfulness | 0.64 | 0.69 |
| Answer Relevancy | 0.70 | 0.66 |
| Context Precision | 0.58 | 0.63 |
| Context Recall | 0.50 | 0.50 |
| **RAGAS Score** | **0.60** | **0.62** |

HyDE improved faithfulness and context precision (+0.05 each), while standard RAG achieved higher answer relevancy. Both configurations obtained identical context recall (0.50), the weakest metric across both systems and the primary limitation of the current pipeline.

## Setup

1. **Import the workflows.** In n8n, import each `.clean.json` file (Workflows → Import from File).
2. **Configure credentials** in n8n for: OpenAI, Supabase, Google Drive, Google Sheets, and LlamaParse.
3. **Replace placeholders** in the imported workflows:
   - `YOUR_PROJECT.supabase.co` → your Supabase project URL
   - `YOUR_DRIVE_FOLDER_ID` → the Google Drive folder holding your PDFs
   - `YOUR_GOOGLE_DOC_ID` → your Google Sheets document IDs (questions and results)
   - `YOUR_N8N_INSTANCE.app.n8n.cloud` → your n8n instance / webhook URLs
   - any `REDACTED` header values → your API keys/tokens
4. **Prepare Supabase:** create a vector table sized to 1536 dimensions and the `match_documents` RPC used for similarity search. The HyDE agent additionally expects a `hydesearch` edge function.
5. **Run ingestion** (`DataIngestion`) to populate the vector store, then activate either agent workflow.
6. **Evaluate** by populating the questions/ground-truth Google Sheet and running `EVALUACIONRAGASgit`.

## Limitations

The system was evaluated with an automated LLM judge rather than human clinical review, and LLM judges are known to exhibit biases and inconsistencies. Low context recall is attributed to ground truth that extends beyond the ingested guidelines, chunking that can fragment coherent passages, and tabular/diagrammatic content that text-based parsing does not capture well. The agent should not be used for clinical decision-making without expert validation.

## Citation

If you use this work, please cite the accompanying paper:

> I. S. Tovar Sánchez, R. Manrique, N. Ortega, and L. F. Giraldo, "RAG-Powered Clinical Conversational Agent for Real-Time Clinical Decision Support," ColCACI, 2026.
