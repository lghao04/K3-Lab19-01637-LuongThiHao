# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Lường Thị Hảo
**Khóa học:** 3
**Ngày thực hiện:** [19/08/2026]  

---

## PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** chunk `76879c031f77eb7392eb::c0000` — text gốc: *"Innovative Industrial has consistently operated at fairly low leverage financing most of its acquisitions with equit..."*
- **Hiện tượng:** Trong quá trình resolve, cơ chế coreference đã làm rớt mất mã ticker **`IIPR`** vốn xuất hiện trong text gốc (thường ở dạng "Innovative Industrial (IIPR)") — resolved_text không còn giữ mention này. Đây được bộ validate fact-preservation (regex bắt ticker) tự động phát hiện qua `coref_fact_flags = [tickers_missing:['IIPR']]`, thuộc nhóm chỉ 2/400 chunk (0.5%) bị flag trên toàn bộ tập chạy.
- **Hậu quả đối với Knowledge Graph:** Nếu không có bước validate và safety-net revert, việc mất ticker `IIPR` trong evidence text có thể khiến Triple Extraction (Section 2) không nhận diện được entity Company qua cách gọi tên bằng ticker — nếu ở chunk khác trong cùng bài báo entity này được nhắc lại bằng "IIPR" thay vì tên đầy đủ, Entity Resolution sẽ khó gộp 2 mention thành 1 node duy nhất, dẫn đến **graph bị phân mảnh (duplicate/orphan entity)** thay vì một Company node thống nhất. Đây chính là failure mode "false coreference → false/missing edge" được cảnh báo trong đề bài.
- **Cách xử lý đã áp dụng:** Thay vì cố sửa thủ công, pipeline áp dụng nguyên tắc conservative — **revert chunk bị flag về text gốc**, chấp nhận mất phần pronoun chưa được resolve còn hơn giữ lại 1 bản resolved có khả năng sai fact.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao ($> 0.85$) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.90` (tham số mặc định trong `build_resolution_map(raw_triples_df, threshold=0.90, top_k=5)`) — dùng embedding `sentence-transformers/all-MiniLM-L6-v2` qua FAISS `IndexFlatIP` (inner product trên vector đã normalize = cosine similarity).
- **Cặp thực thể bị Guard chặn:** `X90A` vs `X90` (type: Technology, similarity = **0.9206**, decision = `REJECT_GUARD`)
- **Lý do chặn:** Hai chuỗi có vector embedding gần giống nhau (0.92 — cao hơn ngưỡng merge thuần vector) vì tên gọi chỉ khác 1 ký tự hậu tố, nhưng về mặt ngữ nghĩa `X90A` và `X90` là **hai phiên bản sản phẩm khác nhau** (model variant), không phải cùng một thực thể. Lexical Guard bắt được sự khác biệt ký tự cuối (suffix mismatch) mà similarity vector thuần túy bỏ sót — đây là bằng chứng guard hoạt động đúng, ngăn hiện tượng "entity collapsing" giữa các phiên bản sản phẩm gần giống tên.
- **Ghi chú về audit set:** Tổng cộng chỉ có 2 candidate pairs được đưa vào audit (1 `MERGE_VECTOR`, 1 `REJECT_GUARD`) trên toàn bộ graph 203 edges — con số khá nhỏ, nhiều khả năng do `EXTRACTION_MAX_CHUNKS=400` giới hạn phạm vi trích xuất triple nên entity pool chưa đủ lớn để sinh nhiều candidate trùng lặp.

---

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy $N$ cạnh ($N=50$) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Top 3 Super-nodes:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|------|--------------|---------------------|----------------------|
| 1 | ServiceNow | Company | 8 |
| 2 | Infineon | Company | 5 |
| 3 | Sysdig Inc. | Company | 5 |

> `test_supernode_policy()` trong notebook chỉ lấy **top 1** (`LIMIT 1`). Để lấy top 3, chạy Cypher sau:
> ```cypher
> MATCH (n:Entity)-[r]-()
> WITH n, count(r) AS degree
> ORDER BY degree DESC LIMIT 3
> RETURN n.id, n.name, n.entity_type, degree
> ```

> **Kết quả thực tế:** ServiceNow (`Company`) có degree 8; Infineon (`Company`) và Sysdig Inc. (`Company`) cùng có degree 5. Trong schema Neo4j của lab, loại thực thể được lưu ở property `entity_type` (không phải `type`).

- **Nhận xét quan trọng:** Với dataset lab (1500 bài báo, scale guard), degree cao nhất chỉ đạt **8** (ServiceNow) — thấp hơn nhiều so với ngưỡng `SUPER_NODE_DEGREE=100`. Nghĩa là cơ chế cap **chưa từng được kích hoạt thực tế** trong lần chạy này; đây là kết quả của quy mô dữ liệu nhỏ (1500 articles), không phải bằng chứng cơ chế cap hoạt động sai.
- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - *Ưu điểm:* Khi một node thật sự trở thành super-node (degree > 100) ở dataset lớn hơn, việc chỉ lấy 50 cạnh mới nhất theo `published_date` giúp tránh context explosion khi truy vấn (không kéo hết hàng trăm cạnh vào prompt), đồng thời ưu tiên thông tin gần đây — phù hợp với domain tin tức công nghệ nơi thông tin cũ dễ lỗi thời.
  - *Rủi ro:* Nếu câu hỏi liên quan đến sự kiện lịch sử/nền tảng của công ty (ví dụ: "công ty được thành lập khi nào", "ai là nhà đầu tư đầu tiên") mà cạnh chứa thông tin đó có `published_date` cũ, cạnh này có thể bị cắt khỏi top-50 mới nhất, dẫn đến GraphRAG trả lời thiếu hoặc sai dù thông tin vẫn tồn tại trong graph.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|-------------------|----------|----------|--------------------------|-------------------|
| **Comprehensiveness (1–5)** | 2.20 | 2.20 | 0.00 | Hòa tuyệt đối trên cả 10/10 câu — không có câu nào 2 hệ thống chấm khác nhau |
| **Faithfulness (1–5)** | 2.60 | 2.20 | **−0.40 (Flat thắng)** | Toàn bộ chênh lệch đến từ **duy nhất 1 câu** (G5000-39) — xem phân tích ca lỗi bên dưới |
| **Multi-hop Reasoning (1–5)** | 2.20 | 2.20 | 0.00 | Hòa tuyệt đối trên cả 10/10 câu |
| **Latency trung bình (s)** | 2.836 | 2.718 | −0.118 (Graph nhanh hơn nhẹ) | Chênh lệch nhỏ, không đáng kể ở scale dữ liệu này |
| **Token usage trung bình** | 801.7 | 803.4 | +1.7 (Graph nhỉnh hơn không đáng kể) | GraphRAG không hề tốn token hơn Flat RAG trong sample này, dù có bước extraction phụ |

**Breakdown theo loại câu hỏi** (từ `graphrag_vs_flatrag_summary.csv`):
- **factoid** (1 câu): cả hai đạt tuyệt đối 5/5/5 — câu hỏi đơn giản, cả 2 hệ thống đều trả lời đúng và trích dẫn đúng chunk (`ServiceNow, NVIDIA, Accenture` — AI Lighthouse).
- **cross-doc** (4 câu): cả hai đều ở mức thấp (1.0/1.0/1.0) — cả Flat lẫn Graph đều **không tìm ra thông tin liên quan**, trả lời "I'm sorry, không có thông tin" cho toàn bộ 4 câu cross-doc.
- **multi-hop** (5 câu): đây là nhóm duy nhất có chênh lệch — faithfulness Flat 3.4 vs Graph 2.6, hoàn toàn do 1 câu G5000-39 kéo điểm graph xuống.

> **Nhận xét quan trọng:** Tổng cộng **4/10 câu** cả Flat RAG lẫn GraphRAG đều trả lời "không tìm thấy thông tin" — tỷ lệ refusal cao ở cả hai hệ thống cho thấy vấn đề nằm nhiều hơn ở **retrieval coverage** (dữ liệu/index chưa phủ đủ) hơn là ở phương pháp truy xuất. GraphRAG không hề vượt trội Flat RAG ở bất kỳ tiêu chí nào trên 10 câu golden query, và cũng không tốn token/latency đáng kể hơn — kết quả trung lập, chưa đủ bằng chứng ủng hộ giả thuyết "GraphRAG tốt hơn cho multi-hop" ở scale lab này (203 edges).

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi Flat RAG thất bại (GraphRAG thành công):**
   - *Không tìm thấy case nào như vậy trong 10 câu golden query.* Trên toàn bộ dataset, GraphRAG **không thắng Flat RAG ở bất kỳ câu nào** — mọi câu đều hòa điểm hoặc Flat RAG bằng/hơn Graph. Đây là một phát hiện cần nêu thẳng thắn thay vì cố ép ra 1 ví dụ không có thật.
2. **Ca lỗi GraphRAG thất bại (Q&A ID: G5000-39, nhóm multi-hop):**
   - *Câu hỏi:* "What two strategic capability areas did HPE expand in 2023 through the Axis Security deal and its later AI cloud announcement?"
   - *Điểm số:* `flat_faithfulness=5` vs `graph_faithfulness=1` (chênh lệch −4, toàn bộ delta của cả bảng đến từ câu này)
   - *Nguyên nhân thật sự:* Cả **Flat RAG lẫn GraphRAG đều trả lời giống hệt nhau về nội dung** — cả hai đều nói "không có thông tin về HPE/Axis Security trong context được cung cấp" (retrieval thất bại như nhau ở cả 2 hệ thống, khả năng do bài báo về HPE Axis Security không nằm trong tập 1500 bài / 400 chunk được extract). Điều bất thường là **LLM Judge chấm 2 câu trả lời gần như giống hệt nhau theo 2 cách khác nhau**: với Flat RAG, judge coi việc "thừa nhận không có thông tin" là *trung thực với context* → faithfulness 5; với GraphRAG, judge lại coi cùng một kiểu câu trả lời là *mâu thuẫn với reference answer* → faithfulness 1. Đây là bằng chứng cho **sự thiếu nhất quán (inconsistency) của LLM-as-a-Judge** khi chấm các câu trả lời dạng "từ chối trả lời" — không phải bằng chứng GraphRAG kém hơn về mặt kỹ thuật.
   - *Đề xuất khắc phục:* (1) Chuẩn hoá rubric chấm điểm cho riêng trường hợp "refusal" — quy định rõ khi nào một câu từ chối được coi là "faithful" để tránh judge tự do diễn giải khác nhau giữa 2 lần chấm; (2) tăng `EXTRACTION_MAX_CHUNKS` hoặc kiểm tra xem bài báo liên quan đến HPE/Axis Security có nằm trong `LAB_MAX_ARTICLES=1500` hay không — nếu bị loại khỏi tập mẫu thì đây là vấn đề coverage ở tầng dữ liệu, không phải ở retrieval logic.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:** 
> - So sánh sự đánh đổi giữa GraphRAG vs Flat RAG về Latency, Token và Indexing Overhead.
> - Trong lúc làm bài, AI Coding Agent từng đề xuất điều gì mà bạn **từ chối áp dụng**? Tại sao?
> - Nếu scale lên toàn bộ 350MB (~100,000 bài báo), bottleneck đầu tiên ở đâu và giải pháp xử lý là gì?

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:** Ở scale lab (1500 bài, 203 edges), GraphRAG có **Indexing Overhead cao hơn hẳn** Flat RAG — phải chạy thêm NER+RE extraction qua LLM (tốn token + thời gian), Entity Resolution, và bulk insert Neo4j — trong khi Flat RAG chỉ cần embed + FAISS index. Đổi lại, latency truy vấn của GraphRAG lại **nhỉnh hơn Flat RAG một chút** (2.72s vs 2.84s) trong lần chạy này, nhưng chất lượng câu trả lời (faithfulness) lại thấp hơn — nghĩa là ở quy mô nhỏ này, chi phí indexing bỏ ra của GraphRAG **chưa đem lại lợi ích quality tương xứng**. Trade-off chỉ có khả năng nghiêng về GraphRAG khi graph đủ dày (nhiều multi-hop relationship thật sự tồn tại giữa các thực thể).
- **Quyết định từ chối AI Coding Agent:**
  1. Khi liên tục gặp lỗi 429 (rate limit) trong lúc chạy Triple Extraction, AI Coding Agent đề xuất chạy hết 50 batch trong 1 lượt (`batch_size` mặc định × toàn bộ `extraction_source`). Đã **từ chối** đề xuất này và chọn chạy theo lô nhỏ hơn (10 batch/lần) để kiểm soát rủi ro mất dữ liệu khi 1 batch fail giữa chừng, đồng thời dễ theo dõi tiến độ token/quota còn lại trước khi tiếp tục — thay vì đặt cược cả 1 lượt lớn vào 1 lần gọi có khả năng cao dính rate limit.
  2. Khi Groq liên tục báo lỗi 429 do hết quota TPD, agent gợi ý cân nhắc đổi sang gọi trực tiếp OpenAI API (`OPENAI_API_KEY`) cho bước extraction thay vì Groq. Đã **giữ nguyên Groq** (chỉ đổi model trong cùng hệ sinh thái Groq, từ `openai/gpt-oss-120b` sang `openai/gpt-oss-20b`) để không phá vỡ kiến trúc pipeline đã thiết kế — theo đúng phân công vai trò ban đầu của lab (`OPENAI_API_KEY` chỉ dành riêng cho LLM-as-a-Judge, không dùng cho extraction), tránh làm lệch phạm vi so sánh chi phí/latency giữa các phần của hệ thống.
- **Giải pháp scale 350MB (~100,000 bài báo):** Bottleneck đầu tiên nhiều khả năng là **triple extraction qua LLM** (NER+RE) — hiện `EXTRACTION_MAX_CHUNKS=400` đã là giới hạn scale-guard ở quy mô nhỏ; ở 100K bài, số lượng chunk cần extract sẽ tăng theo cấp số nhân, dẫn đến rate-limit và chi phí token bùng nổ nếu chạy tuần tự. Hướng xử lý: batch + async request với worker queue (giới hạn concurrency theo rate limit của provider), cache kết quả extraction theo `chunk_id` để tránh chạy lại, và cân nhắc dùng model rẻ/nhanh hơn cho extraction (giữ model mạnh cho generation/judge). Về phía Neo4j, cần chuyển từ `UNWIND` đơn giản sang batch theo lô cố định (vd. 5000 rows/lần) để tránh transaction quá lớn.

---

## PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Chạy ổn định trên 1500 bài, nhưng vì đây là bước gọi LLM (Groq) theo batch nên cũng là nơi bị dính lỗi 429 nhiều nhất trong pipeline — cùng vấn đề rate-limit mô tả ở mục Debugging bên dưới. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Allowlist giữ graph gọn (chỉ 203 edges trên 400 chunk được extract) — cho thấy guard đã lọc bỏ khá nhiều triple không khớp schema thay vì insert bừa mọi thứ LLM trả về. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND` bulk insert chạy nhanh hơn hẳn insert từng dòng; edge provenance đạt 100% (203/203 edges có đủ `source_chunk_id` + `published_date`) — chứng tỏ pipeline gắn provenance đúng ngay tại bước insert chứ không phải patch thêm sau. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Chỉ sinh ra 2 candidate pairs trên toàn bộ graph — khá ít, nhiều khả năng do `EXTRACTION_MAX_CHUNKS=400` giới hạn entity pool. Nhưng chất lượng guard tốt: cặp `X90A`/`X90` similarity 0.92 vẫn bị `REJECT_GUARD` đúng như kỳ vọng, cho thấy Union-Find + Lexical Guard không merge nhầm chỉ vì vector gần nhau. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()` | Top-degree node (ServiceNow) chỉ đạt degree 8, chưa chạm ngưỡng `SUPER_NODE_DEGREE=100` — cơ chế cap được implement đúng nhưng chưa có dịp trigger thực tế ở scale dữ liệu lab (1500 bài). Cần dataset lớn hơn để kiểm chứng cap hoạt động khi có node degree thật sự cao. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()` | Trên 10 câu golden query, GraphRAG không vượt Flat RAG ở comprehensiveness/multi-hop (hòa) và thua ở faithfulness (2.2 vs 2.6) — cho thấy với graph còn thưa (203 edges), retrieval qua vector đơn thuần vẫn bám nguồn tốt hơn traversal qua đồ thị chưa đủ dày. |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất gặp phải:** Trong quá trình chạy Section 2 (Triple Extraction) và Section 1 (Coreference Resolution) — cả hai đều gọi Groq API theo batch trên nhiều chunk liên tiếp — gặp lỗi **429 (rate limit exceeded)** lặp lại nhiều lần. Đây là lỗi khó xử lý hơn lỗi 404 model deprecation (gặp trước đó) vì 429 không phải lỗi cố định: request đôi khi thành công, đôi khi bị từ chối tùy tốc độ gửi request và quota còn lại tại thời điểm gọi, nên khó tái hiện ổn định để debug.
- **Cách xử lý — nhiều lớp, theo đúng thứ tự phát hiện vấn đề:**
  1. **Retry với exponential backoff + jitter** (baseline sẵn có): `time.sleep(min(20, 2**attempt + random.random()))`, tối đa 4 lần thử. Đủ cho lỗi mạng thoáng qua, nhưng **không đủ** khi Groq báo phải chờ hàng phút (thấy rõ khi extraction với `openai/gpt-oss-120b` dính 89/100 batch lỗi 429, TPD đã dùng 199,059/200,000 — retry ngắn không cứu được vì bản chất là hết quota ngày, không phải lỗi tạm thời).
  2. **Rate-limit-aware retry**: vá `groq_chat()` để parse trực tiếp thời gian chờ Groq trả về trong message lỗi (`"try again in Xm Ys"`), tự `sleep` đúng thời gian đó nếu ≤90s; nếu Groq yêu cầu chờ lâu hơn, coi như quota gần cạn và raise ngay thay vì lãng phí thời gian retry vô ích.
  3. **Đổi model để có quota riêng**: chuyển từ `openai/gpt-oss-120b` sang `openai/gpt-oss-20b` (Groq tính TPD theo từng model độc lập) — giúp unblock ngay lập tức thay vì chờ reset theo ngày. Có thêm cột `model_used` vào mỗi triple để audit khi dữ liệu được extract bởi nhiều model khác nhau trong cùng 1 lần chạy.
  4. **Chống crash khi model trả sai schema JSON**: gặp `AttributeError: 'str' object has no attribute 'get'` giữa chừng vì có batch trả `items` chứa string thay vì dict lồng nhau — nguyên nhân do vòng lặp parse response nằm ngoài `try/except`, khiến 1 batch lỗi làm sập cả hàm và **mất luôn các batch đã chạy thành công trước đó** (không kịp `return` DataFrame). Đã vá bằng cách bọc toàn bộ phần parse `items`/`relations` vào `try/except` riêng với `isinstance()` type-check ở từng cấp, log batch lỗi vào `dropped_relations_df` thay vì crash.
  5. **Phát hiện & vá nguyên nhân gốc của lỗi schema JSON**: qua breakdown `dropped_relations_df["reason"]` thấy các giá trị `malformed_item_type` chứa mảnh JSON rời rạc (`""`, `"relations"`, `":"`, list thô) — dấu hiệu **reasoning content của GPT-OSS models bị lẫn vào phần JSON output**, làm hỏng cấu trúc lồng nhau. Vá bằng cách thêm `include_reasoning: False` vào request khi gọi model dòng `gpt-oss`.
  6. **Resume có chọn lọc thay vì chạy lại từ đầu**: viết `resume_failed_extraction()` chỉ gọi lại đúng các batch bị lỗi (theo `start` index lưu trong `extraction_errors_df`), tránh tốn quota vào các batch đã thành công — quan trọng khi quota hàng ngày là tài nguyên giới hạn cứng.
- **Bài học:** Lỗi 429 khó debug hơn lỗi 404 (model deprecated) gặp trước đó vì không cố định — cùng 1 đoạn code có lúc chạy được, có lúc không, tùy tốc độ gửi request và quota còn lại tại thời điểm gọi. Ngoài ra, một lỗi tưởng như đơn giản (rate limit) khi kết hợp với thiết kế try/except không đủ chặt (bug 4) có thể khuếch đại hậu quả — mất dữ liệu đã xử lý thay vì chỉ mất 1 batch — cho thấy tầm quan trọng của việc **cô lập lỗi ở đúng cấp độ** (per-batch, không phải per-run) trong pipeline gọi LLM theo batch.

---

## TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG |3| |
| Khả năng kiểm soát AI Coding Agent |5| |
| Chất lượng đồ thị tri thức xây dựng |5| |
| Khả năng phân tích và debug hệ thống |5| |