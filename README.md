# 🎵 FeelTune
### AI-Powered Semantic Music Recommendation Platform

> *Recommend songs based on emotions, situations, and natural language — not just genres or listening history.*

[![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-green.svg)](https://fastapi.tiangolo.com)
[![React](https://img.shields.io/badge/React-18+-61DAFB.svg)](https://reactjs.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-336791.svg)](https://postgresql.org)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 📌 Table of Contents

- [Project Overview](#-project-overview)
- [The Problem We Solve](#-the-problem-we-solve)
- [System Architecture](#-system-architecture)
- [Component Architecture](#-component-architecture)
- [Data Architecture](#-data-architecture)
- [AI Pipeline](#-ai-pipeline)
- [Database Schema](#-database-schema)
- [API Documentation](#-api-documentation)
- [Project Structure](#-project-structure)
- [Development Roadmap](#-development-roadmap)
- [Getting Started](#-getting-started)
- [Tech Stack](#-tech-stack)

---

## 🎯 Project Overview

**FeelTune** is a natural language music recommendation system that understands how you *feel*, not just what you've listened to before. It combines NLP, Sentence Transformers, Semantic Vector Search, and Audio Feature Matching into a hybrid recommendation engine.

**Example Interaction:**

```
User: "I'm feeling lonely after my exams. I want something peaceful but motivating."

FeelTune understands →
  Emotion   : Lonely
  Mood      : Calm
  Energy    : Low
  Intent    : Healing

Returns →
  🎵 Fix You — Coldplay
  🎵 Phir Le Aya Dil — Arijit Singh
  🎵 Kho Gaye Hum Kahan — Prateek Kuhad
  🎵 The Night We Met — Lord Huron
```

This is significantly different from Spotify's "users like you also listened to" approach. FeelTune asks: **What is the user actually feeling right now?**

---

## 🧩 The Problem We Solve

| Traditional Recommendation | FeelTune |
|---|---|
| Based on listening history | Based on current emotion |
| Requires large user data | Works for new users instantly |
| Classifies one mood per query | Understands multiple emotions in a sentence |
| Genre-based filtering | Semantic understanding of lyrics + mood |
| Cold start problem | No cold start — just describe how you feel |

---

## 🏗 System Architecture

```
                          USER
                            │
                            ▼
                   ┌─────────────────┐
                   │  React Frontend │
                   │  Chat + Search  │
                   └────────┬────────┘
                            │ REST / WebSocket
                            ▼
                   ┌─────────────────┐
                   │  FastAPI Backend │
                   └────────┬────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │Recommendation│  │ Query Parser │  │ User Service │
  │   Service    │  │              │  │              │
  └──────┬───────┘  └──────┬───────┘  └──────────────┘
         │                 │
         │                 ▼
         │        ┌──────────────────┐
         │        │ NLP Intent +     │
         │        │ Emotion Analysis │
         │        └──────────────────┘
         │
         ▼
  ┌──────────────────────────┐
  │  Embedding Generator     │
  │  (Sentence Transformer)  │
  └──────────┬───────────────┘
             │
             ▼
  ┌──────────────────────────┐
  │  Vector Search (FAISS /  │
  │  pgvector)               │
  └──────────┬───────────────┘
             │
             ▼
  ┌──────────────────────────┐
  │  Candidate Songs         │
  └──────────┬───────────────┘
             │
             ▼
  ┌──────────────────────────┐
  │  Hybrid Ranking Engine   │
  │  Semantic + Audio +      │
  │  Popularity + Diversity  │
  └──────────┬───────────────┘
             │
             ▼
        Top-K Songs
             │
             ▼
        Frontend
```

---

## 🧱 Component Architecture

FeelTune is organized into 6 layers:

```
FeelTune
│
├── 1. Data Layer        → Data engineering pipeline
├── 2. Database Layer    → PostgreSQL + pgvector
├── 3. AI Layer          → NLP, embeddings, search, ranking
├── 4. Backend Layer     → FastAPI REST APIs
├── 5. Frontend Layer    → React + TypeScript + Tailwind
└── 6. Deployment Layer  → Docker + CI/CD
```

### 1. Data Layer

Responsible for raw data ingestion, cleaning, and building the master dataset.

```
Raw CSVs (Hindi, Punjabi, Haryanvi, Old)
        │
        ▼
   CSV Loader
        │
        ▼
   Data Cleaner
   ├── Type Normalization
   ├── Duplicate Detection (song_name + singer)
   ├── Missing Value Handler
   └── Recency Scoring
        │
        ▼
  Feature Engineer
  ├── Song Description Builder
  ├── Mood Tag Generator
  └── Recency Score
        │
        ▼
  master_songs.csv
```

**Key decisions:**
- Duplicates resolved by keeping highest popularity version
- Old Bollywood capped at 1000 songs to prevent era bias
- `recency_score = 1 / (1 + log(current_year - released_year))` applied so recent songs rank naturally higher

---

### 2. Database Layer

**Technology:** PostgreSQL 15 + pgvector extension

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│    songs     │   │    lyrics    │   │   artists    │
│──────────────│   │──────────────│   │──────────────│
│ id           │   │ id           │   │ id           │
│ song_name    │   │ song_id (FK) │   │ name         │
│ artist_id    │   │ original     │   │ country      │
│ language     │   │ translated   │   │ image_url    │
│ danceability │   │ source       │   │ spotify_id   │
│ energy       │   │ embed_status │   └──────────────┘
│ valence      │   └──────────────┘
│ tempo        │   ┌──────────────┐   ┌──────────────┐
│ popularity   │   │    users     │   │ playlists    │
│ released_date│   │──────────────│   │──────────────│
│ recency_score│   │ id           │   │ id           │
│ embedding    │   │ email        │   │ user_id (FK) │
└──────────────┘   │ preferences  │   │ song_ids     │
                   │ fav_language │   └──────────────┘
                   │ history      │
                   └──────────────┘
```

---

### 3. AI Layer

The core intelligence of FeelTune. Contains 5 modules:

#### Module 1 — Intent Classifier
```
Input : "Need breakup songs"
Output: Intent = BREAKUP
```

#### Module 2 — Emotion Detector
```
Input : "Feeling lonely and anxious"
Output: [Loneliness: 0.82, Anxiety: 0.71, Sadness: 0.60]
        (Multi-label — not just one emotion)
```

#### Module 3 — Embedding Generator
```
Model   : all-MiniLM-L6-v2  (default)
          BAAI/bge-small-en  (alternative)
Input   : Song lyrics + description + mood tags
Output  : 384-dimensional float vector
Storage : pgvector column in songs table
          + FAISS index file for fast retrieval
```

#### Module 4 — Vector Search
```
User query → Sentence Transformer → Query vector (384-d)
                    │
                    ▼
          Cosine Similarity Search
          (FAISS ANN or pgvector)
                    │
                    ▼
          Top 50 Candidate Songs
```

#### Module 5 — Hybrid Ranking Engine
```
Final Score = (0.60 × Semantic Similarity)
            + (0.15 × Mood Match)
            + (0.10 × Audio Feature Match)
            + (0.10 × Popularity Score)
            + (0.05 × Diversity Penalty)

Output → Top 10 songs
```

**Audio Feature Matching Logic:**

| User Mood | Energy | Valence | Tempo | Acousticness |
|-----------|--------|---------|-------|--------------|
| Happy     | High   | High    | Fast  | Low          |
| Sad       | Low    | Low     | Slow  | High         |
| Motivated | High   | Medium  | Fast  | Low          |
| Calm      | Low    | Medium  | Slow  | High         |
| Angry     | High   | Low     | Fast  | Low          |
| Romantic  | Low    | High    | Slow  | Medium       |

---

### 4. Backend Layer

**Technology:** FastAPI + Python 3.10+

```
backend/
├── app.py                        ← FastAPI app entry point
├── config.py                     ← Environment config
├── dependencies.py               ← DB session, auth dependencies
│
├── api/                          ← Route handlers
│   ├── recommend.py
│   ├── songs.py
│   ├── artists.py
│   ├── users.py
│   └── auth.py
│
├── services/                     ← Business logic
│   ├── recommendation_service.py ← Main orchestrator
│   ├── embedding_service.py      ← Sentence Transformer wrapper
│   ├── lyrics_service.py         ← Lyrics fetch + store
│   ├── search_service.py         ← FAISS / pgvector search
│   └── ranking_service.py        ← Hybrid ranker
│
└── repositories/                 ← DB access layer
    ├── song_repository.py
    ├── lyrics_repository.py
    └── user_repository.py
```

---

### 5. Frontend Layer

**Technology:** React 18 + TypeScript + TailwindCSS

```
Pages:
├── /           → Home (mood input chat interface)
├── /search     → Search by song/artist
├── /recommend  → Recommendation results
├── /song/:id   → Song detail page
├── /artist/:id → Artist page
├── /profile    → User preferences
└── /history    → Past recommendations
```

---

## 📊 Data Architecture

```
Raw Datasets
├── hindi.csv      (1,184 songs)
├── punjabi.csv    (3,818 songs)
├── haryanvi.csv   (228 songs)
└── old.csv        (2,448 songs)
          │
          ▼
   Cleaning Pipeline
   ├── Type standardization
   ├── Duplicate removal (by song_name + singer)
   ├── Recency score calculation
   └── Era balancing (old Bollywood capped at 1000)
          │
          ▼
   master_songs.csv (~6,000 songs)
          │
          ▼
   Lyrics Collection
   ├── API integration
   └── Web scraping (fallback)
          │
          ▼
   songs_with_lyrics.csv
          │
          ▼
   Sentence Transformer
          │
          ▼
   embeddings.parquet
          │
          ▼
   PostgreSQL + pgvector / FAISS Index
```

**Dataset columns:**

| Column | Type | Description |
|--------|------|-------------|
| song_name | str | Lowercase normalized song title |
| singer | str | Artist name |
| language | str | hindi / punjabi / haryanvi |
| released_date | datetime | Release date |
| danceability | float | 0.0 – 1.0 |
| acousticness | float | 0.0 – 1.0 |
| energy | float | 0.0 – 1.0 |
| valence | float | 0.0 – 1.0 (positivity) |
| tempo | float | BPM |
| loudness | float | dB |
| popularity | Int64 | 0 – 100 |
| recency_score | float | Computed: 1 / (1 + log(age)) |

---

## 🤖 AI Pipeline

```
User Query (natural language)
        │
        ▼
Language Detection
(langdetect)
        │
        ▼
Intent Detection
(zero-shot or fine-tuned classifier)
        │
        ▼
Emotion Detection
(multi-label — e.g. lonely + anxious + hopeful)
        │
        ▼
Query Embedding Generation
(Sentence Transformer → 384-d vector)
        │
        ├──────────────────► Semantic Search (FAISS)
        │                           │
        ├──────────────────► Audio Feature Filter
        │                     (energy, valence, tempo)
        │
        ├──────────────────► Language Filter (optional)
        │
        └──────────────────► Recency Boost
                                    │
                                    ▼
                           Hybrid Ranker
                                    │
                                    ▼
                            Final Top-10 Songs
```

### Embedding Strategy

Each song is embedded from a **combined text description**, not just lyrics. Format:

```
"{song_name} by {singer}. Language: {language}. 
Mood: {mood_tags}. Energy: {energy_label}. 
Lyrics theme: {lyrics_summary}. 
Audio feel: {tempo_label}, {acousticness_label}."
```

Example:
```
"Tere Bina by Arijit Singh. Language: Hindi.
Mood: romantic, melancholic, longing. Energy: low.
Lyrics theme: missing someone deeply, soulful love.
Audio feel: slow tempo, highly acoustic."
```

This ensures semantic search matches *meaning and feeling*, not just genre keywords.

---

## 🗄 Database Schema

```sql
-- Enable vector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Songs table
CREATE TABLE songs (
    id              SERIAL PRIMARY KEY,
    song_name       TEXT NOT NULL,
    artist_id       INTEGER REFERENCES artists(id),
    language        TEXT,
    danceability    FLOAT,
    acousticness    FLOAT,
    energy          FLOAT,
    liveness        FLOAT,
    loudness        FLOAT,
    speechiness     FLOAT,
    tempo           FLOAT,
    valence         FLOAT,
    mode            INTEGER,
    key             INTEGER,
    time_signature  INTEGER,
    popularity      INTEGER,
    stream          BIGINT,
    released_date   DATE,
    recency_score   FLOAT,
    embedding       vector(384),     -- pgvector column
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Artists table
CREATE TABLE artists (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    country     TEXT,
    image_url   TEXT,
    spotify_id  TEXT
);

-- Lyrics table
CREATE TABLE lyrics (
    id              SERIAL PRIMARY KEY,
    song_id         INTEGER REFERENCES songs(id),
    original_text   TEXT,
    translated_text TEXT,
    language        TEXT,
    source          TEXT,
    embed_status    BOOLEAN DEFAULT FALSE
);

-- Users table
CREATE TABLE users (
    id              SERIAL PRIMARY KEY,
    email           TEXT UNIQUE NOT NULL,
    password_hash   TEXT,
    fav_language    TEXT,
    preferences     JSONB,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Query history
CREATE TABLE queries (
    id              SERIAL PRIMARY KEY,
    user_id         INTEGER REFERENCES users(id),
    query_text      TEXT,
    detected_emotion TEXT,
    detected_intent TEXT,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Recommendations log
CREATE TABLE recommendations (
    id          SERIAL PRIMARY KEY,
    query_id    INTEGER REFERENCES queries(id),
    song_id     INTEGER REFERENCES songs(id),
    final_score FLOAT,
    rank        INTEGER
);

-- Vector index for fast search
CREATE INDEX ON songs USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

---

## 📡 API Documentation

### Base URL
```
http://localhost:8000/api/v1
```

### Authentication
All protected routes require a Bearer token in the Authorization header:
```
Authorization: Bearer <token>
```

---

### POST `/recommend`
Get song recommendations from a natural language query.

**Request:**
```json
{
  "query": "I'm stressed about my placement exams, need something calming but motivating",
  "language_filter": ["hindi", "punjabi"],
  "top_k": 10,
  "recency_boost": true
}
```

**Response:**
```json
{
  "query_id": "q_abc123",
  "detected": {
    "emotions": ["stress", "anxiety", "hope"],
    "intent": "MOTIVATIONAL",
    "energy_preference": "low-medium"
  },
  "recommendations": [
    {
      "rank": 1,
      "song_id": 4821,
      "song_name": "kun faya kun",
      "singer": "a.r. rahman",
      "language": "hindi",
      "released_date": "2011-04-14",
      "popularity": 87,
      "scores": {
        "semantic_score": 0.89,
        "mood_score": 0.81,
        "audio_score": 0.74,
        "final_score": 0.85
      }
    }
  ]
}
```

---

### POST `/search`
Search songs by name, artist, or keyword.

**Request:**
```json
{
  "query": "arijit singh sad songs",
  "language": "hindi",
  "limit": 20
}
```

**Response:**
```json
{
  "results": [
    {
      "song_id": 102,
      "song_name": "tum hi ho",
      "singer": "arijit singh",
      "language": "hindi",
      "popularity": 94
    }
  ],
  "total": 1
}
```

---

### GET `/song/{id}`
Get full details of a specific song.

**Response:**
```json
{
  "id": 102,
  "song_name": "tum hi ho",
  "singer": "arijit singh",
  "language": "hindi",
  "released_date": "2013-01-11",
  "audio_features": {
    "energy": 0.35,
    "valence": 0.21,
    "danceability": 0.42,
    "tempo": 72.4,
    "acousticness": 0.81
  },
  "popularity": 94,
  "recency_score": 0.31
}
```

---

### POST `/login`
Authenticate and receive a JWT token.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "yourpassword"
}
```

**Response:**
```json
{
  "access_token": "eyJhbGci...",
  "token_type": "bearer"
}
```

---

### POST `/register`
Create a new user account.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "yourpassword",
  "fav_language": "hindi"
}
```

---

### GET `/history` *(Protected)*
Get past recommendation queries for the logged-in user.

**Response:**
```json
{
  "queries": [
    {
      "query_id": "q_abc123",
      "query_text": "I'm feeling lonely after my exams",
      "detected_emotion": "lonely",
      "created_at": "2026-07-10T14:32:00Z"
    }
  ]
}
```

---

## 📁 Project Structure

```
FeelTune/
│
├── README.md
├── requirements.txt
├── .gitignore
├── .env.example
│
├── data/
│   ├── raw/
│   │   ├── hindi.csv
│   │   ├── punjabi.csv
│   │   ├── haryanvi.csv
│   │   ├── old.csv
│   │   └── lyrics/
│   ├── interim/
│   │   ├── cleaned_hindi.csv
│   │   ├── cleaned_punjabi.csv
│   │   ├── cleaned_haryanvi.csv
│   │   └── cleaned_old.csv
│   └── processed/
│       ├── master_songs.csv
│       ├── songs_with_lyrics.csv
│       └── embeddings.parquet
│
├── notebooks/
│   ├── 01_data_cleaning.ipynb
│   ├── 02_feature_engineering.ipynb
│   ├── 03_database_import.ipynb
│   ├── 04_lyrics_collection.ipynb
│   ├── 05_embeddings.ipynb
│   ├── 06_vector_search.ipynb
│   ├── 07_recommendation.ipynb
│   └── 08_evaluation.ipynb
│
├── database/
│   ├── schema.sql
│   ├── seed.py
│   ├── db.py
│   ├── models.py
│   ├── crud.py
│   └── migrations/
│
├── backend/
│   ├── app.py
│   ├── config.py
│   ├── dependencies.py
│   ├── api/
│   │   ├── recommend.py
│   │   ├── songs.py
│   │   ├── artists.py
│   │   ├── users.py
│   │   └── auth.py
│   ├── services/
│   │   ├── recommendation_service.py
│   │   ├── embedding_service.py
│   │   ├── lyrics_service.py
│   │   ├── search_service.py
│   │   └── ranking_service.py
│   └── repositories/
│       ├── song_repository.py
│       ├── lyrics_repository.py
│       └── user_repository.py
│
├── ml/
│   ├── embedding/
│   │   ├── generate_embeddings.py
│   │   └── embedding_model.py
│   ├── retrieval/
│   │   ├── faiss_index.py
│   │   ├── semantic_search.py
│   │   └── retriever.py
│   ├── ranking/
│   │   ├── hybrid_ranker.py
│   │   └── scorer.py
│   └── evaluation/
│
├── frontend/
│   ├── src/
│   ├── public/
│   └── package.json
│
├── scripts/
│   ├── import_master_dataset.py
│   ├── fetch_lyrics.py
│   ├── update_embeddings.py
│   └── rebuild_index.py
│
├── tests/
│
└── docs/
    ├── architecture.md
    ├── api.md
    ├── database.md
    └── workflow.md
```

---

## 🗺 Development Roadmap

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 1 | ✅ Done | Dataset collection, merge, clean, master CSV |
| Phase 2 | ✅ Done | PostgreSQL setup, schema, import pipeline |
| Phase 3 | 🔄 Next | Lyrics collection pipeline, API integration |
| Phase 4 | ⏳ | Embedding generation, pgvector storage, FAISS index |
| Phase 5 | ⏳ | Semantic search, hybrid ranking, recommendation engine |
| Phase 6 | ⏳ | FastAPI backend, REST APIs, JWT authentication |
| Phase 7 | ⏳ | React frontend, chat interface, recommendation UI |
| Phase 8 | ⏳ | Evaluation, testing, Docker, deployment, monitoring |

---

## 🚀 Getting Started

### Prerequisites
- Python 3.10+
- PostgreSQL 15+ with pgvector extension
- Node.js 18+ (for frontend)

### Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/FeelTune.git
cd FeelTune

# Create virtual environment
python -m venv .venv
.venv\Scripts\activate        # Windows
source .venv/bin/activate     # Mac/Linux

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env
# Edit .env with your PostgreSQL credentials

# Set up database
psql -U postgres -f database/schema.sql

# Import master dataset
python scripts/import_master_dataset.py

# Start the backend
uvicorn backend.app:app --reload

# Start the frontend (new terminal)
cd frontend
npm install
npm run dev
```

---

## 🛠 Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Python 3.10+, TypeScript |
| ML / NLP | Sentence Transformers, HuggingFace |
| Embedding Model | all-MiniLM-L6-v2 |
| Vector Search | FAISS + pgvector |
| Backend | FastAPI |
| Database | PostgreSQL 15 |
| Frontend | React 18 + TailwindCSS |
| Auth | JWT (python-jose) |
| Containerization | Docker + Docker Compose |
| Data Processing | Pandas, NumPy, Scikit-learn |

---

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

<div align="center">
  Built with ❤️ for people who feel music, not just play it.
</div>
