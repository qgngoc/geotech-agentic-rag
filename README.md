# Geotech Agentic RAG

## About this project

This project is a geotechnical **Agentic RAG API** built with FastAPI. It is designed to answer general geotechnical questions using a local knowledge base and to run basic engineering calculations through tool-calling.

At runtime, the `/chat` workflow combines:
- **Input rails** to screen unsafe/off-topic prompts before generation.
- **OpenAI LLM + tool calling** to decide when to retrieve context or execute calculations.
- **Semantic retrieval from Qdrant** (client-scoped collections) using OpenAI embeddings.
- **Citation extraction and tracing logs** so each answer can be audited via `trace_id`.

The service also includes document ingestion and operations endpoints:
- `/index_document` to parse and index files (`pdf`, `txt`, `xlsx`, `md`, `pptx`, `docx`, `csv`) into the vector database.
- `/create_client` and `/delete_client/{client_id}` to manage tenant/client collections.
- `/metrics/{tracing_id}` to inspect generation/retrieval/calculation logs stored in MongoDB.
- `/get_llm_config_keys`, `/get_llm_config`, and `/get_read_file_config` for runtime config discovery.

The codebase follows a layered, ports-and-adapters style (`core` use cases/entities, `adapters` for REST/services/repositories, and `infrastructure` wiring), making it straightforward to swap implementations such as vector DB, LLM provider, or logging backend.

## Getting Started

### Setup
1. **Clone the repository:**
   ```bash
   git clone https://github.com/qgngoc/geotech-agentic-rag.git
   cd geotech-agentic-rag
   ```

2. **Configure environment variables:**
   - Copy the `.env` and `env.docker` files with your credentials (Replace your OpenAI API Key only).

3. **Start the services:**
   
    Note: In order to deply these smoothly, all sudo permission must be granted to this directory. Command to grant permission: `sudo chmod -R 777 ./`

   ```bash
   sh build.sh
   docker compose --env-file .env.docker up -d
   ```
   **Or Start the services locally:**
   - Install the requirements:
   ```bash
   pip install -r src/requirements.txt
   ```
   - Run service:
   ```bash
   cd src/
   python main.py
   ```

5. **Run usecases**

   API `/health`:
   ```
   curl -X 'GET' \
     'http://localhost:8000/health' \
     -H 'accept: application/json'
   ```
   
   API `/chat` (Important: `client.id` must be `"0"`.):
   ```
   curl -X 'POST' \
     'http://localhost:8000/api/v1/chat' \
     -H 'accept: application/json' \
     -H 'Content-Type: application/json' \
     -d '{
           "messages": [
               {"role": "user", "content": "If the load is 1500 and the modulus is 30000, what is the settlement?"}
           ],
           "client": {
                   "id": "0"
               },
               "rag_config": {
                   "llm_config": {
                       "model_path": "gpt-4.1-mini"
                   },
                   "top_k": 5
               }
           }'
   ```
   
   API `/metrics` (Replace `trace_id` by the `trace_id` returned from `/chat`):
   ```
   curl -X 'GET' \
     'http://localhost:8000/api/v1/metrics/{trace_id}' \
     -H 'accept: application/json'
   ```

Please checkout `API Documentations.md` to get more details of those APIs.

## Evaluation
   Note: The service must be deployed by Docker to run the evaluation script.
   
   Run retrieval evaluation scripts:
   ```bash
   python retrieval_evaluation/eval.py
   ```
   Results are saved in `retrieval_evaluation/data/`.
   Results stats:
   ```
   +---------+---------+-----------+
   |   Top K |   Total |   Matched |
   +=========+=========+===========+
   |       1 |       8 |         7 |
   +---------+---------+-----------+
   |       2 |       8 |         8 |
   +---------+---------+-----------+
   |       3 |       8 |         8 |
   +---------+---------+-----------+
   ```

## Unit Testing
   Run unit tests:
   ```bash
   PYTHONPATH=src/ pytest
   ```
