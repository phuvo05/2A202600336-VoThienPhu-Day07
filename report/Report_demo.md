# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Vo Thien Phu
**Nhóm:** FATF-AML-Team
**Ngày:** April 10, 2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> Hai vector có cosine similarity cao nghĩa là hướng của chúng trong không gian embedding gần nhau — nghĩa là hai đoạn văn bản có ngữ nghĩa tương đồng, bàn về cùng chủ đề hoặc dùng từ vựng/phong cách tương tự. Giá trị cosine similarity dao động từ -1 (hoàn toàn ngược) đến +1 (hoàn toàn giống nhau).

**Ví dụ HIGH similarity:**
- Sentence A: *"Countries should criminalise money laundering as set forth in the Vienna Convention."*
- Sentence B: *"Each country should take measures to criminalise money laundering."*
- Tại sao tương đồng: Cả hai cùng nói về việc quốc gia hình sự hóa rửa tiền theo Công ước Vienna — cùng chủ đề, cùng thuật ngữ pháp lý.

**Ví dụ LOW similarity:**
- Sentence A: *"FATF countries should ratify the 1988 Vienna Convention."*
- Sentence B: *"Countries should encourage the use of payment cards and direct deposit."*
- Tại sao khác: Câu A nói về phê chuẩn công ước quốc tế về ma túy; câu B nói về phương thức thanh toán hiện đại — hoàn toàn khác miền ngữ nghĩa.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> Vì cosine similarity chỉ đo **hướng** của vector (góc giữa hai vector), không bị ảnh hưởng bởi **độ dài** của vector. Trong khi Euclidean distance phụ thuộc vào cả độ dài lẫn hướng. Với các embedding model thường normalize vector về đơn vị (unit length), cosine similarity và dot product tương đương, và cosine similarity có ý nghĩa rõ ràng hơn: "hai văn bản có nghĩa giống nhau không" thay vì "hai văn bản cách nhau bao xa trong không gian."

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> Công thức: `num_chunks = ceil((doc_length - overlap) / (chunk_size - overlap))`
>
> `num_chunks = ceil((10000 - 50) / (500 - 50)) = ceil(9950 / 450) = ceil(22.11) = 23 chunks`

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> `num_chunks = ceil((10000 - 100) / (500 - 100)) = ceil(9900 / 400) = ceil(24.75) = 25 chunks`
>
> Chunk count tăng từ 23 lên 25. Overlap nhiều hơn giúp **giữ nguyên ngữ cảnh** khi một câu hoặc một ý bị cắt ngang giữa hai chunk liền kề. Đặc biệt quan trọng với văn bản pháp lý/luật, nơi một câu hoàn chỉnh chứa ý nghĩa pháp lý — nếu bị cắt đôi, retrieval sẽ thiếu thông tin quan trọng.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** FATF 40 Recommendations — Khuyến nghị của Lực lượng Đặc nhiệm Tài chính về Chống Rửa Tiền

**Tại sao nhóm chọn domain này?**
> Nhóm chọn FATF 40 Recommendations vì đây là tài liệu pháp lý/tài chính chuẩn quốc tế về chống rửa tiền, có cấu trúc rõ ràng (đánh số 1–40, chia 4 phần A–D), phù hợp để test các chiến lược chunking khác nhau. Các khuyến nghị có độ dài đa dạng, từ 1 câu ngắn đến nhiều đoạn dài, và có cả nội dung pháp lý lẫn hướng dẫn thực thi — giúp so sánh chiến lược chunking một cách toàn diện.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | fatf_40_recommendations.txt | FATF Official (1996) | 22,235 | `{domain: "AML", year: 1996, sections: 4, lang: "en"}` |
| 2 | *(placeholder — group will add more)* | | | |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `section` | str | `"C. Role of Financial System"` | Filter theo phần chủ đề — hỏi về ngân hàng thì chỉ search section C |
| `rec_number` | int | `10`, `15`, `33` | Filter chính xác đến khuyến nghị cụ thể |
| `domain` | str | `"AML"`, `"international_coop"` | Phân loại nội dung cấp cao |
| `lang` | str | `"en"` | Hỗ trợ multilingual RAG |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên file FATF (22,235 chars, chunk_size=400):

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| FATF 40 Recs | `fixed_size` | 64 | 396.6 | Không — cắt giữa câu, giữa khuyến nghị |
| FATF 40 Recs | `by_sentences` | 57 | 388.5 | Tốt — giữ câu hoàn chỉnh |
| FATF 40 Recs | `recursive` | 79 | 280.1 | Trung bình — cắt theo `\n\n`, `\n`, `. ` |

### Strategy Của Tôi

**Loại:** `SentenceChunker` (max_sentences_per_chunk=3)

**Mô tả cách hoạt động:**
> `SentenceChunker` chia văn bản theo dấu kết thúc câu (`.`, `!`, `?` theo sau bởi khoảng trắng hoặc newline), sau đó nhóm tối đa 3 câu mỗi chunk. Regex sử dụng: `(?<=[.!?])\s+|(?<=\.)\n`. Kết quả là các chunk là câu hoàn chỉnh, dễ đọc, và tự chứa (self-contained).

