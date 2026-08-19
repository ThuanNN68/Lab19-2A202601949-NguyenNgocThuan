# Báo cáo Lab 19 — Production-Grade GraphRAG vs Flat RAG

**Học viên:** Nguyễn Ngọc Thuận  
**Track:** AICB-K34 · Track 3: GraphRAG  
**Golden Set:** `data/golden_dataset.csv` — 50 câu, phạm vi 5.000 dòng đầu của nguồn dữ liệu.

## 1. Kết quả thực thi và kiểm tra bắt buộc

Graph Neo4j sau ingestion có **109 Entity nodes** và **134 edges**. Kiểm tra provenance trả về **0** cạnh thiếu `source_chunk_id` hoặc `published_date`. Hai file benchmark đã xuất là `outputs/graphrag_eval_results.csv` và `outputs/graphrag_vs_flatrag_summary.csv`.

Golden evaluation đã chạy đủ **50/50** câu: 23 `multi-hop`, 22 `cross-doc`, 5 `factoid`.

## 2. Thuyết minh kỹ thuật và failure modes

### 2.0 Near-Dedup (bonus)

Near-Dedup dùng **SimHash 64-bit + LSH 4 bands**, không quét cosine pairwise toàn bộ dataset. Một cặp chỉ được merge khi `Hamming distance <= 6` và `5-gram Jaccard >= 0.82`; bucket vượt 100 phần tử bị bỏ qua và log lại. Trên 5.000 source records, LSH sinh **1.171 candidate pairs**, trong đó **903** đạt ngưỡng merge; toàn bộ quyết định merge/reject và lý do nằm trong `outputs/near_dedup_audit.csv`. Các cặp merge cần được review bằng URL/đoạn văn bản trước khi dùng ở môi trường production để bắt false positive do bài tin có template giống nhau.

### 2.1 Coreference Resolution

Pipeline dùng conservative coreference guard: LLM chỉ đề xuất `mention → antecedent`; code chỉ thay thế khi antecedent có nguyên văn trong cùng chunk và mention xuất hiện duy nhất. Số tiền, ngày, phần trăm và ticker được kiểm tra bất biến. Các trường hợp không đạt được giữ nguyên và ghi vào `unresolved_mentions` với các mã như `ANTECEDENT_NOT_IN_SAME_CHUNK`, `AMBIGUOUS_MENTION_OCCURRENCES` và `INVARIANT_VIOLATION`.

Failure mode cần tránh là **false coreference → false edge**. Ví dụ, nếu “the company” bị gán cho công ty được nhắc ở câu trước thay vì chủ thể của giao dịch hiện tại, extractor có thể tạo sai cạnh `ACQUIRED` hoặc `INVESTED_IN`. Chính sách hiện tại ưu tiên bỏ sót một resolution hơn là đưa cạnh sai vào graph.

### 2.2 Entity Resolution: threshold và lexical guard

Ngưỡng vector matching là **cosine similarity = 0.90**. Candidate được tìm bằng HNSW ANN (chỉ dùng FlatIP cho tập rất nhỏ), sau đó qua guard theo loại entity:

- `Company`: chấp nhận alias/ticker trong allowlist hoặc khác biệt hậu tố Inc./Corp./Ltd.; lexical ratio phải từ 0.92.
- `Person`: chỉ merge exact-normalized; chặn các trường hợp cùng họ nhưng khác tên như `Sam Altman` và `Steve Altman`.
- `Technology`: non-exact merge bị chặn; bảo vệ `Apple` và `Apple Watch` không bị hợp nhất.

Audit ghi rõ `MERGE_MANUAL`, `MERGE_VECTOR`, `REJECT_GUARD`, `guard_reason` và nguồn candidate. Lần kiểm tra cuối có **20 candidate cùng type** từ graph thực tế; cả 20 đều là `REJECT_GUARD`, vì không cặp nào thỏa điều kiện alias chính xác/company-suffix để merge an toàn. File `outputs/entity_resolution_audit.csv` giữ lại toàn bộ các cặp này để review thủ công.

### 2.3 Super-node analysis

Top degree nodes thực tế:

| Hạng | Entity | Type | Degree |
|---:|---|---|---:|
| 1 | Apple | Company | 7 |
| 2 | Railergy | Company | 5 |
| 3 | Meeno | Company | 3 |

Node có degree cao nhất trong graph hiện tại là **Apple (7)**, nên cap 50 cạnh chưa được kích hoạt trên dữ liệu thật. `outputs/supernode_policy_check.csv` lưu cả live check và unit check nhánh policy (`degree=101 → limit=50`, PASS). Khi có super-node, policy lấy tối đa 50 cạnh mới nhất và toàn context không vượt 250 cạnh. Lợi ích là giảm token/context explosion; rủi ro là câu hỏi về sự kiện lịch sử có thể bị bỏ mất cạnh cũ.

### 2.4 Benchmark Flat RAG vs GraphRAG

Kết quả trung bình từ 50 dòng checkpoint:

