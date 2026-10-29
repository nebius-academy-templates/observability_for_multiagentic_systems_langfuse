In this task, you'll connect the level 4 code review agent to Langfuse and use it to observe every node execution, LLM call, and conditional routing decision in real time.

You'll create a Langfuse project, configure the credentials, add trace tags to distinguish scenarios, and then trace three distinct runs to understand what observability looks like in a LangGraph pipeline.

## 👩‍💻Workflow at a glance

1. Create a Langfuse project
2. Configure credentials
3. Add trace tags to `main.py`
4. Run the three scenarios
5. Explore the traces
6. Submit your task

> ⚠️ This task uses an LLM provider.
>
> The default setup uses Anthropic Claude. If you do not have an Anthropic API key, you can still complete the task by switching the project to another provider.
>
> Recommended **no-cost** options:
>
> - **Groq**: hosted API with a free plan and an OpenAI-compatible endpoint.
> - **Google Gemini**: hosted API with a free developer tier. Good option if you already use Google AI Studio.
> - **Ollama**: local model runner. You need to install the runner, as well as any models you want to use, and need to have enough local machine resources to run them.

## 1. Create a Langfuse project

1. Go to [cloud.langfuse.com](https://cloud.langfuse.com/) and sign in (or create a free account).
2. Create a new project and name it `level-4-code-review`. If Langfuse asks for an organization first, create one with any name.
3. Open the project and confirm it's empty.
4. Go to "Settings" → "API Keys" and create a new key pair. Copy both keys: you won't be able to see the secret key again.

### Repository check

Download [`observability_for_multiagentic_systems_langfuse`](https://github.com/nebius-academy-templates/observability_for_multiagentic_systems_langfuse) and install its dependencies, which include `langfuse`. Use Python 3.11 or newer:

```
pip install -r requirements.txt
```

## 2. Configure credentials

Open `.env.example` in the project root and copy it to a new file called `.env` in the same directory (or create `.env` from scratch if you prefer). Set `ANTHROPIC_API_KEY` to your Anthropic key and keep `ANTHROPIC_OPEN_TELEMETRY=false`: the Langfuse handler in `main.py` already records every Claude call, and without this line the Anthropic SDK adds a second copy of each call to the trace. Then add the following three Langfuse variables:

```
LANGFUSE_PUBLIC_KEY=pk-lf-your_key_here
LANGFUSE_SECRET_KEY=sk-lf-your_key_here
LANGFUSE_BASE_URL=https://cloud.langfuse.com
```

Replace both placeholders with the keys you copied in step 1.

The key pair also picks the Langfuse project: traces go to the project the keys belong to. `main.py` already passes a Langfuse `CallbackHandler` to the graph, so once the keys are set, every node execution, LLM call, and conditional edge decision is captured without any code changes to your graph.

> **Note on variable naming:** Older Langfuse examples use `LANGFUSE_HOST` for the base URL. It still works, but current Langfuse docs use `LANGFUSE_BASE_URL`.

Verify that `python-dotenv` loads the file by running:

`python -c "import os; from dotenv import load_dotenv; load_dotenv(); print(os.getenv('LANGFUSE_BASE_URL'))"`

Expected output: `https://cloud.langfuse.com`.

## 3. Add trace tags to `main.py`

Right now, `main.py` passes a `config` dict with the Langfuse handler to the graph. Extend the config to also include tags so each scenario is easy to filter in Langfuse.

Find the `config` dict inside the `run()` function and add a `tags` key and a `metadata` key. `run_id` is already created right above the dict:

```python
config = {
    "callbacks": [langfuse_handler],
    "tags": [label],
    "metadata": {
        "filename": label,
        "run_id": run_id,
    },
}
```

The `label` argument is already `"buggy diff"` or `"clean diff"`. It will appear as a tag on every trace in the Langfuse UI, making it trivial to filter one scenario from the other.

## 4. Run the three scenarios

Run `main.py` twice to produce three distinct traces.

### Scenario A: Security blocked (buggy diff)

`security_auditor` lists the security issues it finds in the diff, and `security_gate` blocks the pipeline when that list is not empty.

Run normally:

```
python main.py
```

This produces two traces in one run. Focus on the buggy diff trace. It should be blocked by `security_gate` before reaching `summarizer`.

### Scenario B: Full pipeline (clean diff)

The second trace produced by the same run covers `clean.diff`. It should pass through `security_gate` and reach `summarizer`. If the model scores the clean diff below 8, `quality_router` sends it back to `bug_detector` for a second pass, so run it again.

### Scenario C: Force a retry loop

To trigger the `quality_router` retry branch, temporarily raise the score threshold in `graph.py`:

```python
# change this line inside quality_router:
if score >= 8:
```

to:

```python
if score >= 10:
```

Run again:

```
python main.py
```

This forces the clean diff to loop back to `bug_detector` until `retry_count` reaches 2. If the model scores the clean diff 10, the run ends without a retry, so run it again. Restore the threshold to `>= 8` after the run.

## 5. Explore the traces

Open [cloud.langfuse.com](https://cloud.langfuse.com/), open the `level-4-code-review` project, go to **Tracing**, and confirm that traces are appearing.

### What to verify for Scenario A (buggy diff: blocked)

- The trace is tagged `buggy diff`
- The node sequence ends at `security_auditor`, and `summarizer` is absent
- Click into `security_auditor` → open **Preview** → confirm `security_issues` in the output is a non-empty list
- Click into the `ChatAnthropic` generation inside `security_auditor` (the LLM call, named after the chat model class) → inspect the system prompt and the raw model response

### What to verify for Scenario B (clean diff: full pipeline)

- The trace is tagged `clean diff`
- The node sequence includes `summarizer` after `security_auditor`
- Click into `security_auditor` → confirm `security_issues` is `[]`
- Click into `summarizer` → confirm `score` and `summary` are present in the output
- Click into `quality_router` (shown under `summarizer`) → confirm its output is `END`

### What to verify for Scenario C (retry loop)

- The same nodes appear twice in the trace: `bug_detector`, `style_reviewer`, `improvement_advisor`, `security_auditor`, `summarizer`
- The `retry_count` value in `summarizer`'s output increases from `1` to `2` across the two passes
- The run ends after the second pass because `retry_count >= 2` forces the exit

### Langfuse trace view reference

| Panel | What you find there |
| --- | --- |
| **Timeline** | Visual sequence of all node and LLM calls with duration, next to the **Tree** and **Graph** views |
| **Preview** | The state entering each node and the fields the node returned |
| **Attributes** → **Metadata** | `filename` and `run_id` values passed from `main.py` |
| Tags in the trace header | `buggy diff` or `clean diff` label for filtering in **Tracing** |
| `ChatAnthropic` generations | Raw system prompt, user message, and model response for every `_call()` invocation |

## 6. Submit your task

Before submitting, review the checklist.

### ✅ Submission checklist

- [ ] Langfuse project `level-4-code-review` exists and is accessible
- [ ] `main.py`: `"tags": [label]` added to the `config` dict
- [ ] Scenario A trace: tagged `buggy diff`, `summarizer` absent, `security_issues` non-empty
- [ ] Scenario B trace: tagged `clean diff`, `summarizer` present, `security_issues` is `[]`
- [ ] Scenario C trace: node sequence shows two full passes, `retry_count` reaches `2`
- [ ] `quality_router` threshold restored to `>= 8` after Scenario C
- [ ] The project folder with your changes is packed as a .zip

Submit your task as a .zip.
