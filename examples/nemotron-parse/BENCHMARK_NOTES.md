# Parse and Batch benchmark notes

## Parse comparison

The same 512-page input took 240.130 seconds through Direct and 260.389 seconds through Batch. Batch took 8.44% longer in this fixed-order comparison.

Both paths called the same Dynamo frontend and Parse vLLM worker. Direct sent HTTP requests with concurrency 64 and used four collectors. Batch submitted eight 64-page jobs to llm-d Batch Gateway inside Dynamo, using synchronous dispatch, 16 workers, global and endpoint concurrency 64, and four collectors.

Each path returned six token-limit failures. The [validation notes](curator/tutorials/interleaved/nemotron_parse_pdf/VALIDATION.md) describe the output-quality checks and later experiments.

## Recorded timing milestones

Each timestamp is seconds after that arm's caller phase began. The rows are checkpoints, not separate stages. Requests, serving and collection overlap.

| Milestone | Direct | Batch |
| --- | ---: | ---: |
| First model HTTP request or first input upload | 2.423 | 5.086 |
| All requests offered or all eight jobs created | 212.544 | 11.040 |
| Last model HTTP response or last output download | 239.939 | 254.583 |
| Last non-length page written durably | 225.721 | 259.333 |
| Caller phase complete | 240.130 | 260.389 |

Batch downloaded the first completed job at 170.163 seconds. Its last download finished at 254.583 seconds, then the last non-length page became durable 4.750 seconds later. The uploaded JSONL contained 447,964,976 bytes.

These timestamps do not explain the full 20.258-second difference. Direct excluded a final artifact check that Batch included. We need the same final readback checks on both paths before attributing the gap to Batch overhead.

## What to profile next

Use a matched input and the same completion checks in both arms. The client now records PDF preparation, upload, job creation, status requests, polling sleeps, downloads, replay, postprocessing and Parquet rereading. It also records local Ray startup and shutdown.

The server needs correlated job IDs, request IDs and trace IDs to separate queue wait, serving admission, prefill, decode and result publication. Client polling time cannot distinguish these. The historical run had pprof and database OTel tracing disabled.

The [profile field reference](curator/tutorials/interleaved/nemotron_parse_pdf/VALIDATION.md#read-the-caller-profile) explains which intervals overlap. Keep failed and incomplete runs in the results. New timing fields are instrumentation, not evidence that a bottleneck has been found or fixed.

## Earlier usable results in the warm-Ray comparison

Here, a usable result group means completed records that passed structural and request-association checks, were written durably and were read back by the caller. Downstream work can consume that group while other jobs finish. There is no priority tag, and those checks do not establish semantic accuracy.

Batch collected completed jobs independently. Ray used HTTP actors, Ray blocks, postprocessing and `iter_rows`, then accumulated 1,024 records before writing through the same durable writer. Both paths received the same controlled ordering across 2,057 contexts. The experiment did not test automatic grouping or retrieval from inside a running Batch job.

Batch delivered groups earlier in that workflow. Much of its full-completion advantage occurred before the first measured request. Ray spent an additional 17.35 seconds and 15.95 seconds there in the two pairs. From the first request to final readback, Ray was 1.77 seconds faster in one pair and 7.16 seconds slower in the other. A later delay before Ray's second group reached the writer remains unexplained.

This supports earlier caller delivery for the tested workflow. It does not establish a consistent GPU inference-throughput advantage, and further Ray tuning may change the result.

## Source evidence

The detailed benchmark brief retains `parse-workers16-consolidated-audit-20261113.json`, `warm-ray-customer-scrutiny-20261113.json` and `warm-ray-completed-study-20261113.json`. They contain the recorded measurements, but no complete distributed trace or independently reproduced customer environment.
