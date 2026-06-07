# 🌊 FloatChat — ARGO Ocean Data Explorer

> An AI-powered conversational interface for exploring and visualizing ARGO ocean float data.

---

## 📖 Overview

**FloatChat**  is a Streamlit-based web application that lets users interact with ARGO oceanographic data through natural language. Instead of writing SQL or wrestling with NetCDF files directly, you simply ask questions like *"What is the temperature profile at 500m depth?"* and FloatChat handles the rest — querying the database, generating insights, and rendering interactive visualizations.

The project combines a vector database (ChromaDB) for semantic search, a relational layer (SQLAlchemy) for structured queries, an LLM processor for natural language understanding, and a rich visualization stack (Plotly + Folium) for charts and maps.

---

## ✨ Features

- 💬 **Conversational chat interface** — ask questions about ocean data in plain English
- 🗺️ **Interactive maps** — visualize float locations via Folium
- 📊 **Dynamic charts** — temperature/salinity vertical profiles and time-series plots via Plotly
- 📂 **NetCDF file upload** — ingest your own ARGO `.nc` / `.nc4` files
- 🧠 **LLM-powered query parsing** — natural language → database queries via LangChain + OpenAI
- 🔍 **Vector search** — semantic retrieval of relevant dataset summaries using ChromaDB
- 📋 **Sample data mode** — built-in sample ARGO dataset to get started immediately
- 🔄 **Alternate UI** — a secondary interface (`app_test.py`) with a different layout

---

## 🗂️ Project Structure

```
Flowchat/
├── app.py              # Main Streamlit application (primary UI)
├── app_test.py         # Alternate UI (experimental layout)
├── requirements.txt    # Python dependencies
├── data/               # Data directory (uploads go here)
├── chroma/             # ChromaDB vector store persistence
└── src/
    ├── data_ingestion.py   # ARGO NetCDF ingestion & sample data generation
    ├── database.py         # SQLAlchemy DB manager + ChromaDB integration
    └── llm_processor.py    # LangChain-based natural language query processor
```

---

## 🚀 Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/ramkrishnatiwari1108-cat-to-1108/Flowchat.git
cd Flowchat
```

### 2. Create and activate a virtual environment

```bash
python -m venv venv

# On Windows
venv\Scripts\activate

# On macOS/Linux
source venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

> If you run into issues, refer to the individual package docs or use an AI assistant to troubleshoot version conflicts.

### 4. Set up environment variables

Create a `.env` file in the project root and add your OpenAI API key:

```env
OPENAI_API_KEY=your_openai_api_key_here
```

### 5. Run the app

**Primary UI (recommended):**
```bash
streamlit run app.py
```

**Alternate UI:**
```bash
streamlit run app_test.py
```

---

## 🧭 Usage

1. Open the app in your browser (Streamlit will provide a local URL, typically `http://localhost:8501`).
2. Click **"Load Sample Data"** in the sidebar to populate the database with built-in ARGO data.
3. Type a question in the chat input at the bottom, such as:
   - *"Show temperature profiles for all platforms"*
   - *"What is the salinity at 200m depth?"*
   - *"Plot temperature over time"*
4. Optionally, **upload a NetCDF file** (`.nc` / `.nc4`) via the sidebar to explore your own data.

---

## 🧠 RAG Architecture — How It Works

