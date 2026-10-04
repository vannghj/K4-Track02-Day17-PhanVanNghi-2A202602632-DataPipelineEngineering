# B2 — Brainstorm: RAG từ kho PDF hợp đồng tiếng Việt

## 1. Bài toán và ràng buộc

**Người dùng:** phòng pháp chế và vận hành của một doanh nghiệp cỡ vừa, hỏi trợ lý nội bộ
những câu như *"Hợp đồng thuê kho với công ty X hết hạn khi nào, phạt chậm thanh toán bao
nhiêu?"*. Câu trả lời phải **trích dẫn đúng trang, đúng điều khoản**, vì người hỏi sẽ dùng
nó để ra quyết định có hệ quả pháp lý.

**Dữ liệu (giả định để thiết kế):** khoảng 50.000 PDF tích luỹ 10 năm, thêm vài trăm file
mỗi tuần.
- Khoảng 40% là bản **scan**: ảnh nghiêng, con dấu đỏ đè lên chữ, chữ ký.
- Bảng biểu nhiều cột (lịch thanh toán, phụ lục giá).
- Tiếng Việt có dấu, nhiều file cũ còn dùng font TCVN3/VNI, nên lớp text bị lỗi mã hoá.
- Có **phụ lục sửa đổi**: phụ lục số 3 thay điều 5.2 của hợp đồng gốc, nên câu trả lời đúng
  phụ thuộc vào *phiên bản hiệu lực*, không phải đoạn văn giống câu hỏi nhất.
- Có PII: số CCCD, số tài khoản, họ tên, địa chỉ người ký.

**Vì sao khó:** lỗi xảy ra *im lặng*. OCR đọc "phạt 0,5%" thành "phạt 5%" thì retrieval vẫn
chạy, model vẫn trả lời trôi chảy, và không check nào fail.

## 2. Kiến trúc

```
 SharePoint / email / máy scan
          │  (file mới, sửa, xoá)
          ▼
 BRONZE  raw PDF bất biến, key = sha256(file)   ── metadata: nguồn, thời điểm, người upload
          │
          ▼
 PARSE   router theo trang ──┬─ có text layer tốt ─▶ trích text + bảng (rẻ)
          (cache theo        └─ scan / font hỏng ──▶ OCR tiếng Việt (đắt)
           file hash +
           parser version)          │
          ▼                         ▼
 QUALITY GATE  điểm tin cậy OCR, tỉ lệ ký tự có dấu hợp lệ, bảng đủ cột ──▶ quarantine + hàng đợi review
          │
          ▼
 SILVER  documents (1 hàng / tài liệu, có khoá) · clauses (điều khoản + trang + bbox)
         · amendments (phụ lục X thay điều Y của hợp đồng Z) · PII đã che
          │
          ├─▶ GOLD chunks + embedding (cache theo hash(chunk) + model version) ─▶ vector index
          └─▶ GOLD contract_graph: hợp đồng ─ phụ lục ─ điều khoản ─ đối tác ─▶ bộ lọc "điều khoản hiệu lực"
                                         │
 QUERY  câu hỏi ─▶ tra đối tác/hợp đồng (graph) ─▶ vector search trong điều khoản hiệu lực ─▶ LLM + trích dẫn
```

## 3. Các câu hỏi then chốt

### Q1 — Nguồn và hình dạng: OCR toàn bộ hay route theo từng trang?

**Quyết định:** route **theo trang**. Trang có text layer sạch thì trích trực tiếp, trang scan
hoặc font hỏng mới đưa qua OCR. Kiểm tra text layer bằng tỉ lệ ký tự tiếng Việt hợp lệ:
text TCVN3 bị lỗi sẽ đầy ký tự lạ như "Hîp ®ång".

**Đánh đổi:** OCR toàn bộ thì đơn giản, đồng nhất, nhưng tốn gấp khoảng 2,5 lần và còn làm
*xấu đi* các trang vốn có text chuẩn. Route theo trang thì rẻ hơn và chính xác hơn, cái giá là
thêm một nhánh logic và một ngưỡng phải hiệu chỉnh. Chọn route, vì chi phí OCR là khoản lớn
nhất của pipeline (xem Q3).

### Q3 — Cái gì vỡ khi scale: chi phí parse và chạy lại

**Quyết định:** cache kết quả parse theo `sha256(file) + parser_version`, giống
`embedding_cache` theo `hash + model_version` trong lab. Đổi model OCR thì tăng
`parser_version` và **chủ động** parse lại, có ước tính chi phí trước, như bonus B1.

**Đánh đổi:** cache tốn dung lượng lưu trữ và phải quản lý version. Không cache thì mỗi lần
sửa bug ở bước chunking lại phải OCR lại 50.000 file, mất nhiều ngày và tốn tiền. Ở quy mô
10 lần, bottleneck đầu tiên không phải vector DB mà là OCR và **người review quarantine**.
Vì vậy phải đo tỉ lệ quarantine ngay từ đầu.

### Q4 — Hợp đồng và chất lượng: chặn gì trước khi vào index?

**Quyết định:** quality gate ở biên Parse → Silver:
- Điểm tin cậy OCR trung bình trên trang không dưới ngưỡng.
- Số và tỉ lệ phần trăm trong bảng phải parse được.
- Mỗi hợp đồng phải tìm thấy đối tác và ngày hiệu lực.

