# FinMind

FinMind is a Python notebook that explores how financial language, possible cognitive biases and recent news can inform a financial conversation. You ask a question, and it combines text analysis with relevant articles to generate an educational answer with source references.

For example, a question such as ?Everyone is buying NVIDIA. Am I missing out?? can prompt the system to examine possible FOMO or herd mentality, find related news, and explain considerations behind the question. These labels are tentative interpretations of the wording, not diagnoses.

The project runs in a Jupyter notebook. It does not place trades, manage portfolios or predict stock prices.

## How it works

```text
Question + recent conversation
          |
          +--> FinBERT: financial sentiment
          +--> Gemini: possible biases, tone and market topic
                         |
                    GNews articles
                         |
                  Clean and deduplicate
                         |
                 MiniLM embeddings
                         |
                FAISS similarity search
                         |
              Topic, date and URL filters
                         |
               FinBERT news sentiment
                         |
            Gemini answer + checked source IDs
```

1. **Analyze the question.** FinBERT scores positive, negative and neutral financial language. Gemini separately identifies possible biases, expressed tone and whether the question needs market news.
2. **Find news.** GNews searches the identified topic. Results are cached for six hours to reduce repeated requests.
3. **Select relevant articles.** MiniLM converts titles and descriptions into numerical vectors representing their meaning. FAISS searches those vectors, then the code checks topic aliases, publication age and source URLs. Live articles must be within 14 days; up to five are retained.
4. **Summarize news sentiment.** FinBERT scores the selected articles. The average positive-minus-negative probability describes that sample, not the whole market.
5. **Generate the answer.** Gemini receives the question, analysis, articles and recent conversation. Python checks its source IDs and adds references using retrieved article metadata.

General questions such as ?What is mental accounting?? can skip news retrieval. Follow-up questions use recent conversation to resolve the topic. If news is unavailable, the answer is instructed to acknowledge the gap.

## Main components

| Component | Role |
|---|---|
| FinBERT (`ProsusAI/finbert`) | Financial sentiment for questions and articles |
| Gemini | Structured bias, tone and topic analysis; final answer generation |
| GNews | News search |
| MiniLM (`all-MiniLM-L6-v2`) | Text embeddings for semantic similarity |
| FAISS | Vector search over the article collection |
| Pandas | Article cleaning and result tables |
| Matplotlib | Sentiment charts |
| JSON Schema | Required fields, types and allowed values in Gemini responses |

The analysis schema separates `biases`, `tone` and `market`. Bias entries include an explanation, an exact supporting quote and evidence strength. Python rejects bias entries whose quote does not appear in the current question. Schema validation checks response structure; it does not establish factual correctness.

## Project files

```text
FinMind.ipynb        All application code, examples, charts and local checks
requirements.txt    Python dependencies
.env.example        API configuration template
.env                Your local keys and model setting (ignored by Git)
data/news_cache.json Created at runtime for cached news (ignored by Git)
.gitignore          Files excluded from Git
```

## Setup

Use Python 3.12. From the project folder in PowerShell:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements.txt
```

If you do not already have `.env`, create it from the template:

```powershell
Copy-Item .env.example .env
```

Fill in `GEMINI_API_KEY`, `GNEWS_API_KEY` and `GEMINI_MODEL`. Choose a Gemini model your account can access that supports structured JSON responses. The template's model name is a configuration value, not a guarantee of availability.

The notebook checks model access before analysis. If verification fails, run `list_available_models()`, update `GEMINI_MODEL`, and restart the kernel. Restart after changing keys too, because existing environment variables are preserved.

Local models run on CPU and download their weights on first use. Allow disk space and internet access for these downloads. Gemini and GNews also require internet access and available API quota.

## Run the notebook

1. Open `FinMind.ipynb` in VS Code with the Python and Jupyter extensions.
2. Select `.venv\Scripts\python.exe` as the kernel.
3. Run all cells in order. With the default switches, this loads definitions, runs local checks and lists example questions without API calls or model downloads.
4. Add a code cell and ask a question:

```python
answer = ask_finmind("Everyone is buying NVIDIA. Am I missing out?")
```

By default, the function displays financial sentiment, possible biases, tone, selected articles and the final answer. It also returns the answer as a string.

Continue the conversation or start a new one:

```python
ask_finmind("What if the price keeps rising?")

reset_conversation()
ask_finmind("What is mental accounting?")
```

To show only the returned answer:

```python
answer = ask_finmind("Why do investors avoid selling at a loss?", show_details=False)
print(answer)
```

You can also launch JupyterLab directly:

```powershell
.\.venv\Scripts\python -m jupyterlab
```

## Inspect results and charts

`last_result` stores the latest question, success status and available results. Successful runs include `analysis`, `news`, `market_sentiment`, `news_status`, `answer` and `citations_valid`. Failed runs include an `error`; fields from unfinished stages may be absent.

```python
if last_result and last_result.get("success"):
    display(last_result["news"])
    plot_news_sentiment(last_result["news"])
```

Charts show article sentiment counts and probabilities. A publication-date chart appears when enough dated articles are available. These describe the retrieved sample, not historical market performance.

## Demo and evaluation options

Set these switches in the configuration cell before running the notebook:

| Switch | What it does |
|---|---|
| `RUN_OFFLINE_DEMO=True` | Runs MiniLM, FAISS and FinBERT on fictional articles without API keys. Initial model downloads still require internet. |
| `OFFLINE_DEMO=True` | Makes questions use fictional articles with actual Gemini analysis and answers. A Gemini key is still needed. |
| `RUN_LIVE_EXAMPLE=True` | Runs the two-question follow-up example using the configured news mode. |
| `RUN_EVALUATION=True` | Runs ten independent questions and records results and errors. Uses Gemini quota and, in live news mode, GNews quota. |

Fictional articles are labeled and are never silently substituted when a live API fails. `reset_conversation()` clears conversational state and the analysis cache; it does not delete the disk news cache.

## Checks and limitations

The default notebook execution and local checks have passed. Local checks cover article cleaning, topic matching and citation membership. A direct structured Gemini request with gemini-3.1-flash-lite has passed. The complete live pipeline, GNews retrieval and pretrained model inference have not been verified end to end in this workspace. The optional evaluation records behavior; it is not an accuracy benchmark with labeled expected answers.

FinBERT measures financial language rather than emotions. Bias and tone interpretations can be wrong. News may be delayed, truncated or unrepresentative, and topic filters can miss relevant articles. A valid citation ID means the source was retrieved, not that it supports every claim in the answer. Review the referenced articles before relying on a response.

FinMind is for educational use, not personalized investment decisions. News sentiment does not predict future returns.

## Data and privacy

Questions, recent conversation and selected article data are sent to Google for analysis and generation. Search terms are sent to GNews. News metadata is cached locally in `data/news_cache.json`.

Keep `.env` out of Git and clear notebook outputs containing private questions before sharing the notebook.
