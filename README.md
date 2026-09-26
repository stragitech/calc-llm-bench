# Calculus I LLM Evaluation

This repository compares language models on a multiple-choice Calculus I dataset from the University of Houston. It contains notebook-based evaluation pipelines for OpenAI, Anthropic Claude, and locally hosted Ollama models.

The notebooks currently use `df.head(10)`, so their default runs evaluate the first 10 questions rather than all rows in the dataset.

## Repository structure

```text
.
├── .env
├── .gitignore
├── README.md
├── requirements.txt
├── data/
│   └── input_data.csv
├── notebooks/
│   ├── claude-structured-output-test.ipynb
│   ├── ollama-unstructured-output-test.ipynb
│   ├── openai-structured-output-test.ipynb
│   └── openai-unstructured-output-test.ipynb
└── output/
    ├── claude-opus-4-5-20251101/
    ├── gpt-4o-2024-08-06/
    ├── gpt-5-mini/
    └── llama3.2-1b/
```

The tree includes the local `.env` file, local `data/`, and generated `output/` directories; these are excluded from Git and are not supplied with the repository. The `output/` subdirectories are generated from the model selected in each notebook. Checkpoints and result files therefore remain separated when the model changes.

## Dataset

The dataset is not distributed with this repository. Obtain an authorized copy and place it at:

```text
data/input_data.csv
```

It contains 331 (after removing errors it contains 325) rows and three columns:

| Column | Meaning |
| --- | --- |
| `ID` | Unique question identifier used as the checkpoint key |
| `Question` | Calculus I multiple-choice question and options in HTML with embedded LaTeX math format |
| `Correct Answer` | Expected lowercase answer-choice letter, or `NA` when no listed solution matches |

All notebooks locate the dataset relative to the repository root, so they can be launched from either the root directory or `notebooks/`. They load CSV files with `keep_default_na=False` to preserve literal `NA` values, summarize the dataset, and inspect duplicate rows, questions, and IDs.

## Evaluation notebooks

| Notebook | Default model | Response strategy | Service |
| --- | --- | --- | --- |
| `openai-unstructured-output-test.ipynb` | `gpt-4o-2024-08-06` | Generates a free-form solution, then uses `gpt-4o-mini` to extract a structured answer letter | OpenAI API |
| `openai-structured-output-test.ipynb` | `gpt-5-mini` | Prompts the Responses API for JSON with web search disabled and validates the response locally with Pydantic | OpenAI API |
| `claude-structured-output-test.ipynb` | `claude-opus-4-5-20251101` | Uses `client.messages.parse()` with a Pydantic output model | Anthropic API |
| `ollama-unstructured-output-test.ipynb` | `llama3.2:1b` + `gpt-4o-mini` | Ollama generates a free-form solution; `gpt-4o-mini` extracts the answer letter with structured output | Ollama + OpenAI APIs |

### Shared notebook structure and instructions

All four notebooks have the same seven sections:

1. Imports and configuration.
2. Provider clients and response schemas.
3. Load and inspect the dataset.
4. Primary-model response pipeline, including the example and system instructions.
5. Prepare answer choices.
6. Normalize and evaluate answers.
7. Export results.

The few-shot example and core `SYSTEM_PROMPT` are identical across the notebooks. Each model must explain its solution and return the matching option letter. If no listed solution matches the calculation, it must return `NA`, including when a “None of the above” option exists. `RESPONSE_FORMAT_INSTRUCTIONS` specifies JSON fields for the structured workflows and a plain-text `Final answer: <option letter>` or `Final answer: NA` line for the unstructured workflows.

In section 5, structured workflows read answers from their checkpoints. Unstructured workflows use the extraction model to identify the final answer and preserve `NA`. All four then use the same normalization and evaluation code.

### OpenAI unstructured output

[Open the notebook](notebooks/openai-unstructured-output-test.ipynb)

This is a two-stage pipeline:

1. The primary model produces a free-form explanation and answer.
2. `gpt-4o-mini` converts that response to a structured `correct_option_choice_letter` value.

The primary responses are checkpointed before extraction. A complete 10-question run normally makes up to 10 primary-model calls and 10 extraction-model calls. Both stages use `OPENAI_API_KEY` and may incur API charges.

### OpenAI structured output

[Open the notebook](notebooks/openai-structured-output-test.ipynb)

Changing `MODEL_NAME` changes the output directory and model portion of the generated filenames; `RUN_NAME` can also be adjusted.