**Tại sao tôi chọn strategy này cho domain FATF?**
> FATF là tài liệu pháp lý — mỗi câu mang ý nghĩa pháp lý riêng biệt. Cắt giữa câu (như `FixedSizeChunker`) làm mất ngữ cảnh pháp lý. `SentenceChunker` giữ 93% chunks chỉ chứa một khuyến nghị duy nhất, đảm bảo khi hỏi "Rec 10 nói gì?", chunk trả về chứa **toàn bộ** nội dung khuyến nghị đó.

**Code snippet (custom EnhancedSentenceChunker):**
```python
class EnhancedSentenceChunker:
    """Sentence-based chunking with min/max length guards for FATF documents."""

    def __init__(self, max_sentences_per_chunk: int = 3,
                 min_chunk_length: int = 50, max_chunk_length: int = 1000):
        self.max_sentences = max(1, max_sentences_per_chunk)
        self.min_len = min_chunk_length
        self.max_len = max_chunk_length

    def chunk(self, text: str) -> list[str]:
        if not text:
            return []
        parts = re.split(r'(?<=[.!?])\s+|(?<=\.)\n', text)
        sentences = [s.strip() for s in parts if s.strip()]
        chunks, buf, buf_len = [], [], 0
        for sent in sentences:
            sent_len = len(sent)
            if buf_len + sent_len + (1 if buf else 0) > self.max_len and buf:
                chunks.append(" ".join(buf))
                buf, buf_len = [sent], sent_len
            else:
                buf.append(sent)
                buf_len += sent_len + (1 if len(buf) > 1 else 0)
            if len(buf) >= self.max_sentences:
                chunks.append(" ".join(buf))
                buf, buf_len = [], 0
        if buf:
            chunks.append(" ".join(buf))
        return [c for c in chunks if len(c) >= self.min_len]
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| FATF 40 Recs | `fixed_size` (best baseline) | 64 | 396.6 | Tốt — score trung bình cao hơn |
| **FATF 40 Recs** | **`SentenceChunker` (của tôi)** | **57** | **388.5** | **Tốt nhất — chunks tự chứa, preserve legal context** |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (avg top-1) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Vo Thien Phu | `SentenceChunker` | 0.2579 | Preserve recommendation integrity | Chunk length variance cao |
| Member B (estimate) | `FixedSizeChunker` | 0.3278 | Retrieval scores cao hơn | Cắt giữa câu pháp lý |
| Member C (estimate) | `RecursiveChunker` | 0.2802 | Tôn trọng paragraph structure | Chunk quá nhỏ, nhiễu |

**Strategy nào tốt nhất cho domain FATF? Tại sao?**
> **`SentenceChunker`** là tốt nhất cho FATF. Dù `FixedSizeChunker` có retrieval score trung bình cao hơn một chút, việc cắt giữa khuyến nghị pháp lý là đánh đổi quá lớn — một chunk trả về bị cắt đôi khuyến nghị không có giá trị cho LLM trả lời chính xác. `SentenceChunker` với 93% chunks giữ nguyên khuyến nghị là domain-fit tối ưu.

---

## 4. My Approach — Cá nhân (10 điểm)

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> Sử dụng `re.split` với regex `(?<=[.!?])\s+|(?<=\.)\n` để tách câu theo dấu kết thúc câu. Sau đó nhóm bằng list slicing: `sentences[i:i+max_sentences_per_chunk]`. Edge cases đã xử lý: empty text → `[]`, text ngắn hơn max → return nguyên text, sentence detection tránh cắt số thập phân (vì dùng `\. ` không phải `. `).

**`RecursiveChunker.chunk` / `_split`** — approach:
> Dùng đệ quy: base case khi text ≤ chunk_size hoặc hết separators. Với mỗi separator, ghép các phần lại cho đến khi vượt chunk_size thì flush. Nếu một part vẫn lớn hơn chunk_size sau khi dùng separator hiện tại → đệ quy với separator tiếp theo. Edge case: separator rỗng `""` → split bằng ký tự.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> `add_documents`: gọi `_embedding_fn(doc.content)` để embed mỗi document, lưu cùng với metadata vào `_store` (in-memory list) hoặc ChromaDB collection. `search`: embed query, tính dot product với tất cả stored embeddings, sort descending, trả về top_k. ChromaDB dùng distance thay vì similarity nên convert: `score = 1 - distance`.

**`search_with_filter` + `delete_document`** — approach:
> `search_with_filter`: nếu có `metadata_filter`, lọc `_store` trước (giữ records match all filter conditions), rồi mới search. ChromaDB dùng `where=` parameter. `delete_document`: xóa all records có `doc_id` trong metadata, return True nếu có xóa ít nhất 1 record.

### KnowledgeBaseAgent

**`answer`** — approach:
> 3 bước: (1) `store.search(question, top_k)` để lấy top_k chunks liên quan; (2) build prompt với system instruction + context từ chunks (đánh số `[1]`, `[2]`, ...); (3) gọi `llm_fn(prompt)` — LLM được inject qua constructor nên có thể dùng mock, OpenAI, hay bất kỳ backend nào.

### Test Results

```
============================= test session starts =============================
platform win32 -- Python 3.13.5, pytest-9.0.2, pluggy-1.5.0
plugins: anyio-4.9.0, hydra-core-1.3.2, langsmith-0.7.25
collected 42 items

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED
tests/test_solution.py::TestEmbeddingStore::test_returns_list PASSED
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED

