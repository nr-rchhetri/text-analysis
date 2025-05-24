# text-analysis

# Client Sentiment & Engagement Scoring System (Ollama)

A system for automatically analyzing Salesforce email text data to generate monthly client sentiment and engagement scores using locally hosted Ollama LLMs. Scores are updated weekly to Snowflake tables.

## Project Structure


├── config/                    # Configuration management
│   ├── init.py           # Exports config classes/objects
│   └── settings.py           # Ollama service, Snowflake, prompts, model names
├── jobs/                      # Individual scripts for each step in the workflow
│   ├── init.py
│   ├── ingest_sfdc_data.py   # Salesforce data ingestion
│   ├── preprocess_text.py    # Text cleaning and preparation
│   ├── score_sentiment.py    # Sentiment scoring via Ollama
│   ├── score_engagement.py   # Engagement scoring via Ollama/heuristics
│   ├── aggregate_scores.py   # Monthly aggregation of scores
│   └── load_to_snowflake.py  # Loading scores to Snowflake
├── src/                       # Core modules for business logic
│   ├── init.py
│   ├── ollama_client.py      # Client for interacting with Ollama service
│   ├── text_utils.py         # Text preprocessing utilities
│   ├── scoring_logic.py      # Core sentiment/engagement logic
│   └── db_handler.py         # Snowflake interaction (read/write)
├── orchestration/             # Scripts/definitions for workflow orchestration
│   ├── main_workflow.py      # Main script to run the weekly batch
│   └── schedule.sh           # Example cron scheduler script (or Airflow DAG file)
├── prompts/                   # Directory for storing complex prompts
│   ├── sentiment_prompt.txt
│   └── engagement_prompt.txt
├── utils/                     # Utility modules
│   ├── init.py
│   ├── logging_setup.py      # Logging configuration
│   └── helpers.py            # General helper functions
├── requirements.txt           # Project dependencies
└── README.md                  # This file


## Features
- Automated weekly sentiment and engagement scoring from Salesforce email communications.
- Utilizes locally hosted Ollama Large Language Models for data privacy and control.
- Configurable prompts, model selection, and scoring thresholds.
- Modular design for easy maintenance and future enhancements.
- Scores are aggregated monthly per account and loaded into Snowflake.

## Prerequisites
- Access to Salesforce data (via API or data dumps).
- Snowflake account, database, schema, and user credentials with appropriate permissions.
- A Linux/macOS server or VM (or a capable local machine) to host Ollama and run the Python batch jobs.
- Python 3.8+ environment.
- Ollama installed and running.
- Required LLMs downloaded via Ollama (e.g., Llama3, Mistral, Phi-3).

## Ollama Setup
1.  **Install Ollama:** Follow the official installation instructions for your OS from [https://ollama.com](https://ollama.com).
2.  **Download LLMs:**
    ```bash
    ollama pull llama3:8b-instruct
    ollama pull mistral:7b-instruct
    # Add other models as needed
    ```
3.  **Run Ollama Service:** Ensure the Ollama service is running. It typically starts automatically after installation or can be started with:
    ```bash
    ollama serve
    ```
    By default, it serves an API at `http://localhost:11434`.

## Pipeline Configuration
The system uses a central configuration file (`config/settings.py` or a YAML/JSON file loaded by it) for:
- `OllamaConfig`: Ollama service URL, model names for sentiment/engagement, API timeout settings.
- `SnowflakeConfig`: Credentials, account, warehouse, database, schema, table names.
- `PromptConfig`: Paths to prompt files or inline prompt templates.
- `JobConfig`: Parameters for data ingestion (date ranges), aggregation logic, etc.

## Workflow Steps
The weekly batch process, orchestrated by `orchestration/main_workflow.py`, executes the following steps:
1.  **Data Ingestion (`jobs/ingest_sfdc_data.py`)**
    - Fetches new/updated email data from Salesforce for the relevant period.
    - Stores raw data in a staging area (e.g., local files or Snowflake staging table).
2.  **Text Preprocessing (`jobs/preprocess_text.py`)**
    - Cleans email text (HTML removal, signature stripping, etc.).
    - Prepares text for LLM input.
3.  **Sentiment Scoring (`jobs/score_sentiment.py`)**
    - Iterates through preprocessed emails.
    - Sends text to the configured Ollama sentiment model via `src/ollama_client.py`.
    - Parses sentiment labels and scores.
4.  **Engagement Scoring (`jobs/score_engagement.py`)**
    - Calculates engagement metrics from email metadata and/or uses Ollama for qualitative engagement aspects.
    - Generates engagement scores/labels.
5.  **Score Aggregation (`jobs/aggregate_scores.py`)**
    - Aggregates individual email scores to a monthly, account-level sentiment and engagement score.
6.  **Load to Snowflake (`jobs/load_to_snowflake.py`)**
    - Writes the aggregated monthly scores to final Snowflake tables using `src/db_handler.py`.

## Data Organization (Snowflake)
- **Staging Table:** `stg_sfdc_emails` (stores raw or minimally processed email data for the current batch).
- **Sentiment Scores Table:** `fact_account_monthly_sentiment`
    - Columns: `ACCOUNT_ID`, `SCORE_MONTH` (YYYY-MM-01), `SENTIMENT_LABEL`, `SENTIMENT_SCORE_NUMERIC`, `EMAILS_PROCESSED_COUNT`, `LAST_UPDATED_TS`.
- **Engagement Scores Table:** `fact_account_monthly_engagement`
    - Columns: `ACCOUNT_ID`, `SCORE_MONTH` (YYYY-MM-01), `ENGAGEMENT_SCORE_NUMERIC`, `ENGAGEMENT_LEVEL_LABEL`, `KEY_ENGAGEMENT_METRICS_JSON`, `EMAILS_PROCESSED_COUNT`, `LAST_UPDATED_TS`.

## Workflow Execution & Automation
**Setup:**
1.  Clone the repository.
2.  Set up a Python virtual environment and install dependencies:
    ```bash
    python -m venv venv
    source venv/bin/activate  # On Windows: venv\Scripts\activate
    pip install -r requirements.txt
    ```
3.  Configure `config/settings.py` with your environment details (Ollama endpoint, Snowflake credentials, model names).
4.  Ensure Ollama service is running and accessible.

**Manual Execution (for a single run):**
```bash
python orchestration/main_workflow.py --processing_date YYYY-MM-DD