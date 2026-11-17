# Parse PDFs with Dynamo Batch

Use the Curator Parse tutorial to submit PDF pages to Dynamo Batch and write the results to Parquet. Curator prepares the PDFs and postprocesses the responses. The remote Dynamo endpoint runs the Parse model.

Start with [Parse one PDF](curator/tutorials/interleaved/nemotron_parse_pdf/BATCH_ASYNC_PREVIEW.md). It walks through the client setup, submission and output checks.

## Included files

- `curator/tutorials/interleaved/nemotron_parse_pdf/` contains the Batch client, tutorial and instructions to copy into the specified Curator checkout.
- `dynamo/examples/deployments/batch-gateway/profile-parse.yaml` is an Async deployment template for endpoint operators. Set its storage and deployment values before using it.
- `provenance.json` records the source revisions and tested dependency versions.
- `tests/` contains offline client tests.

## Current status

An earlier one-page run completed through the public Files and Batch APIs and produced readable Parquet output. The current source adds output validation and phase timings. Those changes have offline coverage. A fresh installation and the Async deployment still need an end-to-end test.

Read the [validation notes](curator/tutorials/interleaved/nemotron_parse_pdf/VALIDATION.md) for tested versions, resource settings and known Parse quality issues. The [benchmark notes](BENCHMARK_NOTES.md) explain the measured Direct and Batch timings and what we still need to profile.

## Run the offline tests

From the repository root, use Python 3.12 with pytest, HTTPX, PyArrow and Pillow installed:

```bash
python -m pytest -q examples/nemotron-parse/tests
```

These tests use synthetic pages and mock HTTP responses. They do not need Curator, credentials or a GPU. Test the recipe against your endpoint before using it for a larger workload.
