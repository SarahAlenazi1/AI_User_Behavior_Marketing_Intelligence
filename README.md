# 🤖 AI User Behavior & Marketing Intelligence Assistant

**Final Project — Modern Data Engineering for AI Systems**

## 1. Project Overview

A data pipeline that loads AI-assistant usage data with **Spark**, checks it with an automated **data quality gate**, stores the validated data in **Delta Lake**, transforms it into clean/trusted data, and uses a **RAG assistant** (embeddings + ChromaDB + LLM) to turn user behavior into marketing insights.

```text
Understanding AI Users → Identifying Target Audiences → Discovering Marketing Opportunities → Generating Marketing Insights
```

Main file: [`AI_User_Behavior_Marketing_Intelligence.ipynb`](AI_User_Behavior_Marketing_Intelligence.ipynb)

## 2. Problem Description

**What problem are we solving?** Marketing teams of AI products do not know *who* uses AI assistants, *for what*, *on which device and model*, *for how long* and *how satisfied* they are — so targeting and messaging are guesswork.

**What data do we need?** Session logs of AI-assistant usage: time, device, usage category, prompt length, session length, satisfaction rating, AI model and tokens used.

**What is the expected output?** A trusted Delta Lake table, segment profiles, and an AI assistant that answers questions such as:

- What are the most common ways people use AI assistants?
- What are the characteristics of users who frequently use AI for coding?
- How do Coding users differ from Education or Writing users?
- What type of AI users could be relevant for a specific AI product?
- What user behaviors indicate high engagement with AI?
- What marketing opportunities can be identified from AI usage patterns?
- What type of marketing message could be relevant to a specific user segment?

## 3. Data Source

**Daily AI Assistant Usage Behavior Dataset** (public, synthetic) — [`Daily_AI_Assistant_Usage_Behavior_Dataset.csv`](Daily_AI_Assistant_Usage_Behavior_Dataset.csv) — 300 sessions, January–March 2025.

| Column | Values |
|---|---|
| `timestamp` | `yyyy-MM-dd HH:mm:ss` |
| `device` | Desktop, Mobile, Tablet, Smart Speaker |
| `usage_category` | Coding, Education, Writing, Research, Productivity, Entertainment, Daily Tasks |
| `prompt_length` | 5 – 250 |
| `session_length_minutes` | 0.2 – 15 |
| `satisfaction_rating` | 1 – 5 |
| `assistant_model` | GPT-4o, GPT-5, GPT-5.1, Mini, o1 |
| `tokens_used` | 32 – 1 500 |

**Ingestion (batch):** two CSV batches land in `data/landing_zone/` and are loaded with Spark using an explicit schema:
- **Batch 1** — the historical dataset (300 rows).
- **Batch 2** — a new batch of 8 sessions containing data errors (missing device, negative session length, rating 9, unknown device, wrong timestamp format, duplicate session).

## 4. Workflow / Architecture

![Pipeline architecture](docs/architecture.png)

```text
Choose a Problem
      ↓
Get a Dataset
      ↓
Load Data with Spark
      ↓
Data Quality Checks
      ↓
Quality Gate
   ↙       ↘
 FAIL      PASS
  ↓          ↓
Rejected    Delta Lake
               ↓
        Clean/Trusted Data
               ↓
         AI / RAG + Analytics
```

| Step | What happens |
|---|---|
| Load Data with Spark | CSV → Spark DataFrame (explicit schema) + `session_id`, `batch_id`, `ingested_at` |
| Data Quality Checks | 6 checks over Completeness, Accuracy, Uniqueness, Validity → quality report |
| Quality Gate | `passed_all_gates = True` → Delta Lake · `False` → batch rejected (not stored) |
| Delta Lake | `data/delta/validated_sessions` |
| Clean/Trusted Data | transformation → `data/delta/trusted_sessions`, then segment profiles in `data/delta/segment_profiles/` |
| AI / RAG + Analytics | charts + RAG assistant answering the marketing questions |

**Transformation (validated → trusted):** parsed timestamp, `time_of_day`, `session_type` (Quick / Standard / Deep), `satisfaction_level` (Low / Neutral / High), `engagement_score` (0–100 = 40 % session length + 30 % tokens + 30 % prompt length), `engagement_level` (High / Medium / Low), `audience_segment` (e.g. Coding → Developers).

## 5. Data Quality Checks

| Dimension | Check | Rule |
|---|---|---|
| Completeness | `completeness_required_fields` | no required field is missing |
| Accuracy | `accuracy_positive_values` | prompt length, session length and tokens > 0 |
| Accuracy | `accuracy_satisfaction_range` | satisfaction rating between 1 and 5 |
| Uniqueness | `uniqueness_session_id` | each session appears once |
| Validity | `validity_allowed_values` | device, category and model are known values |
| Validity | `validity_timestamp_format` | timestamp is `yyyy-MM-dd HH:mm:ss` |

<!-- RESULTS_QUALITY -->

## 6. AI / RAG and Analytics Output

- **Analytics:** sessions, average satisfaction and high-engagement share per usage category (`outputs/usage_by_category.png`).
- **RAG assistant:**
  1. Segment profiles → one text document per segment (category, device, model, time of day, engagement level).
  2. Embeddings with `sentence-transformers/all-MiniLM-L6-v2`.
  3. Stored in **ChromaDB** (vector database).
  4. The question is embedded and the most relevant documents are retrieved.
  5. Question + retrieved context → **LLM (OpenRouter)**, instructed to answer **only** from the context.
- Answers are saved to `outputs/marketing_insights.md`.

## 7. Results

<!-- RESULTS_MAIN -->

## 8. Technologies Used

| Purpose | Technology |
|---|---|
| Processing | Apache Spark (PySpark 3.5) |
| Storage | Delta Lake (`delta-spark` 3.2) |
| Data quality & logging | Spark quality engine, Loguru |
| Embeddings | Sentence-Transformers `all-MiniLM-L6-v2` |
| Vector database | ChromaDB |
| LLM | OpenRouter (OpenAI-compatible API) |
| Analytics | pandas, matplotlib |
| Environment | Python 3.10+, Java 17, Jupyter / Google Colab |

## 9. How to Run the Project

### Google Colab (recommended)
1. Open [colab.research.google.com](https://colab.research.google.com) → **File → Upload notebook** → choose `AI_User_Behavior_Marketing_Intelligence.ipynb` (or **File → Open notebook → GitHub** and paste this repository URL).
2. *(Optional)* Click the 🔑 **Secrets** icon in the left sidebar → add `OPENROUTER_API_KEY` with your key → enable notebook access.
3. **Runtime → Run all**.
   - When the upload button appears, choose `Daily_AI_Assistant_Usage_Behavior_Dataset.csv`.
   - If you did not add the secret, paste your OpenRouter key into the hidden prompt.

### Local
Requires Python 3.10+ and Java 17.

```bash
git clone <repository-url>
cd <repository-folder>
pip install -r requirements.txt
cp .env.example .env        # add your OPENROUTER_API_KEY
jupyter notebook AI_User_Behavior_Marketing_Intelligence.ipynb
```

Run all cells from top to bottom.

## 10. Future Improvements

- Stream new sessions in real time (Kafka + Spark Structured Streaming) through the same quality gate.
- Add a user ID to build user-level personas and retention metrics.
- Use clustering (Spark MLlib) for data-driven segmentation on a larger dataset.
- Add a chat interface (Streamlit) for the marketing team.

## 11. SDAIA Academy GitHub Repository Link

🔗 [https://github.com/SDAIAAcademy](https://github.com/SDAIAAcademy)
