# Parse one PDF with Dynamo Batch

This tutorial prepares a PDF with Curator, sends its pages to a Dynamo Batch endpoint running Parse 3.5, and writes the parsed content to Parquet.

## What you need

- A Dynamo Batch endpoint serving Parse, including its API URL ending in `/v1`, model name and authentication headers. Ask the endpoint operator for these.
- A Linux machine with Python 3.12, four available CPUs and writable temporary storage. PDF preparation and postprocessing run here. The model runs on the endpoint's GPU.
- A sample PDF and the Curator and Parse client packages described below.

An earlier one-page run passed on a configured host. A clean installation and the included Async deployment still need testing. See [validation notes](VALIDATION.md) for the tested environment and current limitations.

## 1. Copy the tutorial into Curator

Set `HANDOFF_DIR` to this repository's `examples/nemotron-parse` directory. Set `CURATOR_DIR` to a clean Curator checkout at commit `b56bd6cd4fd563d2acadc463cbf01c92a12ef0bb`. Copy the tutorial files:

```bash
test "$(git -C "$CURATOR_DIR" rev-parse HEAD)" = b56bd6cd4fd563d2acadc463cbf01c92a12ef0bb
test -z "$(git -C "$CURATOR_DIR" status --porcelain)"
for name in batch.py batch_client.py BATCH_ASYNC_PREVIEW.md VALIDATION.md; do
  cp "$HANDOFF_DIR/curator/tutorials/interleaved/nemotron_parse_pdf/$name" \
     "$CURATOR_DIR/tutorials/interleaved/nemotron_parse_pdf/$name"
done
export PYTHONPATH="$CURATOR_DIR"
export NO_VCS_VERSION=1
export CUDA_VISIBLE_DEVICES=''
export RAY_USAGE_STATS_ENABLED=0
export OMP_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 MKL_NUM_THREADS=1 NUMEXPR_NUM_THREADS=1
```

The client starts local Ray runtimes for preparation and postprocessing. It needs writable temporary storage. See the [resource notes](VALIDATION.md#host-resource-settings) if running in a container.

## 2. Install the client packages

Use a separate Python environment with Git and pip installed. Set `PARSE_CLIENT_DIR` to a new absolute directory. Install the matching Parse package without downloading the model weights:

```bash
python -m pip install -e "$CURATOR_DIR[interleaved_cpu,text_cpu,cv2]" 'httpx==0.28.1' 'ray==2.57.0'
hf download nvidia/Nemotron-3.5-Parse \
  --revision 62cb1e4ba153d113ff64aaf2be6e068c377cf53c \
  --exclude '*.safetensors' --local-dir "$PARSE_CLIENT_DIR"
python -m pip install --no-deps "$PARSE_CLIENT_DIR"
```

These commands have not yet been tested in a clean environment. Keep the client package and the server model on the same revision.

## 3. Submit one page

Put a PDF in `PDF_DIR`. Create a manifest file at `MANIFEST` containing one JSON line, replacing the filename with yours:

```json
{"file_name":"sample.pdf"}
```

Choose a new absolute `OUTPUT_DIR` that does not already exist. Export the following variables for the command below:

| Variable | Value |
| --- | --- |
| `CURATOR_DIR` | Your Curator checkout |
| `MANIFEST` | Path to the JSONL manifest |
| `PDF_DIR` | Directory containing your PDF |
| `OUTPUT_DIR` | New directory for results |
| `PARSE_CLIENT_DIR` | Matching Parse client package directory |
| `PARSE_MODEL_NAME` | Model name served by the endpoint |
| `BATCH_API_BASE` | Batch API URL, including `/v1` |
| `BATCH_HEADERS_FILE` | Private JSON file containing the operator's authentication headers |

The command processes one page. Its client deadline is ten minutes. If it times out, the remote job may still be running. Keep the job ID so you can check it before submitting again.

```bash
export HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1
python - <<'PYTHON'
import importlib.util
import json
import os
from pathlib import Path
import sys
import time
import httpx

source = Path(os.environ['CURATOR_DIR']) / 'tutorials/interleaved/nemotron_parse_pdf'
sys.path.insert(0, str(source))
spec = importlib.util.spec_from_file_location('parse_batch_preview', source / 'batch.py')
cli = importlib.util.module_from_spec(spec)
spec.loader.exec_module(cli)
sys.argv = [str(source / 'batch.py'),
    '--manifest', os.environ['MANIFEST'], '--pdf-dir', os.environ['PDF_DIR'],
    '--output-dir', os.environ['OUTPUT_DIR'], '--model-path', os.environ['PARSE_CLIENT_DIR'],
    '--inference-server-model-name', os.environ['PARSE_MODEL_NAME'],
    '--parse-mode', 'single', '--max-pdfs', '1', '--max-pages', '1',
    '--pdfs-per-task', '1', '--pages-per-job', '1',
    '--batch-api-base', os.environ['BATCH_API_BASE']]
headers = json.loads(Path(os.environ['BATCH_HEADERS_FILE']).read_text())
with httpx.Client(headers=headers, trust_env=False, timeout=180,
                  follow_redirects=False) as client:
    cli.main(client=client, absolute_deadline=time.monotonic() + 600)
PYTHON
```

Start with one page and inspect the output before increasing the page count. Large rendered images can exceed the client's memory or request limits. See [input and request limits](VALIDATION.md#input-and-request-limits) for details.

## 4. Check the output

Open the Parquet files and compare the text, tables and images with the PDF. Successful request counts do not guarantee accurate extraction.

| File | What to check |
| --- | --- |
| `batch_pilot_summary.json` | One successful output page, no missing pages or errors, and complete page accounting |
| `batch_request_accounting.json` | The response matches your submitted page, with no rejected response |
| `batch_job_events.jsonl` | The accepted job ID and its status history |
| Parquet files | The expected content from your PDF |

The client rereads Parquet before reporting success. It treats an empty parsed document or a token-limit response as a page failure.

After a disconnect, check the accepted job with the same authenticated tenant and download its results while they are retained. Rerunning the tutorial creates new jobs. A lost job-creation response needs operator investigation because the client may not have received the job ID.

## Deploy an endpoint

Endpoint operators can start from the example's `dynamo/examples/deployments/batch-gateway/profile-parse.yaml`. Replace its model, DGD, storage and credential placeholders, then follow the deployment instructions for the pinned Dynamo source. The profile does not start model workers.

## Inspect timings and known issues

See [validation and profiling notes](VALIDATION.md) for timing fields, resource settings and known output-quality issues.
