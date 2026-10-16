# 🔍 Text-to-SQL LLM App

**Ask your database a question in plain English. Get back SQL, results, and a chart.**

A fine-tuned GPT-4o-mini model, wrapped in a small agent that knows when to ask "wait, what do you mean?" before it writes a query. Built with a Streamlit front end on top of SQLite.

> Project for **DATA 266: Generative Models** at San Jose State University (Spring 2025), under the guidance of Dr. Simon Shim

---

## Why we built this

In most organizations, the people who need data (managers, product teams, analysts) aren't the people who can write SQL. So they ask someone technical, wait, and that someone gets buried in ad-hoc requests.

We wanted to see how far we could close that gap: let a non-technical person ask a question in everyday language, and have a model handle the translation. And since natural language is messy ("show me the good movies" means... what, exactly?), we didn't want the model to just guess. We wanted it to ask.

## What it does

1. **Translates natural language to SQL** using a GPT-4o-mini model we fine-tuned on the Spider benchmark.
2. **Catches ambiguous questions** and asks clarifying questions instead of guessing. You can also expand a "Reasoning Process" panel to see how the model interpreted your question.
3. **Runs the query** against a SQLite database (pick one of the sample Spider databases or upload your own `.db` file).
4. **Visualizes the results automatically**: bar, line, scatter, or pie charts depending on the shape of the data, or a clean table when charts don't make sense.
5. **Keeps a query history** for the session so you can revisit earlier questions.

### An example

| You ask | The app does |
|---|---|
| *"List all students along with the city and state they are from"* | Sees no ambiguity, generates a `JOIN` between `Student` and `City`, and shows the results table |
| *"List the names of customers and the count of orders for each customer"* | Generates a `GROUP BY` query and draws a bar chart |
| *"Can you tell me the movies which are good"* | Stops and asks: *what does "good" mean here? Rating? Box office? Which table?* |

## How it works

```
User question (Streamlit)
        │
        ▼
 TextToSQLAgent ──► Understand intent & entities
        │
        ▼
   Ambiguous? (score > 0.5)
     │ yes                    │ no
     ▼                        ▼
 Ask clarifying          Generate SQL (fine-tuned GPT-4o-mini)
 questions                    │
     │                        ▼
     └─ user clarifies    Execute on SQLite
                              │
                              ▼
                    Numeric result? ── yes ──► Chart
                              │
                              no ──► Table
```

**A note on what this is and isn't:** this is *not* a RAG system. The database schema is serialized into text and placed directly in the prompt, with no retrieval step. The "agentic" part is the ambiguity detection and clarification loop around the model.

## The model

**Base model:** `gpt-4o-mini-2024-07-18`
**Method:** Supervised fine-tuning through the OpenAI fine-tuning API

We picked GPT-4o-mini after comparing T5-base, a Spider-pretrained T5, GPT-3.5, GPT-4, and LLaMA 2/3. It was fine-tunable, fast, cheap to run, and speaks the chat format natively, so it fit well with the data format we wanted.

### Training data