This workflow prompts the Responses API to return JSON and validates the result locally with Pydantic; it does not request API-enforced structured output. Web search is disabled; the request includes no tools. It uses `OPENAI_API_KEY` and makes billable API calls.

### Claude structured output

[Open the notebook](notebooks/claude-structured-output-test.ipynb)

This notebook uses Anthropic's native structured-output parser. The `Calculus` Pydantic model defines the expected explanation and answer-choice fields, and `response.parsed_output` supplies the validated result.

It uses `ANTHROPIC_API_KEY` and makes billable Anthropic API calls.

### Ollama unstructured output with OpenAI extraction

[Open the notebook](notebooks/ollama-unstructured-output-test.ipynb)

This notebook uses an unstructured primary Ollama request. Ollama produces a free-form solution with temperature `0`; a separate `gpt-4o-mini` call then extracts `correct_option_choice_letter` using OpenAI structured output.

The primary solution is checkpointed before extraction. A complete 10-question run can make up to 10 local Ollama calls and 10 billable OpenAI extraction calls. This workflow requires both a running local Ollama model and `OPENAI_API_KEY`.

Ollama model tags contain characters such as `:` that are inconvenient in portable filenames. The notebook converts the tag to a filesystem-safe artifact name:

```text
llama3.2:1b  ->  llama3.2-1b
```

Ollama must be installed, its server must be running, and the selected model must be available locally. The OpenAI extraction stage requires `OPENAI_API_KEY`.

## Installation

The latest dependency resolution and offline notebook checks used Python 3.11 on macOS ARM64. Use Python 3.11 to match that validation environment. The saved notebook metadata still names a Python 3.10 kernel; select the environment created below when running the notebooks.

Create and activate a virtual environment:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
```

On Windows PowerShell, create and activate it with:

```powershell
py -3.11 -m venv .venv
.venv\Scripts\Activate.ps1
```

Install the Python dependencies and register the environment as a notebook kernel:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m ipykernel install --user --name calculus-llm-eval --display-name "Calculus LLM Eval"
```

Then start JupyterLab:

```bash
jupyter lab
```

Select the **Calculus LLM Eval** kernel when opening a notebook.

`requirements.txt` is the source of truth for package minimum versions, including the updated JupyterLab and `python-dotenv` security minimums. It is not a pinned lockfile; fresh installations can resolve newer package versions.

## Environment variables

Create a `.env` file in the repository root and fill in the required values:

```dotenv
OPENAI_API_KEY=your_openai_api_key
ANTHROPIC_API_KEY=your_anthropic_api_key
```

Each notebook explicitly loads `.env` from `PROJECT_ROOT`; existing process environment variables take precedence. Only add the key needed by the notebook you intend to run. The Ollama notebook also needs `OPENAI_API_KEY` for its `gpt-4o-mini` extraction stage.

Never commit `.env` or share its contents. API keys should be revoked and replaced immediately if exposed.

## Ollama setup

The Python package in `requirements.txt` is only the client. Install the Ollama application separately, start the server, and pull the model configured in the notebook:

```bash
ollama serve
ollama pull llama3.2:1b
```

If Ollama is already running as a background service, only the `pull` command is needed. To use a different local model, change `MODEL_NAME` in the notebook and pull the matching tag first.

## Running an evaluation

1. Open the desired notebook.
2. Select the project environment's kernel.
3. Review `MODEL_NAME` and, where present, `EXTRACTION_MODEL_NAME`.
4. Confirm the required API key or local Ollama model is available.
5. Run the notebook from top to bottom.
6. Review the printed classification metrics and the files under `output/<model>/`.

Full runs can create substantial API cost. Check provider pricing, model access, and rate limits before increasing the sample size.

## Model and artifact naming

The model configuration is the single source of truth for artifact locations:

```python
MODEL_NAME = "provider-model-name"
RUN_NAME = f"{MODEL_NAME}-subsample"
```

The Ollama notebook additionally derives `MODEL_ARTIFACT_NAME` by replacing `:` and `/` with `-`.

A typical run creates:

```text
output/<model-artifact-name>/
├── <run-name>-checkpoint.json
├── <run-name>-test-output.csv
└── <run-name>-test-output-incorrect.csv
```


## Checkpoints and resuming

Each checkpoint is a JSON object keyed by the dataset's string-form `ID`. Before making a request, the notebooks check whether that ID is already present. Existing entries are skipped, making interrupted runs resumable.

Structured-output checkpoints from Claude and the OpenAI Responses workflow store:

