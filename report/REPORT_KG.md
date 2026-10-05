# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Thế Hưng  **MSSV:** 2A202602381  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    122.9
graph       196     34619     5642   0.00572    158.4

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       71   0.00010     5.19
graph       1.00   2.00     5172      163   0.00058     9.32
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00572 | +$0.00572 |
| Indexing giây | 122.9s | 158.4s | ×1.29 |
| Mỗi câu: USD | $0.00010 | $0.00058 | ×5.80 |
| Mỗi câu: giây | 5.19s | 9.32s | ×1.80 |
| Mỗi câu: in_tok | 696 | 5172 | ×7.43 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> - **Lúc Indexing:** Flat RAG chỉ gọi API Embedding (miễn phí trên gói Gemini, 176 chunks), trong khi GraphRAG phải gọi thêm 20 lần LLM Chat để trích xuất có cấu trúc (JSON) từ 20 bài báo tin tức (tiêu tốn 34.619 input tokens và 5.642 output tokens, chi phí $0.00572 USD).
> - **Lúc Querying:** GraphRAG sau khi đi multi-hop trên đồ thị tri thức đã bổ sung danh sách các sự kiện pháp lý (facts gồm Điều luật, các Khoản, khung hình phạt, tóm tắt vụ việc) vào prompt gửi cho LLM. Điều này khiến số token đầu vào tăng gấp 7.43 lần (từ 696 lên 5.172 tokens), làm chi phí mỗi câu tăng 5.80 lần ($0.00058 so với $0.00010) và độ trễ tăng 1.80 lần (9.32s so với 5.19s).

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều truy xuất chính xác định nghĩa tiền chất nằm gọn trong Điều 2 Luật Phòng chống ma túy 2021. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều tìm đúng 2 bị cáo Trần Thanh Tuấn và Trần Minh Tâm lãnh án tử hình trong bài báo về đường dây 36kg. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat RAG chỉ tìm được mức án 36 tháng tù trong bài báo nhưng không biết Điều luật; GraphRAG đi qua node `Crime` nối sang Điều 251 khoản 1 BLHS. |
| Q4 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat RAG biết tội tổ chức sử dụng nhưng không biết mức phạt tối đa; GraphRAG truy xuất được khoản 4 Điều 255 quy định tù 20 năm hoặc tù chung thân. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Flat RAG thiếu thông tin khoản áp dụng; GraphRAG liên kết từ Cái Quang Huy sang chất MDMA 9,6kg và đối chiếu khoản 4 Điều 250 (khung tử hình). |
| Q6 | aggregation | 0.00 / 1 | 1.00 / 2 | Graph | Flat RAG chỉ trả về các đoạn trích rời rạc thiếu tên vụ án; GraphRAG tổng hợp đầy đủ tất cả các vụ án và nhân vật liên quan đến MDMA từ đồ thị. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật đối với khung hình phạt tăng nặng / tối đa

- **Hiện tượng:** Nếu chỉ lấy khoản 1 (khung cơ bản) và các khoản nhắc đến tên chất ma túy của vụ án (như gợi ý ban đầu), câu trả lời cho các câu hỏi về mức phạt tối đa (như Q4 về Hoàng Nato phạm tội tổ chức sử dụng trái phép chất ma túy) sẽ bị thiếu khoản 4 của Điều 255.
- **Bằng chứng:** Truy vấn Cypher kiểm tra Điều 255:

```cypher
MATCH (a:Article {id: 'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
OPTIONAL MATCH (cl)-[:MENTIONS]->(s:Substance)
RETURN cl.number, cl.penalty, collect(s.name) AS substances ORDER BY cl.number;
```

```
cl.number | cl.penalty                                    | substances
1         | phạt tù từ 02 năm đến 07 năm                  | []
2         | phạt tù từ 07 năm đến 15 năm                  | []
3         | phạt tù từ 15 năm đến 20 năm                  | []
4         | phạt tù 20 năm hoặc tù chung thân             | []
5         | phạt tiền từ 50.000.000 đồng đến 500.000.00... | []
```

Khoản 4 của Điều 255 quy định mức phạt cao nhất là "20 năm hoặc tù chung thân", nhưng trong nội dung khoản không hề nhắc đến tên chất ma túy cụ thể mà phân loại theo hậu quả gây chết người hoặc tỷ lệ thương tật. Nếu hàm `context()` chỉ lọc `cl.number = 1` hoặc các khoản `MENTIONS` chất, khoản 4 sẽ bị bỏ sót hoàn toàn và LLM không thể trả lời đúng khung hình phạt tối đa.
- **Nguyên nhân:** Cấu trúc văn bản quy phạm pháp luật không chỉ phân hóa khung hình phạt theo chất và khối lượng chất (như Điều 249, 250, 251), mà còn theo tình tiết tăng nặng định khung độc lập với chất (như Điều 255: gây chết người, phạm tội nhiều lần).
- **Đề xuất sửa:** Trong câu truy vấn Cypher của `context()`, mở rộng điều kiện lọc khoản: lấy khoản 1, các khoản nhắc đến chất ma túy của vụ án, VÀ các khoản có chứa khung hình phạt cao nhất (`cl.penalty CONTAINS 'chung thân'` hoặc `cl.penalty CONTAINS 'tử hình'`). Đánh đổi: tăng thêm khoảng 100-200 token vào prompt nhưng đảm bảo tính bao quát cho mọi câu hỏi hỏi về mức phạt trần.

