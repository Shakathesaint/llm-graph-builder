# Project Overview

This project is a Knowledge Graph Builder that transforms unstructured data from various sources into a structured Knowledge Graph using Large Language Models (LLMs). The application is composed of a React frontend and a Python backend.

## Key Technologies

*   **Frontend:** React, Vite, Material-UI, Tailwind CSS
*   **Backend:** Python, FastAPI, LangChain, Neo4j
*   **Data Sources:** Local files, S3, GCS, Web pages, YouTube, Wikipedia
*   **Deployment:** Docker, Google Cloud Platform

## Architecture

The application is composed of two main services:

*   **Frontend:** A React application that provides the user interface for uploading files, managing data sources, and interacting with the knowledge graph.
*   **Backend:** A Python FastAPI application that handles the core logic of the application, including:
    *   Connecting to the Neo4j database
    *   Processing data from various sources
    *   Creating chunks of text
    *   Extracting entities and relationships using LLMs
    *   Saving the graph to Neo4j
    *   Providing a chat interface for querying the data

# Building and Running

## Docker Compose

The easiest way to run the application is using Docker Compose.

1.  **Configure Environment Variables:**
    *   Copy `backend/example.env` to `backend/.env` and `frontend/example.env` to `frontend/.env`.
    *   Update the environment variables in the `.env` files as needed. Key variables include:
        *   `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`: Neo4j connection details.
        *   `OPENAI_API_KEY`, `DIFFBOT_API_KEY`: API keys for LLM providers.
        *   `VITE_LLM_MODELS_PROD`: Comma-separated list of production LLM models to enable.
        *   `VITE_REACT_APP_SOURCES`: Comma-separated list of data sources to enable.

2.  **Run Docker Compose:**
    ```bash
    docker-compose up -d
    ```

## Running Backend and Frontend Separately

### Frontend

```bash
cd frontend
yarn
yarn run dev
```

### Backend

```bash
cd backend
python -m venv env
source env/bin/activate
pip install -r requirements.txt
uvicorn score:app --reload
```

# Development Conventions

*   **Linting:** The frontend uses ESLint and Prettier for code formatting and linting.
*   **Commits:** The project uses husky and lint-staged to run linting on pre-commit.
*   **Testing:** The backend has some tests in `test_commutiesqa.py` and `test_integrationqa.py`.

# User tools

When user asks for committing changes, you must refer to the instructions in .claude\commands\changelog-commit.md and execute the commands mentioned there literally.