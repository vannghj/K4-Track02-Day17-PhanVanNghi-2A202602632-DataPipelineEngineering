# K4-Track02-Day17 — Report cá nhân

Phần phân tích tối đa một trang, không tính output ở phần 5.
Định dạng tham chiếu và phạm vi tính trang: [SUBMISSION.md](../docs/SUBMISSION.md).

**Họ tên / MSSV:** Phan Van Nghi / 2A202602632
**Repo:** https://github.com/vannghj/K4-Track02-Day17-PhanVanNghi-2A202602632-DataPipelineEngineering
**Commit bài nộp:** `17672a2` (3 fix: `546fe0f` silver, `97d1148` config, `a141c80` staging)
**AI đã dùng và phạm vi hỗ trợ (hoặc không dùng):** Claude Code (Claude Opus 5.5): đọc code, chạy baseline, đề xuất 3 bản sửa và soạn nháp REPORT; tôi đã review và giải thích được từng dòng thay đổi.
**Nguồn tham khảo khác (nếu có):**

## 1. Ba lỗi

| | Lỗi Silver | Lỗi late data | Lỗi xoá (CDC) |
|---|---|---|---|
| **Triệu chứng** | verify: `24 rows for 12 tickets`, T-91 có 3 hàng; checksum `gold_doc_chunks` đổi sau mỗi lần rerun | verify: feature ≠ full recompute; u05 ngày 08-12 = (2, 0) thay vì (5, 1); rerun lệch fresh build rồi đứng yên | T-97 trong Silver `is_deleted=False`, còn tên/subject; còn trong snapshot `v2026-08-16` và 2 chunk RAG |
| **Nguyên nhân gốc** | Dedup trong batch đúng, nhưng ghi Silver bằng `INSERT`: không có khoá giữa các batch | `LOOKBACK_DAYS = 0` do đoán; event 08-12 đến ngày 08-15 không được tính lại vào ngày 08-12 | `ticket_id` chỉ lấy từ `after`; delete có `after = null` → khoá NULL → bị lọc mất |
| **Cách sửa** | `silver.py`: `MERGE ON ticket_id`, update khi `s._lsn > t._lsn` | `config.py`: `LOOKBACK_DAYS = 3` | `staging.py`: `coalesce(after.ticket_id, before.ticket_id)` |
| **Khái niệm** | Silver có khoá; MERGE + LSN guard | Event time ≠ ingest time; lookback = ceil(P99) | CDC log-based; "Xoá phải lan" |

## 2. Các con số

- Lateness đo từ Bronze: P50 `0.00`, P95 `2.90`, P99 `3.00` ngày → `LOOKBACK_DAYS = 3`
- `checksums.txt`: **PASS**, Gold `39e115c510ecdf526800eac227158a4f` (C0 = C1 = C2 = C3)
- `make parity`: **PARITY**. Trước khi sửa: verify 8/18, rerun FAIL; sau khi sửa: verify 18/18, pytest 34 passed, dbt PASS=19

## 3. Lựa chọn kỹ thuật

- **MERGE vs overwrite-partition:** ticket là thực thể, thay đổi có thể đến bất kỳ ngày nào nên cần MERGE theo khoá; feature là aggregate theo ngày, tính lại được từ Silver nên ghi đè cửa sổ `[day-3, day]` là đủ.
- **Tombstone thay vì xoá hẳn:** giữ khoá và `_lsn` của delete để replay 08-12 không hồi sinh T-97, và để downstream thấy `is_deleted`; đánh đổi là hàng tồn mãi.
- **Snapshot as-of từ Bronze:** tái lập được dữ liệu đã train; có thay đổi thì sinh version mới, không sửa bản cũ.
- **DuckDB/dbt, không Spark:** khoảng 80 bản ghi, chạy dưới 2 giây trên một máy; Spark chỉ thêm chi phí cluster.

## 4. Hai câu hỏi suy ngẫm

