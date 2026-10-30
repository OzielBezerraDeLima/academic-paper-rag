# Academic Paper RAG

**Local Retrieval-Augmented Generation for Scientific Paper Analysis**

A privacy-oriented RAG application for querying scientific papers using local language models, semantic search, and automated evaluation.

The system enables users to upload PDF documents, ask natural-language questions, and receive answers based on retrieved passages from the uploaded content.

**Tech stack:** Python · FastAPI · React · Ollama · ChromaDB · Hugging Face · RAGAS

---

## Overview

Academic literature reviews require researchers to read and compare numerous papers, often before determining whether each document is relevant to their work.

Academic Paper RAG addresses this challenge by providing a local question-answering system designed to help students and researchers:

- Identify relevant information in scientific papers.
- Ask questions about methodologies, findings, and references.
- Retrieve supporting passages from PDF documents.
- Explore research material without sending documents to external LLM APIs.

The application is designed to run locally after the initial setup, making it suitable for working with research drafts and other documents that should remain on the user's machine.

## System Architecture

```text
                   PDF Upload
                       |
                       v
              Text Extraction
                       |
                       v
             Document Chunking
                       |
                       v
          Ollama Embedding Model
              nomic-embed-text
                       |
                       v
                  ChromaDB
              Vector Storage
                       |
User Question ---------+
                       |
                       v
              Semantic Retrieval
                  Top-k Chunks
                       |
                       v
              Context + Question
                       |
                       v
            Qwen2.5-1.5B-Instruct
                 Local LLM
                       |
                       v
             Grounded Response
```

The application consists of a React frontend, a FastAPI backend, a document ingestion pipeline, a vector database, and a locally hosted language model.

### Main Components

| Component | Technology | Responsibility |
|---|---|---|
| Frontend | React | PDF upload and question-answering interface |
| Backend | FastAPI | API endpoints and application orchestration |
| Embeddings | Ollama / nomic-embed-text | Semantic representation of document chunks |
| Vector Database | ChromaDB | Persistent storage and similarity search |
| Generation | Qwen2.5-1.5B-Instruct | Context-grounded answer generation |
| Evaluation | RAGAS and custom metrics | Retrieval and answer quality assessment |

## Key Features

- **PDF ingestion:** Extracts and processes text from uploaded research documents.
- **Semantic retrieval:** Uses vector similarity to identify passages relevant to a question.
- **Local LLM inference:** Generates answers without relying on a hosted generation API.
- **Configurable retrieval:** Supports different top-k settings for experimentation.
- **Automated evaluation:** Calculates answer quality and retrieval metrics.
- **Experimental comparison:** Evaluates multiple local LLM judges and retrieval configurations.

## Technical Decisions

### Document Processing

Documents are split using `RecursiveCharacterTextSplitter` with the following configuration:

| Parameter | Value |
|---|---|
| Chunk size | 1,000 characters |
| Chunk overlap | 100 characters |
| Default top-k | 5 |
| Embedding model | nomic-embed-text |

The overlap helps preserve context across chunk boundaries. The default retrieval configuration balances context coverage and irrelevant information.

### Local-First Design

The system uses Ollama for embeddings, ChromaDB for vector storage, and a locally hosted language model for generation.

This design reduces dependence on external services and allows users to retain control over their documents.

### Evaluation Strategy

The evaluation pipeline includes traditional text-based metrics, retrieval metrics, and LLM-based evaluation using RAGAS.

Different evaluator models are compared to investigate how judge selection affects the resulting scores.

## Experimental Evaluation

The repository includes experiments comparing local LLM judges and retrieval configurations.

The documented judge comparison uses a reduced evaluation dataset containing four questions. These experiments are exploratory and should not be interpreted as evidence of generalization across scientific domains.

### Comparison of LLM Judges

All three configurations below use top-k = 5.

| Metric | Qwen 0.5B | Qwen 1.5B | Llama 3.2 |
|---|---:|---:|---:|
| Execution time | 17 min | 23 min | 38 min |
| Answer F1 | 0.39 | 0.39 | 0.39 |
| Retrieval F1 | 0.47 | 0.47 | 0.47 |
| Retrieval MRR | 0.63 | 0.63 | 0.63 |
| RAGAS Faithfulness | 1.00 | 1.00 | 0.81 |
| RAGAS Answer Relevancy | 0.15 | 0.58 | 0.60 |
| RAGAS Context Precision | 1.00 | 0.95 | 0.99 |
| RAGAS Context Recall | 1.00 | 1.00 | 1.00 |