Trang không đạt thì vào `quarantine` kèm lý do, **không chặn cả lô**, giống
`quarantine_events` trong lab. Tỉ lệ quarantine tăng vọt sau khi đổi parser là tín hiệu để
cảnh báo.

**Đánh đổi:** ngưỡng chặt thì index sạch nhưng thiếu tài liệu, người dùng hỏi sẽ nhận "không
tìm thấy". Ngưỡng lỏng thì đủ tài liệu nhưng có rủi ro trích sai con số. Với dữ liệu pháp lý,
chọn **chặt**: trả lời "không có trong tài liệu đã kiểm duyệt" tốt hơn trả lời sai một cách tự
tin. Các trang bị quarantine vẫn tìm được bằng full-text search, có gắn nhãn "chưa kiểm duyệt".

### Q6 — Vector RAG hay knowledge graph?

**Quyết định:** **hybrid nhẹ**. Vector search để tìm nội dung điều khoản. Thêm một graph nhỏ
có cấu trúc rõ (hợp đồng → phụ lục → điều khoản bị thay thế → đối tác) để **lọc** ra điều
khoản đang hiệu lực *trước khi* tìm theo vector.

**Đánh đổi:** chỉ dùng vector thì đơn giản, nhưng sẽ trả về điều 5.2 *gốc* vì nó giống câu hỏi
hơn bản sửa trong phụ lục. Đây đúng là lỗi "trạng thái cũ thắng trạng thái mới" của
`silver_tickets` trong lab. Graph tốn công trích quan hệ "phụ lục thay điều khoản nào", nhưng
quan hệ này ít và có mẫu câu ổn định ("Sửa đổi Điều 5.2 như sau"), nên trích bằng rule cộng
LLM có kiểm duyệt là khả thi.

### Q8 — Failure semantics: chạy lại, sửa, xoá

**Quyết định:**
- Mọi bước dùng khoá là content hash, nên chạy lại một ngày không sinh chunk trùng.
- File bị sửa có hash mới. Tài liệu cũ chuyển thành tombstone ở Silver (`is_deleted`), chunk
  của nó bị gỡ khỏi index.
- Yêu cầu xoá (hợp đồng huỷ, hoặc chủ thể dữ liệu yêu cầu xoá) phải lan tới vector index và
  cache, đúng như T-97 trong lab.

**Đánh đổi:** tombstone giữ khoá mãi, đổi lại không thể vô tình "hồi sinh" tài liệu khi
backfill. Xoá hẳn thì gọn hơn nhưng mất khả năng audit. Chọn tombstone, nhưng **xoá nội
dung** (text, embedding) và chỉ giữ hash cùng metadata.

### Q10 — Bối cảnh Việt Nam

**Quyết định:**
- Che CCCD, số tài khoản và số điện thoại bằng regex ngay ở Silver. Họ tên dùng NER tiếng Việt.
- Dữ liệu và model OCR/embedding chạy trong hạ tầng đặt tại Việt Nam, hoặc nhà cung cấp có
  cam kết xử lý dữ liệu, để tuân thủ Nghị định 13/2023/NĐ-CP về bảo vệ dữ liệu cá nhân.
- Chuẩn hoá Unicode (NFC) trước khi hash và embed. Cùng chữ "hợp đồng" có thể được gõ ở dạng
  NFC hoặc NFD; nếu không chuẩn hoá thì cache miss và kết quả retrieval lệch nhau.

**Đánh đổi:** gửi PDF tới API OCR/LLM quốc tế thì chất lượng cao hơn và triển khai nhanh hơn,
nhưng có rủi ro pháp lý khi chuyển dữ liệu cá nhân ra nước ngoài. Chọn xử lý trong nước cho
bước có PII thô (OCR), chỉ gửi văn bản *đã che PII* ra ngoài khi cần.

## 4. Phương án bị loại

**Bỏ pipeline, đưa thẳng PDF vào LLM context dài mỗi lần hỏi.** Phương án này hấp dẫn vì
không cần parse hay index. Mình loại vì:
1. Chi phí và độ trễ nhân theo *số câu hỏi × số trang*, thay vì trả một lần lúc ingest.
2. Không có quality gate. OCR sai thì không ai biết.
3. Trích dẫn không tái lập được: hai lần hỏi giống nhau có thể ra hai câu trả lời khác nhau.
4. Không có chỗ nào để thực thi "xoá phải lan".

**Cũng bị loại: GraphRAG đầy đủ** (LLM trích mọi thực thể và quan hệ, dựng community
summary). Với câu hỏi chủ yếu là tra cứu điều khoản, chi phí token để dựng graph lớn hơn nhiều
lợi ích. Graph nhỏ chỉ chứa quan hệ phụ lục và đối tác đã đủ cho multi-hop thực sự cần thiết.

## 5. Bước tiếp theo nếu làm thật

Lấy mẫu 200 trang có nhãn tay (gồm cả trang scan và trang TCVN3) để đo character error rate
theo từng nhánh router, rồi chọn ngưỡng quality gate *từ số đo*, giống cách lab chọn
`LOOKBACK_DAYS` từ P99 thay vì đoán.