1. **Snapshot bất biến vs quyền xoá:** quyền xoá thắng. Dựng lại các snapshot chứa T-97 thành version mới đã loại ticket, retire bản cũ, ghi erasure log để audit. Dài hạn: mã hoá text theo khoá từng user (crypto-shredding); xoá khoá là mọi bản cũ không đọc được.
2. **Chốt PII cho tên:** đặt ở Bronze → Silver, cạnh `mask_pii`, dùng NER tiếng Việt thay tên bằng `<NAME>`; thêm contract ở Gold. Đo bằng recall trên ~200 ticket gán nhãn tay và canary (tên giả cài vào seed test, assert không lọt tới Gold).

## 5. Output (dán nguyên văn)

```text
$ make verify
=== verify.py — Day 17 pipeline contracts ===
  [OK ] Bronze  every daily batch landed as Parquet (7 days x 3 sources)
  [OK ] Bronze  re-landing a batch is a no-op (append-only, no duplicate file)
  [OK ] Bronze  Bronze keeps the raw truth: Kafka tombstone + redelivered events are still there
  [OK ] Silver  silver_tickets has exactly one row per ticket_id
  [OK ] Silver  T-91 shows its latest state: high / closed / bug
  [OK ] Silver  deleted ticket T-97 is a tombstone: is_deleted and no personal data left
  [OK ] Silver  no email / phone number survives past Bronze
  [OK ] Silver  silver_events has one row per event_id (Kafka redeliveries removed)
  [OK ] Silver  2 malformed events quarantined with a reason; the run did not halt
  [OK ] Gold    gold_feature_daily reconciles with a full recompute from Silver
  [OK ] Gold    u05's offline events of 08-12 (arrived 08-15) are counted on 08-12
  [OK ] Gold    LOOKBACK_DAYS covers measured P99 lateness (p99=3.00 days)
  [OK ] Gold    training set uses point-in-time priority (T-91 created as 'low')
  [OK ] Gold    late feedback creates a NEW snapshot version; the old one is untouched
  [OK ] Gold    latest training snapshot excludes the deleted ticket T-97
  [OK ] Gold    deletes propagate to the RAG index: no chunk of T-97
  [OK ] Gold    gold_doc_chunks: one row per chunk, and a re-run embeds 0 new chunks
  [OK ] Rerun   re-run 2026-08-12 three times -> Gold checksum identical to a fresh build

RESULT: 18/18 checks — ALL PASS
re-run checksums written to submission/checksums.txt

$ make test
..................................                                       [100%]
34 passed in 1.06s

$ make rerun3
# Lab 17 — re-run check for 2026-08-12

run                     gold_feature_daily    gold_training_set     gold_doc_chunks       gold (combined)
fresh build             8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #1 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #2 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #3 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f

RESULT: PASS — 3 re-runs, identical checksums

$ make lateness
event lateness over 43 Bronze records (calendar days): p50=0.00 p95=2.90 p99=3.00 max=3
-> lookback must be >= ceil(p99) = 3 day(s); config.LOOKBACK_DAYS = 3

$ make dbt
cd dbt_project && DBT_PROFILES_DIR=. /Users/vannghj/Documents/AIthucchien/K4-Track02-Day17-Data-Pipeline-Engineering/.venv/bin/dbt build --event-time-start 2026-08-10 --event-time-end 2026-08-17
07:10:47  Running with dbt=1.12.5
07:10:47  Registered adapter: duckdb=1.11.0
07:10:47  Unable to do partial parsing because saved manifest not found. Starting full parse.
07:10:48  Found 5 models, 13 data tests, 2 sources, 502 macros, 1 unit test
07:10:48  
07:10:48  Concurrency: 1 threads (target='dev')
07:10:48  
07:10:48  1 of 19 START sql view model main.stg_events ................................... [RUN]
07:10:48  1 of 19 OK created sql view model main.stg_events .............................. [OK in 0.04s]
07:10:48  2 of 19 START sql view model main.stg_ticket_changes ........................... [RUN]
07:10:48  2 of 19 OK created sql view model main.stg_ticket_changes ...................... [OK in 0.01s]
07:10:48  3 of 19 START sql incremental model main.silver_events ......................... [RUN]
07:10:48  3 of 19 OK created sql incremental model main.silver_events .................... [OK in 0.05s]
07:10:48  4 of 19 START unit_test silver_tickets::silver_tickets_latest_change_wins_and_delete_is_tombstone  [RUN]
07:10:48  4 of 19 PASS silver_tickets::silver_tickets_latest_change_wins_and_delete_is_tombstone  [PASS in 0.12s]
07:10:48  8 of 19 START sql incremental model main.silver_tickets ........................ [RUN]
07:10:48  8 of 19 OK created sql incremental model main.silver_tickets ................... [OK in 0.04s]
07:10:48  5 of 19 START test not_null_silver_events_event_id ............................. [RUN]
07:10:48  5 of 19 PASS not_null_silver_events_event_id ................................... [PASS in 0.02s]
07:10:48  6 of 19 START test not_null_silver_events_user_id .............................. [RUN]
07:10:48  6 of 19 PASS not_null_silver_events_user_id .................................... [PASS in 0.01s]
07:10:48  7 of 19 START test unique_silver_events_event_id ............................... [RUN]
07:10:48  7 of 19 PASS unique_silver_events_event_id ..................................... [PASS in 0.01s]
07:10:48  9 of 19 START test accepted_values_silver_tickets_category__bug__billing__other  [RUN]
07:10:48  9 of 19 PASS accepted_values_silver_tickets_category__bug__billing__other ...... [PASS in 0.01s]
07:10:48  10 of 19 START test accepted_values_silver_tickets_priority__low__medium__high . [RUN]
07:10:49  10 of 19 PASS accepted_values_silver_tickets_priority__low__medium__high ....... [PASS in 0.01s]
07:10:49  11 of 19 START test accepted_values_silver_tickets_status__open__pending__closed  [RUN]
07:10:49  11 of 19 PASS accepted_values_silver_tickets_status__open__pending__closed ..... [PASS in 0.01s]
07:10:49  12 of 19 START test not_null_silver_tickets__lsn ............................... [RUN]
07:10:49  12 of 19 PASS not_null_silver_tickets__lsn ..................................... [PASS in 0.01s]
07:10:49  13 of 19 START test not_null_silver_tickets_is_deleted ......................... [RUN]
07:10:49  13 of 19 PASS not_null_silver_tickets_is_deleted ............................... [PASS in 0.01s]
07:10:49  14 of 19 START test not_null_silver_tickets_ticket_id .......................... [RUN]
07:10:49  14 of 19 PASS not_null_silver_tickets_ticket_id ................................ [PASS in 0.01s]
07:10:49  15 of 19 START test unique_silver_tickets_ticket_id ............................ [RUN]
07:10:49  15 of 19 PASS unique_silver_tickets_ticket_id .................................. [PASS in 0.01s]
07:10:49  16 of 19 START sql microbatch model main.gold_feature_daily .................... [RUN]
07:10:49  Batch 1 of 7 START batch 2026-08-10 of main.gold_feature_daily ....................... [RUN]
07:10:49  Batch 1 of 7 OK created batch 2026-08-10 of main.gold_feature_daily .................. [OK in 0.01s]
07:10:49  Batch 2 of 7 START batch 2026-08-11 of main.gold_feature_daily ....................... [RUN]
07:10:49  Batch 2 of 7 OK created batch 2026-08-11 of main.gold_feature_daily .................. [OK in 0.03s]
07:10:49  Batch 3 of 7 START batch 2026-08-12 of main.gold_feature_daily ....................... [RUN]
07:10:49  Batch 3 of 7 OK created batch 2026-08-12 of main.gold_feature_daily .................. [OK in 0.01s]
07:10:49  Batch 4 of 7 START batch 2026-08-13 of main.gold_feature_daily ....................... [RUN]
07:10:49  Batch 4 of 7 OK created batch 2026-08-13 of main.gold_feature_daily .................. [OK in 0.02s]
07:10:49  Batch 5 of 7 START batch 2026-08-14 of main.gold_feature_daily ....................... [RUN]
07:10:49  Batch 5 of 7 OK created batch 2026-08-14 of main.gold_feature_daily .................. [OK in 0.02s]
07:10:49  Batch 6 of 7 START batch 2026-08-15 of main.gold_feature_daily ....................... [RUN]
07:10:49  Batch 6 of 7 OK created batch 2026-08-15 of main.gold_feature_daily .................. [OK in 0.01s]
07:10:49  Batch 7 of 7 START batch 2026-08-16 of main.gold_feature_daily ....................... [RUN]
07:10:49  Batch 7 of 7 OK created batch 2026-08-16 of main.gold_feature_daily .................. [OK in 0.01s]
07:10:49  16 of 19 OK created sql microbatch model main.gold_feature_daily ............... [SUCCESS in 0.13s]
07:10:49  17 of 19 START test dbt_utils_free_unique_combination_gold_feature_daily_user_id__event_date  [RUN]
07:10:49  17 of 19 PASS dbt_utils_free_unique_combination_gold_feature_daily_user_id__event_date  [PASS in 0.01s]
07:10:49  18 of 19 START test not_null_gold_feature_daily_event_date ..................... [RUN]
07:10:49  18 of 19 PASS not_null_gold_feature_daily_event_date ........................... [PASS in 0.01s]
07:10:49  19 of 19 START test not_null_gold_feature_daily_user_id ........................ [RUN]
07:10:49  19 of 19 PASS not_null_gold_feature_daily_user_id .............................. [PASS in 0.01s]
07:10:49  
07:10:49  Finished running 3 incremental models, 13 data tests, 1 unit test, 2 view models in 0 hours 0 minutes and 0.63 seconds (0.63s).
07:10:49  
07:10:49  Completed successfully
07:10:49  
07:10:49  Done. PASS=19 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=19

$ make parity
=== parity: lite pipeline vs dbt ===
  [OK ] silver_tickets       lite 3c15dfd43701  dbt 3c15dfd43701
  [OK ] gold_feature_daily   lite 8630e04a61d1  dbt 8630e04a61d1
RESULT: PARITY — both implementations agree
```

