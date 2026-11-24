# SupportPilot — Agentic RAG

An Agentic Retrieval-Augmented Generation (RAG) system for IT support and technical knowledge assistance.

SupportPilot uses **LangGraph, Groq, Pinecone, Hugging Face embeddings, and Tavily** to retrieve relevant information, evaluate evidence, and generate context-grounded answers.

The system follows a private-knowledge-first approach: it checks the indexed knowledge base before using external web search.

## Overview

Traditional RAG systems generally follow a simple process:

```text
User Question
     ↓
Retrieve Documents
     ↓
Generate Answer
```

SupportPilot extends this process with an agentic workflow that can make decisions based on retrieval quality.

```text
User Question
     ↓
Question Routing
     ↓
Private Knowledge Retrieval
     ↓
Evidence Evaluation
     ↓
Is the Evidence Sufficient?
     │
     ├── Yes → Generate Answer from Private Knowledge
     │
     └── No  → Tavily Web Search
                    ↓
              Evaluate Web Evidence
                    ↓
              Is Evidence Sufficient?
                    │
                    ├── Yes → Generate Web-Grounded Answer
                    │
                    └── No  → Rewrite Query and Retry
                                   ↓
                              Final Answer or
                              Insufficient Evidence
```

## Key Features

- **Question Routing:** Identifies whether a message requires knowledge retrieval or a direct response.
- **Private Knowledge Retrieval:** Searches relevant document chunks stored in Pinecone.
- **Semantic Search:** Uses Hugging Face sentence embeddings to retrieve semantically related content.
- **Evidence Grading:** Evaluates whether retrieved private knowledge is sufficient to answer the question.
- **Web Search Fallback:** Uses Tavily when private knowledge does not provide sufficient evidence.
- **Web Evidence Grading:** Evaluates external search results before generating an answer.
- **Query Rewriting:** Improves the search query when the retrieved evidence is insufficient.
- **Retry Control:** Limits query rewriting to avoid endless retrieval loops.
- **Source Tracking:** Tracks the selected answer source and workflow state.
- **LangGraph Orchestration:** Connects the individual processing steps into a stateful agent workflow.

## Architecture

```text
                 User Question
                       |
                       v
               Question Router
                       |
             +---------+---------+
             |                   |
          Direct              Knowledge
          Response              Base
                                 |
                                 v
                        Query Embedding
                                 |
                                 v
                         Pinecone Search
                                 |
                                 v
                         Retrieved Chunks
                                 |
                                 v
                         Evidence Grader
                                 |
                    +------------+------------+
                    |                         |
                 Sufficient               Insufficient
                    |                         |
                    v                         v
               Private KB               Tavily Search
                 Answer                      |
                                             v
                                      Web Evidence Grader
                                             |
                                  +----------+----------+
                                  |                     |
                              Sufficient           Insufficient
                                  |                     |
                                  v                     v
                              Web Answer          Query Rewrite
                                                        |
                                                        v
                                                Retry Retrieval
                                                        |
                                                        v
                                                Final Evaluation
```

## Technology Stack

| Component | Technology | Purpose |
|---|---|---|
| Programming Language | Python | Main implementation |
| Agent Framework | LangGraph | Stateful agent workflow |
| LLM | Groq | Routing, evidence grading, query rewriting and answer generation |
| Embedding Model | `sentence-transformers/all-MiniLM-L6-v2` | Local text embeddings |
| Vector Database | Pinecone | Semantic knowledge retrieval |
| Web Search | Tavily | External information retrieval |
| LLM Integration | LangChain | Model and tool integration |
| Data Validation | Pydantic | Structured model outputs |
| Document Loading | WebBaseLoader | Loading source content |
| Text Splitting | RecursiveCharacterTextSplitter | Splitting documents into chunks |
| Configuration | python-dotenv | Environment variable management |

## Project Structure

```text
SupportPilot/
│
├── notebook/
│   ├── SupportPilot_Agentic_RAG.ipynb
│   └── requirements.txt
│
├── .env
├── .gitignore
└── README.md
```

## How It Works

### 1. Document Loading

The notebook loads source content using LangChain's `WebBaseLoader`.

The current demonstration uses the LangGraph Agentic RAG documentation as its knowledge source.

### 2. Text Chunking

The loaded content is divided into smaller chunks using `RecursiveCharacterTextSplitter`.

Current configuration:

| Parameter | Value |
|---|---:|
| Chunk Size | 900 |
| Chunk Overlap | 120 |
| Start Index | Enabled |

Chunking helps divide large documents into manageable passages for retrieval.

### 3. Embedding Generation