FloatChat uses a **Retrieval-Augmented Generation (RAG)** pipeline to answer questions about ocean data. Rather than asking an LLM to answer from memory (which it can't — it has no knowledge of your specific ARGO floats), the app first *retrieves* the relevant data from local storage, then *augments* a prompt with that data before asking the LLM to generate a response. This means answers are always grounded in your actual dataset.

Here is how a single user question flows through the full pipeline:

```
User question
     │
     ▼
┌─────────────────────────────────┐
│   1. SEMANTIC SEARCH (ChromaDB) │  ← "What context is relevant?"
│   Query → vector embedding      │
│   → similarity search           │
│   → top-K matching summaries    │
└────────────────┬────────────────┘
                 │  retrieved context chunks
                 ▼
┌─────────────────────────────────┐
│   2. STRUCTURED QUERY (SQLite)  │  ← "Fetch the actual data"
│   LangChain parses intent       │
│   → generates SQL query         │
│   → SQLAlchemy executes it      │
│   → returns rows / profiles     │
└────────────────┬────────────────┘
                 │  query results
                 ▼
┌─────────────────────────────────┐
│   3. LLM RESPONSE (OpenAI)      │  ← "Compose a human answer"
│   Prompt = question             │
│             + retrieved context │
│             + query results     │
│   → GPT generates final answer  │
└────────────────┬────────────────┘
                 │  answer + chart instructions
                 ▼
┌─────────────────────────────────┐
│   4. VISUALIZATION (Plotly /    │  ← "Show it visually"
│      Folium)                    │
│   Renders charts, maps, tables  │
└─────────────────────────────────┘
```

### Stage 1 — Vector Indexing & Retrieval (`src/database.py` + `chroma/`)

When ARGO data is ingested (either from an uploaded NetCDF file or the built-in sample), `database.py` generates **text summaries** of each float observation — things like platform ID, date, location, depth range, and key measurements. These summaries are converted into **numerical vector embeddings** using OpenAI's embedding model and stored in **ChromaDB**, a local persistent vector database (saved to the `chroma/` folder on disk).

When a user submits a question, the same embedding model converts it into a vector, and ChromaDB performs a **cosine similarity search** to find the most semantically relevant observation summaries. These top-K results become the *retrieved context* — the factual grounding that gets passed to the LLM.

This step solves the core problem of large datasets: you don't dump thousands of rows into an LLM prompt. You retrieve only what's relevant to the specific question.

### Stage 2 — Structured Data Query (`src/database.py` + SQLAlchemy)

Alongside the vector store, all ingested ARGO data is also stored in a **SQLite relational database** managed via SQLAlchemy. This stores structured records like measurement tables, platform metadata, depth profiles, and timestamps.

The LLM processor (`src/llm_processor.py`) uses LangChain to parse the user's question and decide whether a structured SQL query is needed (e.g. "What is the salinity at 300m for platform 1234?"). If so, LangChain generates the SQL, SQLAlchemy executes it, and the result rows are fetched as a DataFrame. This gives the answer *numerical precision* from the real data, not a hallucinated approximation.

### Stage 3 — Prompt Construction & LLM Response (`src/llm_processor.py`)

`llm_processor.py` is the orchestration layer. It:

1. Receives the user's raw question.
2. Calls ChromaDB to retrieve semantically relevant context chunks.
3. Optionally executes a structured SQL query to fetch precise data rows.
4. Builds a **structured prompt** that combines: the user question, the retrieved context, and any SQL results.
5. Sends the assembled prompt to OpenAI's chat completion API via LangChain.
6. Returns both a natural language answer and structured metadata (e.g. chart type, axes, data to plot).

This two-store retrieval strategy — **semantic search for context + SQL for precision** — is what makes the chatbot reliable. Vague questions ("tell me about depth profiles") hit the vector store; specific analytical questions ("show me temperature at 500m between January and March") hit the SQL layer.

### Stage 4 — Visualization (`app.py`)

Once the LLM response returns, `app.py` inspects the structured output to decide what to render:

- **Vertical profiles** (temperature or salinity vs. depth) → Plotly line charts
- **Time-series** (measurement over time) → Plotly scatter/line plots  
- **Geographic float positions** → Folium interactive map (rendered via `streamlit-folium`)
- **Tabular data** → Streamlit `st.dataframe`

All visualizations are rendered inline inside the Streamlit chat interface, alongside the LLM's text explanation.

---

## 🛠️ Tools & Libraries — In Depth

### 🖥️ Streamlit
The entire UI is built in Streamlit. The main `app.py` uses `st.chat_message` and `st.chat_input` to create a persistent conversational thread. The sidebar hosts data management controls (file upload, sample data loader, database status). Columns split the layout between chat and visualizations. `app_test.py` is an alternate experimental layout (Claude-generated, per the repo notes).

### 🗄️ ChromaDB (`chromadb==0.4.18`)
ChromaDB is the local vector database. It persists embeddings to the `chroma/` directory so the index survives app restarts. It uses OpenAI's embedding model to encode observation summaries, and exposes a `.query()` method for top-K cosine similarity retrieval. Collections in ChromaDB map to logical groups of ARGO observations (e.g. by platform or region).

### 🔗 LangChain (`langchain>=0.2.0`, `langchain-community>=0.2.0`)
LangChain provides the glue between the LLM, the vector store, and the SQL database. Key components used:
- **`ChatOpenAI`** — wraps the OpenAI chat API with automatic retry and streaming support
- **`ConversationBufferMemory`** — maintains the full message history for multi-turn chat
- **Prompt templates** — structured templates that inject retrieved context and SQL results into each LLM call
- **Output parsers** — parse structured JSON from the LLM response to extract chart metadata vs. plain text

### 🤖 OpenAI (`openai>=1.6.1`)
Two OpenAI capabilities are used:
- **`text-embedding-ada-002`** (or similar) — converts both data summaries and user questions into vector embeddings for ChromaDB.
- **`gpt-3.5-turbo` / `gpt-4`** — the generation model that reads the assembled prompt (question + context + SQL results) and produces the final answer.

### 🗃️ SQLAlchemy (`sqlalchemy==2.0.23`)
SQLAlchemy manages the relational layer on top of a local SQLite file. `database.py` defines table schemas for ARGO observations (platform ID, latitude, longitude, timestamp, pressure, temperature, salinity). It handles session management, connection pooling, and exposes query helpers that return Pandas DataFrames directly — making it easy to pass data to Plotly.

### 🌊 xarray + NetCDF4
ARGO float data is distributed as NetCDF (`.nc`) files — a self-describing binary format standard in oceanography. `xarray` provides a Pandas-like API for reading multidimensional NetCDF datasets. `data_ingestion.py` opens uploaded `.nc`/`.nc4` files with xarray, extracts variable arrays (PRES, TEMP, PSAL, JULD, LATITUDE, LONGITUDE, PLATFORM_NUMBER), and writes them into both the SQLite DB and the ChromaDB index.

### 📊 Plotly (`plotly==5.18.0`)
Plotly renders all in-app charts as interactive HTML figures:
- `px.line` / `go.Scatter` for depth profile plots (pressure on Y-axis, temperature/salinity on X)
- `px.scatter` / `px.line` for time-series
- `go.Figure` with custom layout for multi-platform overlays

### 🗺️ Folium + streamlit-folium (`folium==0.15.0`, `streamlit-folium==0.15.1`)
Geographic float positions are rendered on a Leaflet.js map via Folium. Each float's last known location is shown as a marker with a popup containing platform metadata. `streamlit-folium` injects the Folium map directly into the Streamlit component tree via `st_folium()`.

### 🔢 Pandas, NumPy, SciPy
Standard scientific Python stack. Pandas DataFrames are the primary in-memory data format moving between SQL queries, LLM prompts, and Plotly. NumPy handles array operations on pressure/temperature/salinity fields. SciPy is used for any interpolation needed to regularise depth profiles to standard pressure levels.

### 🌍 python-dotenv
Loads the `.env` file so `OPENAI_API_KEY` is available as an environment variable without hardcoding secrets in source files.

---

## 📦 Dependencies

```
streamlit==1.29.0
pandas>=2.1.3
numpy>=1.26.0
xarray==2023.10.1
netCDF4==1.6.5
plotly==5.18.0
sqlalchemy==2.0.23
chromadb==0.4.18
langchain>=0.2.0
langchain-community>=0.2.0
openai>=1.6.1
python-dotenv>=1.0.0
folium==0.15.0
streamlit-folium==0.15.1
scipy>=1.11.4
```

---

## 🔬 About ARGO Data

[ARGO](https://argo.ucsd.edu/) is a global array of over 4,000 free-drifting profiling floats that measure temperature, salinity, and other ocean properties from the surface down to 2,000 metres. FloatChat makes this scientific dataset accessible through a simple chat interface.

---

## 🤝 Contributing

Contributions, issues, and feature requests are welcome. Feel free to open a pull request or file an issue.

---

## 📄 License

This project does not currently specify a license. Please contact the repository owner for usage permissions.

---

*Built with 🌊 and Python.*