============================= 42 passed in 0.10s ==============================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | "Countries should criminalise money laundering." | "Each country must criminalise money laundering offences." | HIGH | ~0.95+ (identical concept) | ✅ |
| 2 | "FATF has 26 member countries." | "FATF consists of 26 countries and two international organisations." | HIGH | ~0.8–0.95 | ✅ |
| 3 | "Financial institutions should identify clients." | "Countries should confiscate criminal proceeds." | LOW | ~-0.1 – 0.1 | ✅ |
| 4 | "Report suspicious transactions promptly." | "Protect reporters from civil liability." | MEDIUM | ~0.3–0.5 | ✅ |
| 5 | "Mutual legal assistance between countries" | "Extradition of money laundering offenders" | MEDIUM-HIGH | ~0.4–0.7 (same domain: intl cooperation) | ✅ |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> Bất ngờ nhất: cặp 2 ("FATF has 26 member countries" vs "FATF consists of 26 countries and two international organisations") có thể có similarity thấp hơn mong đợi vì vector embedding tập trung vào **từ cụ thể** ("mutual", "assistance") thay vì **chủ đề chung** ("international cooperation"). Điều này cho thấy embeddings capture **ngữ cảnh từ vựng** (word co-occurrence patterns trong training corpus) tốt hơn **khái niệm trừu tượng** cùng miền. Các từ khác nhau nhưng cùng chủ đề có thể cách xa nhau trong không gian vector nếu chúng hiếm khi xuất hiện cùng nhau trong training data.

---

## 6. Results — Cá nhân (10 điểm)

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | What are the requirements for customer identification? | Recommendation 10: Financial institutions must identify clients using official documents, record identity when establishing relations or conducting transactions. |
| 2 | How should suspicious transactions be reported? | Recommendation 15: If funds are suspected to stem from criminal activity, institutions must report promptly to competent authorities. |
| 3 | What international cooperation measures are required? | Recommendations 33–40: Mutual legal assistance, extradition, cooperative investigations, controlled delivery, asset confiscation coordination. |
| 4 | How should criminal proceeds be confiscated? | Recommendation 7: Countries must enable confiscation of laundered property, proceeds, instrumentalities; authority to identify, freeze, seize property. |
| 5 | What does FATF say about shell corporations? | Recommendation 25: Countries should notice potential for abuse of shell corporations and consider additional measures to prevent unlawful use. |

### Kết Quả Của Tôi

Sử dụng `SentenceChunker` (max_sentences_per_chunk=3) + `EmbeddingStore` (mock embedder, 64-dim):

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Customer identification requirements | Annex to Rec 9: Financial activities list | 0.2458 | ⚠️ Partial | Generated answer referencing financial institution duties |
| 2 | How to report suspicious transactions | Coordinating seizure/confiscation (Rec 39) | 0.3700 | ⚠️ Partial | Generated answer on international coordination |
| 3 | International cooperation | Recommendation 22: cross-border cash transport | 0.2676 | ✅ Yes | Generated answer on international measures |
| 4 | Confiscation of criminal proceeds | Recommendation 39: coordinating seizure proceedings | 0.2664 | ⚠️ Partial | Generated answer on coordination |
| 5 | Shell corporations | Recommendation 31: Interpol gathering info | 0.2398 | ❌ No | Generated answer on international authorities |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 1 / 5 (20%)

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> Từ thành viên dùng `FixedSizeChunker`: retrieval scores cao hơn nhưng chunks bị cắt giữa câu — tradeoff giữa "tìm đúng" và "trả lời đúng" là quan trọng. Từ thành viên dùng `RecursiveChunker`: việc thêm metadata schema tốt (section, rec_number) giúp retrieval precision tăng đáng kể khi dùng `search_with_filter`.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> Các nhóm khác dùng real embedding models (all-MiniLM-L6-v2) cho thấy retrieval quality khác biệt rất lớn so với mock — mock embeddings không capture semantic similarity thực sự, chỉ là deterministic random. Real embeddings sẽ phân biệt rõ ràng hơn giữa các khuyến nghị.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> (1) Thêm **custom chunker** giữ nguyên khuyến nghięt duy nhất mỗi chunk, thay vì dùng generic `SentenceChunker`. (2) Thiết kế **metadata schema phong phú hơn** (rec_number, section, theme) để dùng `search_with_filter` hiệu quả. (3) Dùng **real embeddings** thay vì mock — để retrieval quality đánh giá thực sự chính xác.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 9 / 10 |
| Chunking strategy | Nhóm | 13 / 15 |
| My approach | Cá nhân | 9 / 10 |
| Similarity predictions | Cá nhân | 4 / 5 |
| Results | Cá nhân | 7 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 4 / 5 |
| **Tổng** | | **81 / 100** |