The project uses the Hugging Face model:

`sentence-transformers/all-MiniLM-L6-v2`

The embedding model converts text into numerical vectors.

- Embedding dimension: 384
- Runs locally
- Uses CPU and system memory
- Embeddings are normalized

### 4. Pinecone Vector Storage

The generated embeddings and document chunks are stored in Pinecone.

The vector database uses cosine similarity for semantic retrieval.

Stable chunk IDs are generated using the source, chunk position, and content hash to avoid creating new IDs for unchanged chunks during repeated ingestion.

### 5. Question Routing

The LangGraph workflow first determines whether the user message is:

- A direct conversation, such as a greeting or simple interaction.
- A knowledge-based question requiring document retrieval.

Direct messages bypass unnecessary retrieval operations.

### 6. Private Knowledge Retrieval

For knowledge-based questions, the system retrieves relevant chunks from Pinecone.

The current retriever is configured to return up to four matching chunks.

### 7. Private Evidence Grading

An LLM evaluates whether the retrieved passages contain enough information to answer the original question.

- Sufficient evidence: Generate an answer using private knowledge.
- Insufficient evidence: Continue to external web search.

### 8. Tavily Web Search

When private knowledge is insufficient, the workflow uses Tavily to search for external information.

The returned search content is evaluated before being used to generate an answer.

### 9. Web Evidence Grading

The web evidence grader checks whether the retrieved search results are sufficient to answer the question.

If the evidence is insufficient, the workflow can rewrite the search query and retry retrieval.

### 10. Query Rewriting and Retry Control

The query rewriting step improves the search wording while preserving the original intent.

The current retry limit is:

```python
MAX_QUERY_REWRITES = 1
```

This prevents endless rewriting and retrieval loops.

If sufficient evidence is still unavailable, the workflow returns an insufficient-evidence response instead of generating an unsupported answer.

## Installation and Setup

### Prerequisites

- Python 3.12
- uv package manager
- Groq API key
- Pinecone API key
- Tavily API key
- Internet connection

### 1. Clone the Repository

```bash
git clone YOUR_REPOSITORY_URL
```

Navigate into the project:

```bash
cd SupportPilot
```

### 2. Create a Virtual Environment

```bash
uv venv --python 3.12
```

Activate it on Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

### 3. Install Dependencies

```bash
uv pip install -r .\notebook\requirements.txt
```

Install the Jupyter kernel:

```bash
uv pip install ipykernel
```

### 4. Configure Environment Variables

Create a `.env` file in the project root.

```env
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
PINECONE_API_KEY=your_pinecone_api_key

GROQ_MODEL=openai/gpt-oss-20b

PINECONE_INDEX=supportpilot-knowledge
PINECONE_NAMESPACE=langgraph-docs
```

Replace the example values with your own credentials.

The Groq model identifier can be changed to another model available in your Groq account.

**Security:** Never upload your `.env` file or expose API keys in a public repository.

### 5. Run the Notebook

Open the project in VS Code or Jupyter Notebook.

Open:

```text
notebook/SupportPilot_Agentic_RAG.ipynb
```

Select the Python 3.12 virtual environment as the notebook kernel.

Run the notebook cells in order, starting with dependency imports and configuration.

## Configuration

| Setting | Current Value |
|---|---|
| Embedding Model | `sentence-transformers/all-MiniLM-L6-v2` |
| Embedding Dimensions | 384 |
| Vector Database | Pinecone |
| Similarity Metric | Cosine |
| Retrieval Top K | 4 |
| Chunk Size | 900 |
| Chunk Overlap | 120 |
| Maximum Query Rewrites | 1 |
| Web Search Provider | Tavily |
| LLM Provider | Groq |

## Example Questions

Try these questions after running the notebook.

**Direct conversation**

```text
Hello! What can you help me with?
```

Expected behavior: The router selects the direct-response path.

**Private knowledge retrieval**

```text
What should an Agentic RAG system do when retrieved documents are not relevant?
```

Expected behavior: The system retrieves relevant chunks from the indexed knowledge base and evaluates the evidence.

**External web search**

```text
What is Tavily Search and how can it be used by an AI agent?
```

Expected behavior: The system attempts private retrieval first and uses Tavily if the private evidence is insufficient.

The exact workflow path depends on the retrieved content and LLM evidence evaluation.

## Current Implementation Scope

### Implemented

- Document loading from a web source.
- Recursive text chunking.
- Local Hugging Face embeddings.
- Pinecone vector database integration.
- Semantic document retrieval.
- Question routing using structured LLM output.
- Private evidence grading.
- Tav