---

### Lỗi E3: Trùng thực thể (Entity Resolution / Entity Linking)

- **Hiện tượng:** Trong cơ sở dữ liệu đồ thị Neo4j, cùng một đối tượng ngoài đời thực lại bị tách thành các node `Person` hoặc `Case` riêng biệt do tên gọi trong các bài báo khác nhau.
- **Bằng chứng:** Truy vấn kiểm tra các node liên quan đến đối tượng "Hoàng Nato":

```cypher
MATCH (p:Person)
WHERE p.name CONTAINS 'Hoàng' OR p.name CONTAINS 'Tuấn' OR any(a IN coalesce(p.aliases, []) WHERE a CONTAINS 'Hoàng')
RETURN p.name AS name, p.aliases AS aliases;
```

```
name           | aliases
Dương Minh Tuấn| ["Hoàng Nato"]
Hoàng Nato     | []
```

Ở bài báo `news-100260920221957595.md`, đối tượng được nhắc với tên đầy đủ là `Dương Minh Tuấn` kèm biệt danh `"Hoàng Nato"`. Nhưng ở một bài báo khác, LLM lại trích xuất trực tiếp tên đối tượng là `Hoàng Nato` và `aliases = []`. Kết quả là sinh ra 2 node `Person` song song đại diện cho cùng một cá nhân ngoài đời thực.
- **Nguyên nhân:** Khâu nạp dữ liệu tin tức (`add_news_case`) thực hiện `MERGE (person:Person {name: p.name})` hoàn toàn dựa trên tên chuỗi do LLM tự do sinh ra. Hệ thống chưa có bước Entity Linking / Deduplication cho `Person` giống như đã làm với `Crime` qua hàm `link_entity`.
- **Đề xuất sửa:** Trước khi `MERGE` node `Person`, cần chuẩn hóa tên người: kiểm tra xem tên mới có trùng với bất kỳ `alias` nào của các `Person` đã có trong graph không (dùng Cypher: `OPTIONAL MATCH (existing:Person) WHERE existing.name = $name OR $name IN existing.aliases`). Nếu đã tồn tại, dùng node hiện có và cập nhật thêm alias thay vì tạo mới. Đánh đổi: tăng thời gian nạp graph lúc indexing do phải truy vấn đối chiếu trước mỗi lệnh merge.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> - **Khi nào Flat RAG là đủ:** Đối với các tác vụ hỏi đáp đơn nguồn (`single-hop`), nơi mà dữ kiện câu trả lời nằm trọn vẹn trong một đoạn văn bản cục bộ (như câu Q1 và Q2). Số liệu đo lường chứng minh: ở Q1 và Q2, Flat RAG đạt điểm tuyệt đối (`recall = 1.00`, `judge = 2`), độ trễ chỉ 1.22s (nhanh hơn GraphRAG 1.8 lần) và chi phí chỉ $0.00010/câu (rẻ hơn 5.8 lần). Dùng Knowledge Graph cho những tác vụ này là lãng phí và không đem lại giá trị gia tăng.
> - **Khi nào BẮT BUỘC dùng Knowledge Graph (GraphRAG):** Đối với các bài toán yêu cầu tổng hợp dữ liệu rải rác xuyên qua nhiều nguồn (`cross-kb` như Q3, Q4, Q5) hoặc truy vấn tổng hợp gom nhóm (`aggregation` như Q6). Số liệu cho thấy Flat RAG thất bại nặng nề ở các câu này với `recall` chỉ từ 0.00 đến 0.40 và `judge` chỉ đạt 1 (do vector search chỉ tìm được một phía của thông tin, ví dụ thấy mức án nhưng không thấy điều luật). Ngược lại, GraphRAG đạt độ chính xác hoàn hảo (`recall = 1.00`, `judge = 2.00`) trên toàn bộ các câu hỏi đa nguồn nhờ khả năng duyệt qua node cầu nối `Crime`.
> - **Kết luận triển khai:** Knowledge Graph chỉ thực sự "đáng tiền" khi hệ thống phải xử lý nghiệp vụ đòi hỏi suy luận liên kết nhiều bước (multi-hop) giữa các thực thể mà tìm kiếm ngữ nghĩa độc lập (semantic search) không thể kết nối được.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.07s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 18 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00053. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (bị cáo trong vụ án vận chuyển hơn 9,6kg MDMA qua sân bay Nội Bài).

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> - **Giới hạn tốc độ Gemini Free Tier (RateLimit 429):** Khi chạy `bench_kg.py --judge`, quá trình nhúng đồng thời 176 chunks và gọi liên tiếp 20 bài báo đã chạm ngưỡng 100 requests/phút (cho embedding) và 15 requests/phút (cho chat) của gói miễn phí Google Gemini API.
> - **Giải pháp đã xử lý:** Đã bổ sung cơ chế bắt lỗi `RateLimitError` và tự động backoff thử lại (`sleep 20s - 25s`) trong `src/llm.py`. Nhờ đó toàn bộ pipeline đã chạy tự động tới đích thành công mà không bị sập giữa chừng.
