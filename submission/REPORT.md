# K4-Track02-Day17 — Report cá nhân

Phần phân tích tối đa một trang, không tính output ở phần 5.
Định dạng tham chiếu và phạm vi tính trang: [SUBMISSION.md](../docs/SUBMISSION.md).

**Họ tên / MSSV:** Phan Van Nghi / 2A202602632
**Repo:** https://github.com/vannghj/K4-Track02-Day17-PhanVanNghi-2A202602632-DataPipelineEngineering
**Commit bài nộp:**
**AI đã dùng và phạm vi hỗ trợ (hoặc không dùng):** Claude Code (Claude Opus 5.5): đọc code, chạy baseline, đề xuất 3 bản sửa và soạn nháp REPORT; tôi đã review và giải thích được từng dòng thay đổi.
**Nguồn tham khảo khác (nếu có):**

## 1. Ba lỗi

Mỗi lỗi 4 dòng. Triệu chứng = thứ bạn *thấy* đầu tiên (check nào fail, số nào lạ,
checksum nào lệch) — không phải cách sửa.

| | Lỗi Silver | Lỗi late data | Lỗi xoá (CDC) |
|---|---|---|---|
| **Triệu chứng** | verify: `silver_tickets` **24 rows for 12 tickets**; T-91 có 3 hàng (`low/open`, `high/open`, `high/closed/bug`); `gold_doc_chunks` 22 rows / 9 chunks; rerun3: checksum `gold_doc_chunks` đổi sau **mỗi** lần chạy lại (b4915… → e8996… → f30d9… → c10ba…) | verify: `gold_feature_daily` ≠ full recompute (`c50b8851affe != 8630e04a61d1`); u05 ngày 08-12 = **(2, 0)** thay vì (5, 1); `LOOKBACK_DAYS=0 < 3`; rerun3: `gold_feature_daily` fresh `c50b8851` → rerun `21d1035e` rồi đứng yên ("ổn định sai") | verify: T-97 trong Silver **không phải tombstone** (`is_deleted=False`, còn `user_id`, `subject`, body có tên "Nguyễn Văn An"); T-97 vẫn còn 1 hàng trong snapshot `v2026-08-16` và 2 chunk trong RAG index |
| **Nguyên nhân gốc** | `upsert_silver_tickets` dedup *trong* batch đúng (QUALIFY theo `_lsn`) nhưng ghi sang Silver bằng `INSERT` → không có khoá *giữa các* batch: mỗi batch thêm hàng, replay batch cũ thêm lại trạng thái cũ | `LOOKBACK_DAYS = 0` dựa trên giả định "event tới trong vài giây": run 08-15 chỉ tính lại partition 08-15, nên event `event_time` 08-12 của u05 (ingest 08-15) không bao giờ vào ngày 08-12 trong fresh build | `ticket_changes_sql` lấy `ticket_id` chỉ từ `after`; Debezium `op='d'` có `after = null` → `ticket_id` NULL → bị `WHERE ticket_id IS NOT NULL` lọc mất. Delete không tới Silver, SCD2, snapshot |
| **Cách sửa** (file, vài dòng) | `pipeline/silver.py`: `INSERT` → `MERGE INTO silver_tickets ON ticket_id`, `WHEN MATCHED AND s._lsn > t._lsn THEN UPDATE`, `WHEN NOT MATCHED THEN INSERT` | `pipeline/config.py`: `LOOKBACK_DAYS = 3` = ceil(P99). `build_feature_daily` đã overwrite-partition `[day-3, day]` theo event date | `pipeline/staging.py`: `coalesce(after.ticket_id, before.ticket_id)`. Cột khác vẫn từ `after` (null) → MERGE ghi tombstone `is_deleted=true`, PII null, giữ `_lsn=24020000`; Kafka tombstone (`value=null`) vẫn bị bỏ qua |
| **Khái niệm trên slide** | Silver — có khoá; idempotent = MERGE theo khoá + LSN guard (thay đổi mới hơn thắng) | Data về muộn: event time ≠ ingest time; lookback = ceil(P99) đo từ Bronze; overwrite-partition | CDC log-based (`before`/`after`/`op`/`lsn`; delete ≠ Kafka tombstone); "Xoá phải lan" |

## 2. Các con số