Nếu dùng PowerShell, ghi lệnh tương đương và output thực tế theo [SUBMISSION.md](../docs/SUBMISSION.md).
Nếu làm bonus, thêm output B1 hoặc đường dẫn bằng chứng B2 ở cuối phần này.

## Bonus B1 — Bước LLM có cache (`pipeline/llm_label.py`)

- Khoá cache = `sha256(text)` + `model` + `prompt_version`; cache lưu **câu trả lời thô** (kể cả sai schema) nên chạy lại tốn 0 lần gọi, còn đổi prompt thì gắn nhãn lại toàn bộ có chủ đích.
- Ước tính chi phí (token × giá) chỉ cho các cache miss, **trước** khi gọi model.
- Câu trả lời không parse được về `bug`/`billing`/`other` → `llm_label_quarantine` (có lý do), không vào Gold. Cả hai bảng dựng lại từ cache mỗi lần chạy → idempotent; mỗi hàng mang `model` + `prompt_version`.

```text
$ make bonus-llm
=== bonus: LLM labelling of 11 live tickets ===
  cost estimate before running: ~484 tokens = $0.0010 per full run
  [OK ] first run labels every live ticket
  [OK ] re-run with same model + prompt makes 0 LLM calls
  [OK ] every Gold label is bug / billing / other
  [OK ] off-schema answers go to llm_label_quarantine
  [OK ] new prompt version re-labels on purpose
  [OK ] labels carry their prompt version
BONUS PASS
```

## Bonus B2 — Brainstorm

Chọn hướng brainstorm (không làm Airflow): [`bonus/DESIGN.md`](../bonus/DESIGN.md) — RAG từ kho PDF hợp đồng tiếng Việt: 6 câu hỏi then chốt (router OCR theo trang, cache parse theo hash + version, quality gate + quarantine, hybrid vector + graph phụ lục, tombstone và xoá phải lan, bối cảnh Việt Nam), 2 phương án bị loại, sơ đồ kiến trúc.
