# Bài 7 — RAG Pipeline (Retrieval‑Augmented Generation)

Mục tiêu: Giải thích RAG (cả Traditional và Agentic RAG), cách áp dụng trong repo TripPlanner và hướng dẫn thực tế để triển khai một RAG workflow trong mã nguồn hiện có.

---

## 1) RAG là gì (tóm tắt nhanh)

- RAG = Retrieval‑Augmented Generation: kết hợp truy vấn một kho tri thức (retriever) với khả năng sinh ngôn ngữ của LLM để trả lời chính xác hơn.
- Lợi ích: giảm hallucination, trả lời dựa trên bằng chứng, cho phép sử dụng dữ liệu nội bộ (docs, bài báo, FAQ).

## 2) Traditional RAG vs Agentic RAG

- Traditional RAG: query → retriever (vector/keyword) → LLM tổng hợp.
- Agentic RAG: giống Traditional nhưng có thêm nhiều agent chuyên trách (retrieval, pre/post processing, evaluation, multimodal), có feedback loops và điều phối tác vụ giữa agents.

(Tài liệu tham khảo: `docs/Agentic RAG Pipeline.md` trong repo.)

## 3) Thành phần RAG và map vào repo

- Query / Prompt: do frontend (`demo.py`) hoặc agent nhận từ người dùng.
- Retriever: hiện repo chưa có vector DB cụ thể; bạn có thể thêm một tool wrapper (ví dụ `src/agentic/tools/vector_store.py`) để giao tiếp với FAISS/Pinecone/Weaviate.
- Knowledge Base: có thể là tập tài liệu (Markdown, HTML, JSON) hoặc nội dung thu thập từ `WebResearchAgent`.
- LLM: `LoadModel.load_openai_model()` (xem `src/agentic/utils/main_utils.py`).
- Agent orchestration: các agent trong `src/agentic/agents/` có thể gọi retriever tool rồi tổng hợp kết quả.

## 4) Cách triển khai RAG trong repo (bước‑bước)

1) Chọn và triển khai Vector DB (ví dụ FAISS cho local, Pinecone/Gcore/Weaviate cho production).

2) Tạo tool wrapper `src/agentic/tools/vector_store.py` theo pattern repo:

```python
# pseudo
class VectorStore:
    @classmethod
    def upsert_documents(cls, docs):
        # embed + upsert into index
        return True

    @classmethod
    def retrieve(cls, query, k=5):
        # return top-k documents with scores
        return [{"id":"...","text":"...","score":0.9}]
```

3) Pipeline ingest (script): thu thập tài liệu (từ web tools hoặc folder `docs/`), tách chunk, embed và upsert vào vector store.

4) Retriever tool: viết wrapper trả về callable để agent gọi (ví dụ `VectorStore.retrieve`). Thêm wrapper vào `tools` list khi khởi tạo agent (ví dụ `ReporterAgent` hoặc một `RetrievalAgent`).

5) Khi có query:
   - Agent gọi retriever tool → nhận top‑k documents.
   - Agent kết hợp query + docs thành prompt (context window management: chọn token budget, k sao cho phù hợp).
   - Gọi LLM với prompt đã augment, nhận kết quả.

6) Post‑processing: lọc/format kết quả, gắn nguồn tham chiếu (source IDs, links), và đưa ra feedback loop cho retriever nếu cần.

## 5) Ví dụ pseudo‑code tích hợp vào `ReporterAgent`

```python
# trong src/agentic/agents/retrieval_agent.py hoặc reporter_agent.py
retriever = VectorStore.retrieve  # wrapper callable

def answer_with_rag(agent, query):
    docs = retriever(query, k=5)
    context = "\n\n".join([d['text'] for d in docs])
    prompt = f"Use the following documents as context:\n{context}\n\nQuestion: {query}\nAnswer concisely with sources."
    return agent.llm.generate(prompt)
```

> Ghi chú: `taskflowai` trong repo sử dụng `Task.create(...)` để gửi instruction — bạn có thể viết một `Task` mà agent thực hiện: gọi retriever tool (callable) rồi dùng LLM để tổng hợp.

## 6) Quản lý token / context window

- Chunk tài liệu hợp lý (200–1000 tokens) trước khi embed.
- Lọc theo score, giới hạn tổng tokens gửi đến LLM.
- Khi cần multi‑hop, dùng iterative retrieval: lấy docs → generate intermediate question → retrieve tiếp.

## 7) Multi‑modal & attribution

- Nếu tài liệu chứa ảnh, lưu metadata `image_url` cùng text chunk để reporter có thể hiển thị ảnh.
- Luôn kèm nguồn (source id, URL) trong output để người dùng kiểm tra.

## 8) Testing & validation

- Unit test wrapper vector store bằng cách mock embed + index client.
- Integration test ingest + retrieve bằng sample dataset cục bộ (FAISS local).
- End‑to‑end test: mock LLM hoặc dùng sandbox key với quota giới hạn.

## 9) Lưu ý vận hành

- Chi phí embed/queries: cache nhiều kết quả, batch embed khi ingest.
- Bảo mật dữ liệu: kiểm soát access với vector DB hosted.
- Chống nhiễu (noisy docs): thêm filtering/cleaning bước ingest.

---

Tài nguyên trong repo để tham chiếu:

- [docs/Agentic RAG Pipeline.md](docs/Agentic%20RAG%20Pipeline.md) — khái niệm tổng quan.
- Notebook demo: [notebooks/TripPlanner_Multi_AI_Agent_Experimental.ipynb](notebooks/TripPlanner_Multi_AI_Agent_Experimental.ipynb) — nơi thử nghiệm luồng agent.

Bạn muốn tôi tạo luôn một `src/agentic/tools/vector_store.py` mẫu (FAISS) và script ingest ví dụ không?