```json
{
  "question-id": {
    "detailed_solution_explanation": "...",
    "correct_option_choice_letter": "c"
  }
}
```

The unstructured OpenAI and Ollama checkpoints store the primary response before answer extraction:

```json
{
  "question-id": {
    "Long Answer": "..."
  }
}
```


To rerun questions with the same model, choose a new `RUN_NAME` or move the existing checkpoint after preserving anything needed. Use a new `RUN_NAME` after changing prompts, examples, or the dataset: checkpoints are matched by question ID and do not detect those changes. Changing `MODEL_NAME` automatically selects a different model-specific output directory.

Only primary-model responses are checkpointed. Rerunning section 5 in either unstructured notebook makes new billable extraction calls for each non-empty solution.

## Result files

The complete result CSV contains the source fields plus generated evaluation fields:

| Column | Meaning |
| --- | --- |
| `ID` | Dataset question identifier |
| `Question` | Original multiple-choice question |
| `Correct Answer` | Ground-truth choice |
| `LLM Answer Explanation` | Generated solution or explanation |
| `LLM Answer Choice (RAW)` | Provider response before final letter normalization |
| `LLM Answer Choice` | Lowercase option letter, `NA`, or an empty string for an invalid/missing prediction |
| `Correct` | `1` when the prediction matches the answer key, otherwise `0` |

The `-test-output-incorrect.csv` file contains only rows where `Correct == 0`.

The workflows report accuracy, weighted precision, weighted recall, weighted F1, and a classification report. `NA` is preserved as an answer label and counts as correct only when the answer key is also `NA`. Malformed answer JSON, non-string values, and unsupported answer text normalize to an empty string; they are not converted to `NA`. Missing or invalid predictions remain in the evaluation.

When reloading result CSVs with pandas, use `pd.read_csv(path, keep_default_na=False)` to preserve the literal `NA` label.

## Troubleshooting

### A provider import fails

Activate the intended environment, reinstall the requirements, and restart the notebook kernel:

```bash
python -m pip install -r requirements.txt
```

### An API key is missing or rejected

Verify the relevant variable in `.env`, ensure the notebook was launched with the repository as its working project, and restart the kernel after changing environment variables.

### Ollama cannot connect

Confirm that the server is running and that the configured model is installed:

```bash
ollama list
ollama serve
```

### A run skips every question

The model-specific checkpoint already contains those IDs. This is expected resume behavior. Inspect the checkpoint or use a new run/model name if the goal is a separate experiment.

### A structured response cannot be parsed

Provider output may occasionally violate the schema. The Claude notebook catches query/validation failures and leaves the question absent from the checkpoint so it can be retried. The OpenAI extraction stages catch per-row extraction failures and leave the raw solution available for another extraction attempt. The OpenAI Responses notebook currently raises parsing errors directly; rerun the failed cell or add retry handling before a large unattended run.

## Reproducibility notes

- Model outputs can change across provider revisions even when a dated model name is used.
- Local Ollama results depend on the installed model build, Ollama version, hardware, and runtime configuration.
- Checkpoints prevent accidental duplicate calls but can mix results if prompt logic changes without changing `RUN_NAME`.
- Record package versions, prompt revisions, run date, and model access settings when producing publishable comparisons.
- Saved notebook outputs are cleared for publication. Generated results stay local under `output/`.

## API references

- [OpenAI Responses API](https://platform.openai.com/docs/api-reference/responses)
- [OpenAI structured outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)
- [Ollama unstructured chat responses](https://docs.ollama.com/api/chat)

## Publication and data handling

The repository ignores `.env` files, local datasets, generated responses, and notebook checkpoints. These exclusions do not remove files already committed to Git. Clear all notebook outputs before committing; outputs can contain dataset questions, answers, local paths, or service errors.

Use only data approved for the selected provider. OpenAI and Claude workflows send questions to external APIs; the Ollama workflow sends generated solutions to OpenAI for answer extraction. Web search is disabled in the OpenAI Responses notebook, but questions are still sent to OpenAI for inference.

Treat generated CSV text as untrusted when opening it in spreadsheet software: import text columns as text to avoid interpreting formula-like content. Keep the Ollama server local and retain Jupyter authentication.

Before each release, install `pip-audit` in a separate audit environment, audit a fresh dependency resolution with `pip-audit -r requirements.txt`, and scan the complete Git history for secrets. Requirements specify minimum versions rather than a reproducible lock; record the installed versions used for published experiments.

Choose an appropriate code license and confirm permission to distribute the dataset and the few-shot example before publication. The repository currently contains no license file.