### Key Observations

**1. Evaluation depends on the judge model.**

Answer Relevancy varied substantially between the smaller Qwen evaluator and the larger models. This illustrates the importance of considering evaluator reliability when interpreting LLM-as-judge results.

**2. Lexical overlap does not fully capture semantic correctness.**

Some responses included relevant details beyond the reference answer, resulting in lower token-overlap scores despite being supported by the document.

**3. Retrieval and generation require separate analysis.**

A system may retrieve appropriate passages while still generating incomplete answers. The project includes manual error analysis to distinguish these situations.

**4. Local evaluation introduces computational trade-offs.**

Larger evaluator models increased execution time in the documented CPU experiments.

The full experiment configurations and outputs are available in the `scripts/experimentos/` and `results/` directories.

## Getting Started

### Requirements

- Python 3.11+
- Node.js and npm
- Ollama
- Git

### 1. Clone the repository

```bash
git clone https://github.com/OzielBezerraDeLima/academic-paper-rag.git
cd academic-paper-rag
```

### 2. Install Python dependencies

```bash
python -m venv .venv
```

On Windows (PowerShell):

```powershell
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

On Linux/macOS:

```bash
source .venv/bin/activate
pip install -r requirements.txt
```

### 3. Prepare Ollama

Install Ollama and download the models used by the application:

```bash
ollama pull nomic-embed-text
ollama pull llama3.2
```

Additional models may be required for specific evaluation experiments.

The generation model configured through Hugging Face may also need to be downloaded during the initial setup.

### 4. Start the backend

```bash
uvicorn backend.main:app --reload --host 127.0.0.1 --port 8000
```

API documentation:

`http://localhost:8000/docs`

### 5. Start the frontend

Open a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Copy `frontend/.env.example` to `frontend/.env` and configure it as needed.

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | `/health` | Check API availability |
| POST | `/upload-pdf` | Upload and index a PDF |
| POST | `/chat` | Ask questions about an indexed document |

The upload endpoint returns a session identifier that is used for subsequent questions.

## Running Evaluations

The `avaliar.py` script executes a question-answering evaluation using a PDF document and a JSON dataset.

Example:

```bash
python avaliar.py \
  --pdf backend/pdf/artigoteste.pdf \
  --dataset backend/evaluation/dataset_exemplo.json \
  --output results.csv \
  --top-k 5
```

To enable RAGAS with a local evaluator:

```bash
python avaliar.py \
  --pdf backend/pdf/artigoteste.pdf \
  --dataset backend/evaluation/dataset_exemplo.json \
  --output results.csv \
  --top-k 5 \
  --use-ragas \
  --ragas-llm-model llama3.2:latest \
  --ragas-timeout 600
```

On Windows, these commands can be adapted to PowerShell or executed using an appropriate shell.

## Project Structure

```text
academic-paper-rag/
├── backend/
│   ├── rag/             # Retrieval and vector storage
│   ├── llm/             # Language model integration
│   ├── pdf/             # PDF processing
│   ├── evaluation/      # Evaluation datasets and metrics
│   └── main.py          # FastAPI application
├── frontend/            # React interface
├── scripts/
│   └── experimentos/    # Experimental configurations
├── results/             # Evaluation outputs
├── avaliar.py           # Evaluation entry point
├── requirements.txt
└── README.md
```

## Limitations

- The quality of generated answers depends on PDF text extraction and retrieval accuracy.
- Local inference can be slow without GPU acceleration.
- The experimental evaluation uses a small question set.
- LLM-as-judge metrics may be sensitive to evaluator selection.
- Automated metrics should be complemented with manual inspection.
- The application is an academic prototype, not a production-validated system.

## Future Improvements

Potential extensions include:

- Evaluation using larger and more diverse datasets.
- Improved answer completeness and citation handling.
- Comparison between embedding models.
- Containerization for reproducible deployment.
- Automated tests and continuous integration.
- Performance optimization for local inference.

## Academic Context

This project was developed as part of academic coursework involving artificial intelligence, natural language processing, and software development.

The repository is maintained as a portfolio project to demonstrate practical experience with RAG architectures, local LLM integration, semantic search, and experimental evaluation.

Original repository: [OzielLima/Trabalho1-ODS2](https://github.com/OzielLima/Trabalho1-ODS2).

The original commit history and contributor credits are preserved through the GitHub fork.
