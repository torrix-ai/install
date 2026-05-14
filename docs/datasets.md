# Dataset Evals

Datasets let you define a set of test inputs with expected outputs, run them in batch against any model, and automatically score each result. Use them to validate prompt changes, compare models side by side, or track quality over time.

## When to use datasets

- You have a set of known question-answer pairs and want to verify a model still gets them right after changing a prompt
- You want to run the same inputs against two different models and compare pass rates
- You want a repeatable quality gate you can re-run whenever you change your system prompt or switch providers

## Creating a dataset

1. Go to **Evals** in the left sidebar.
2. Scroll to the **Datasets** section and click **+ New Dataset**.
3. Enter a name (e.g. "Capital cities quiz") and an optional description.
4. Click **Create Dataset**.

Community edition: up to 3 datasets. Pro: unlimited.

## Adding rows

Each row is one test case. A row has:

| Field | Required | Description |
|---|---|---|
| Input | Yes | The user message sent to the model |
| Expected output | No | The correct answer. When set, scoring uses exact match first, then an LLM judge as fallback |
| Name | No | A short label for this row (e.g. "France capital") |

To add a row, expand the dataset by clicking its name, fill in the fields at the bottom of the detail panel, and click **+ Add Row**.

To edit an existing row, click the pencil icon on the right of any row.

Community edition: up to 10 rows per dataset. Pro: unlimited.

## Importing rows from a CSV file

To bulk-add rows, click **Import CSV** next to the **+ Add Row** button inside the expanded dataset panel.

**CSV format requirements:**

- The first row must be a header row.
- The column named `input` is required. Columns named `expected` and `name` are optional.
- Column order does not matter. Any columns not named `input`, `expected`, or `name` are ignored.
- Values containing commas must be wrapped in double quotes.

**Example:**

```csv
input,expected,name
What is the capital of France?,Paris,France capital
What is 2 + 2?,4,Basic arithmetic
Translate "hello" to Spanish.,hola,Translation
```

After selecting the file, Torrix parses it client-side and sends all rows to the server in a single batch. A confirmation message shows how many rows were added (for example, "Added 47 rows").

There is no row limit during import other than the per-dataset row limit for your edition.

## Running a dataset

1. Click **Run** next to the dataset name.
2. Fill in the run configuration:
   - **Provider URL**: the full chat completions endpoint (e.g. `https://api.openai.com/v1/chat/completions`)
   - **API Key**: your provider API key (used only for this run, never stored in Torrix)
   - **Model**: the model to test against (e.g. `gpt-4o-mini`)
   - **Temperature**: sampling temperature (default 0.7)
   - **System prompt**: optional system message prepended to every row
3. Click **Run Dataset**.

Progress is streamed in real time. Each row shows the response, score, latency, and cost as it completes.

## Scoring logic

For rows with an expected output:

1. Torrix normalizes both the response and the expected value (case-insensitive, strips trailing punctuation and extra whitespace) and checks for an exact match. If they match, the row scores as passed.
2. If exact match fails, the configured LLM judge is called with the expected output as the scoring criterion. The judge returns pass or fail.
3. If neither check can score the row (judge not configured or unreachable), the row is marked as failed because an expected answer was provided and was not matched.

For rows without an expected output, the general quality judge is used. If the judge is not configured or fails, the row is left unscored.

## Reading results

The results table shows one row per test case:

| Column | Meaning |
|---|---|
| Input | Truncated preview of the input sent |
| Response | First 60 characters of the model response (hover for full preview) |
| Score | Green checkmark for pass, red X for fail, dash for unscored |
| Latency | Time to first response byte |
| Cost | Estimated cost for this row |

Click any row to open the full run detail page in a new tab.

The summary line below the table shows total passed out of total rows and the combined cost for the run.

The **Pass Rate** column on the datasets list shows the cumulative pass rate across all runs ever executed for that dataset.

## Viewing run history

Every dataset run is saved to the Runs table with `source = dataset`. To see all runs for a specific dataset, go to **Runs** and filter by the dataset tag or search by model name.