- Lateness đo từ Bronze (43 event, `make lateness`): P50 = `0.00`, P95 = `2.90`, P99 = `3.00` ngày, max = 3
- P99 lateness đo từ Bronze: `3.00` ngày → `LOOKBACK_DAYS = 3` (ceil(3.00); trùng dbt `lookback=3`). Kiểm tra: u05 ngày 08-12 = 5 events, 3 clicks, 1 down
- Baseline (chưa sửa): verify `8/18`, pytest `9 failed, 25 passed`, rerun3 `FAIL`
- `submission/checksums.txt`: **PASS** — Gold checksum: `39e115c510ecdf526800eac227158a4f` (C0 = C1 = C2 = C3)
- Sau khi sửa: verify `18/18 ALL PASS`, pytest `34 passed`, dbt `PASS=19`
- `make parity`: **PARITY** (`silver_tickets` 3c15dfd43701, `gold_feature_daily` 8630e04a61d1)

## 3. Lựa chọn công cụ / kỹ thuật (mỗi dòng một câu "vì sao")

- MERGE theo khoá cho `silver_tickets`, overwrite-partition cho `gold_feature_daily`: `silver_tickets` là bảng thực thể, một thay đổi có thể chạm hàng của bất kỳ ngày nào → cần MERGE theo `ticket_id` + LSN guard; `gold_feature_daily` là aggregate theo partition `event_date`, tính lại hoàn toàn được từ Silver → DELETE + INSERT cửa sổ `[day-3, day]` đơn giản, xác định, tự idempotent.
- Tombstone thay vì xoá hẳn hàng trong Silver: hàng giữ khoá + `_lsn` của delete làm mốc, nên replay batch 08-12 (LSN 24011000) không thể hồi sinh T-97; downstream đọc `is_deleted` để lan xoá; PII vẫn bị xoá ngay. Đánh đổi: hàng tồn mãi — có thể dọn sau khi hết cửa sổ replay.
- Snapshot training dựng lại từ Bronze "as of" ngày đó, không sửa snapshot cũ: model phải tái lập được trên đúng dữ liệu đã train; Bronze bất biến nên rebuild luôn ra cùng kết quả, có thay đổi thì sinh version mới (feedback muộn của T-88 → `v2026-08-15`), checksum guard chặn ghi đè.
- DuckDB (lite) / dbt (track dbt) cho bài toán cỡ này, chứ không phải Spark: ~80 bản ghi / 7 ngày chạy một máy dưới 2 giây; Spark chỉ thêm cluster/JVM. dbt cho merge/microbatch/contract/test dạng khai báo, cùng SQL chuyển được sang warehouse lớn khi cần.

## 4. Hai câu hỏi suy ngẫm

1. Snapshot `v2026-08-12`..`v2026-08-14` vẫn chứa văn bản của T-97 (đã bị xoá ngày
   08-15). "Snapshot bất biến" và "quyền được xoá dữ liệu" mâu thuẫn — bạn xử lý thế nào?

   Quyền xoá thắng: "bất biến" nghĩa là không ai *âm thầm* sửa snapshot, chứ không phải giữ PII
   mãi. Ngắn hạn: khi có yêu cầu xoá, dựng lại mọi snapshot chứa ticket đó thành version mới
   (vd. `v2026-08-12-r1`) đã loại T-97, retire bản cũ, ghi erasure log (`ticket_id`, LSN, ngày)
   để audit; model train trên bản cũ được đánh giá/train lại theo chu kỳ. Dài hạn: snapshot
   không chứa PII thô mà mã hoá text theo khoá từng user (crypto-shredding) — xoá khoá là mọi
   bản cũ, cả Bronze, không còn đọc được mà không phải sửa file bất biến nào.
2. Regex che được email và số điện thoại, nhưng tên "Nguyễn Văn An" vẫn còn. Bạn sẽ
   đặt chốt PII nào, ở tầng nào, và đo nó ra sao?

   Chốt ở biên Bronze → Silver, cùng chỗ `mask_pii`, vì mọi bảng downstream chỉ đọc Silver.
   Thêm bộ nhận dạng NER tiếng Việt (vd. Presidio + recognizer tuỳ chỉnh) cho PERSON/ADDRESS
   → `<NAME>`; tên thật chỉ lấy qua join `user_id` khi được phép. Ở Gold thêm contract như check
   "no email/phone survives" nhưng cho tên. Đo: tập ~200 ticket gán nhãn tay → recall/precision
   theo loại PII (ưu tiên recall); canary — cài tên giả biết trước vào seed test và assert không
   lọt tới Gold; theo dõi tỉ lệ thay thế mỗi batch để bắt drift.

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