| Metric | Flat RAG | GraphRAG | Nhận xét |
|---|---:|---:|---|
| Comprehensiveness | 1.66 | 1.64 | Cả hai thấp; GraphRAG chưa cải thiện overall. |
| Faithfulness | 1.80 | 1.78 | Context thường không chứa evidence Golden. |
| Multi-hop reasoning | 1.66 | 1.64 | Graph coverage chưa đủ cho chuỗi suy luận. |
| Latency (giây) | 1.89 | 1.84 | Gần tương đương trên sample này. |
| Token usage | 630.10 | 505.40 | Graph context ngắn hơn trong lần chạy này. |

Theo nhóm, GraphRAG tốt hơn nhẹ ở `cross-doc` (comprehensiveness 1.773 so với 1.727; faithfulness 1.955 so với 1.864) và `factoid`, nhưng thấp hơn Flat RAG ở `multi-hop`.

**Failure case chính — G5000-01:** cả Flat RAG lẫn GraphRAG trả lời rằng context không có Aeris–Ericsson, trong khi reference answer yêu cầu IoT Accelerator, Connected Vehicle Cloud, 100 triệu thiết bị, 9.000 doanh nghiệp và 190 quốc gia. Đây là bằng chứng pipeline đã benchmark với context thiếu evidence. Nguyên nhân gốc được suy luận là lần benchmark đã chạy trước khi Golden-aware selection được áp dụng đầy đủ, hoặc CSV input không khớp đúng phạm vi 5.000 dòng mà Golden Set tham chiếu.

**Failure case GraphRAG — G5000-09:** Flat RAG đạt comprehensiveness 2, GraphRAG đạt 1. Graph extraction/seed matching không đưa được event “Now Assist for Virtual Agent” vào subgraph, nên vector context tương đối ít bị ảnh hưởng hơn.

Khắc phục: chạy lại preprocessing với `source_row_id`, dùng `select_extraction_source(chunks_df, golden_scope_df)`, xác nhận các row ID evidence hiện diện trước extraction, và tăng/điều chỉnh relation allowlist cho các quan hệ Golden yêu cầu.

### 2.5 Trade-offs, Agent Control và Scale

Flat RAG có index đơn giản, nhanh triển khai nhưng thiếu liên kết đa tài liệu. GraphRAG tốn chi phí coreference, extraction, entity resolution và Neo4j ingestion; đổi lại có provenance và traversal có kiểm soát.

Đề xuất bị từ chối: pairwise cosine toàn bộ entity mentions `O(N²)`, vì sẽ tăng RAM/thời gian nhanh khi scale. Thay bằng FAISS HNSW ANN + lexical guard + Union-Find.

Khi scale 350 MB, bottleneck đầu tiên là LLM extraction và rate limit. Giải pháp: queue async theo batch, checkpoint từng batch, HNSW ANN, Neo4j `UNWIND` batch, partition/community summary và retry/backoff có resume.

## 3. Reflection và action plan

| Khái niệm | Module / code | Quan sát |
|---|---|---|
| Conservative Coreference | `run_coref()` + guard 1.7A | Bảo vệ graph khỏi false edge. |
| Schema allowlist | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Chặn relation/node ngoài schema. |
| Bulk Cypher | `bulk_insert_nodes()`, `bulk_insert_edges()` | Nạp theo `UNWIND`, không insert từng row. |
| Entity Resolution | `build_resolution_map()` + HNSW guard | Có audit cho merge/reject. |
| Super-node cap | `retrieve_graph_context()` | Cap degree >100, edge/global context cap. |
| LLM-as-a-Judge | `run_evaluation()` | So sánh quality, latency, token. |

Lỗi khó nhất là benchmark Golden không tìm thấy evidence mặc dù reference answer có nguồn cụ thể. Bài học là phải kiểm tra alignment giữa dataset, source row ID, chunk selection và Golden scope trước khi đánh giá retrieval.

Với đồ án thực tế, chỉ dùng GraphRAG khi câu hỏi thường cần nối quan hệ đa bước/đa tài liệu và provenance. Node có thể là `Document`, `Organization`, `Person`, `Technology`, `Policy`; relation có thể là `OWNS`, `ANNOUNCED`, `APPLIES_TO`, `SUPERSEDES`, `USES`. Entity Resolution cần allowlist và human-review queue cho candidate gần ngưỡng; super-node cần temporal cap và relation-aware ranking.

## 4. Việc cần xác nhận trước khi nộp

- [x] Neo4j connected; graph có node/edge.
- [x] Provenance check: 0 invalid edges.
- [x] Golden checkpoint: 50/50 câu.
- [x] Xuất hai CSV benchmark.
- [x] Đã lưu `entity_resolution_audit.csv`, `coref_audit.jsonl`, `near_dedup_audit.csv` và `supernode_policy_check.csv` làm bằng chứng kiểm tra.
- [ ] Rerun benchmark sau Golden-aware selection để cải thiện coverage và thay checkpoint hiện tại nếu dữ liệu đã align.
- [ ] Kiểm tra `Restart & Run All` (bỏ cell streaming nếu dùng CSV local).