We used the [Spider](https://yale-lily.github.io/spider) dataset: 7,000 training examples and 1,034 validation examples across 200+ databases from different domains (education, business, entertainment, government, and more). Each example has a question, a gold SQL query, and a database ID.

Preparation steps:

- **Checked for malformed entries** (missing `db_id`, `question`, or `query`). Found none.
- **Serialized each schema** from `tables.json` into readable text: tables, columns, primary keys, and foreign keys.
- **Normalized the SQL** with `sqlparse` for consistent formatting.
- **Formatted everything as chat-style JSONL**: schema in the system message, the question as the user message, the SQL as the assistant message.

```json
{
  "messages": [
    {"role": "system", "content": "Table: head\nColumns: head_ID, name, born_state, age\n..."},
    {"role": "user", "content": "How many heads of the departments are older than 56 ?"},
    {"role": "assistant", "content": "select count(*) from head where age > 56"}
  ]
}
```

### Training configuration

| Setting | Value |
|---|---|
| Epochs | 4 (1,556 steps) |
| Learning rate multiplier | 0.05 |
| Batch size | 18 |
| Trained tokens | ~9.4M |
| Final training loss | ≈ 0.17–0.20 |
| Final training token accuracy | ≈ 0.94 |

To be upfront: we settled on this configuration from OpenAI's guidelines and a couple of trial runs, not a systematic hyperparameter search. We also didn't pass a validation file to the fine-tuning job, so the loss and accuracy numbers above are training metrics only.

## Results

We compared the base and fine-tuned models on the first 20 questions of Spider's dev set, using three metrics:

| Metric | Base GPT-4o-mini | Fine-tuned |
|---|---|---|
| Queries returning the correct result | 17 / 20 | 17 / 19 evaluated* |
| Average BLEU vs. gold SQL | 0.37 | 0.57 |
| Exact string match with gold SQL | 4 / 20 | 8 / 19 |

\*One fine-tuned query failed to execute and was excluded from the metrics. Counting it as a miss, the fine-tuned model also lands at 17 / 20.

**What we take from this, honestly:**

- **The biggest effect of fine-tuning was style, not correctness.** The fine-tuned model learned Spider's conventions (lowercase keywords, no `AS` aliases, no trailing semicolons), which is why BLEU and exact match went up. On execution accuracy, the base model was already strong, and on this small sample the two are indistinguishable.
- **Twenty questions is a small sample.** A one-question difference moves the numbers by 5 percentage points, so please read these as a sanity check and not a benchmark. The evaluation notebook is set up to run on more; extending it to the full 1,034-question dev set is the first thing we'd do next.
- **The evaluation prompt differed from the training prompt.** Evaluation used a different schema format (no primary or foreign keys), which probably understates what the fine-tuned model can do.

## Tech stack

- **Model & API:** OpenAI GPT-4o-mini (fine-tuned), OpenAI Python SDK
- **App:** Streamlit
- **Database:** SQLite
- **Data & evaluation:** Hugging Face `datasets`, pandas, `sqlparse`, NLTK (BLEU)
- **Language:** Python

## Project structure

```
├── app.py                  # Streamlit app: UI, session state, input/output flow
├── agent_components.py     # TextToSQLAgent: query understanding, ambiguity check, SQL generation
├── database_utils.py       # DB connections and schema extraction (via PRAGMA)
├── visualization.py        # Picks and renders the right chart for the results
├── data/
│   ├── spider_databases/   # Sample SQLite databases (address, bbc, cinema, flight, pets, ...)
│   └── Load_data_spider/   # SQL schema scripts
└── notebooks/
    ├── Text_to_SQL_LLM.ipynb   # Data prep, fine-tuning, and first evaluation
    └── Evaluate04.ipynb        # Final base vs. fine-tuned comparison
```

## Running it yourself

1. Clone the repo and install dependencies:
   ```bash
   pip install streamlit openai pandas sqlparse datasets nltk
   ```
2. **Fine-tune your own model.** Our fine-tuned model lives in a private OpenAI organization, so you'll need to create your own. Use the notebook to build the JSONL file from Spider, launch a fine-tuning job, and grab the resulting model ID. (You can also point the app at a base model like `gpt-4o-mini` to try it out.)
3. Start the app:
   ```bash
   streamlit run app.py
   ```
4. In the sidebar, enter your OpenAI API key and model ID, click **Initialize LLM**, choose a database, click **Connect Database**, and ask away.

## What we'd do next

- Evaluate on the full Spider dev set, with the same prompt format used in training
- Use Spider's official evaluation script and report execution accuracy as the headline metric
- Add schema retrieval (true RAG) so it scales to databases with hundreds of tables
- Hold out a validation set during fine-tuning and try a small hyperparameter search
- Support multi-turn conversations, in the spirit of CoSQL

## What this project taught me

Fine-tuning is great at teaching a model *how* to answer (format, conventions, tone) and much less obviously at making it *smarter* when the base model is already capable. The more interesting win turned out to be the product design: letting the model say "I'm not sure what you mean" made the whole thing feel more trustworthy than a model that confidently guesses.

## References

- Yu et al. (2018). *Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SQL Task.* EMNLP.
- Zhong et al. (2017). *Seq2SQL: Generating Structured Queries from Natural Language using Reinforcement Learning.*
- Yu et al. (2019). *CoSQL: A Conversational Text-to-SQL Challenge Towards Cross-Domain Natural Language Interfaces to Databases.* EMNLP.
- Rubin & Berant (2021). *SmBoP: Semi-autoregressive Bottom-up Semantic Parsing.*
- Shi et al. (2022). *Learning to Parse with Instruction Tuning.*

---
