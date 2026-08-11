# Multimodal, Structure-Aware RAG Pipeline for PDFs

## The Problem with Traditional RAG on PDFs
When dealing with PDFs that contain tables, images, and text, treating the PDF as plain text and applying fixed-size chunking leads to a loss of critical relationships between these elements.

## The Solution
Instead of traditional plain-text extraction and fixed-size chunking, build a **multimodal, structure-aware RAG (Retrieval-Augmented Generation) pipeline**.

### 1. Layout-Aware Document Parsing
Extract headings, paragraphs, tables, images, and their positions accurately.
- **Tools/Libraries:** PyMuPDF, Docling, Unstructured, LlamaParse

### 2. OCR for Scanned Documents
For scanned PDFs and images, extract text while preserving layout information using Optical Character Recognition (OCR).
- **Tools/Models:** PaddleOCR, Tesseract, EasyOCR, Surya OCR

### 3. Table Extraction
Extract tables as structured formats (Markdown, HTML, or JSON) instead of flattening them into plain text. This preserves rows, columns, and tabular relationships.
- **Tools/Libraries:** Camelot, pdfplumber, Tabula, Docling

### 4. Image & Chart Understanding
Extract images, charts, and diagrams, and generate contextual descriptions for them using Vision Language Models (VLMs).
- **Models:** GPT-4o, Claude 3.5 Sonnet, Gemini 1.5 Pro, Qwen-VL

### 5. Structure-Aware Chunking
Instead of splitting text into arbitrary fixed-size chunks (e.g., 500 tokens), split the content based on its logical structure: headings, sections, tables, and semantic boundaries.
- **Tools/Libraries:** LangChain, LlamaIndex, Docling

### 6. Preserve Relationships
Store rich metadata to keep related text, tables, and images connected via parent-child relationships and neighboring chunk references.
Metadata to store:
- `page_number`
- `section`
- `element_type`
- `parent_id`
- `document_id`

### 7. Embeddings & Storage
Generate embeddings for the extracted and structured content and store them in a Vector Database.
- **Embedding Models:** BGE, E5, OpenAI Embeddings
- **Vector Databases:** Pinecone, Qdrant, Milvus, Chroma, FAISS

### 8. Retrieval & Context Expansion
Use hybrid search (combining keyword and vector search), reranking, and metadata filtering to retrieve the most relevant content. Before sending the context to the LLM, expand the retrieved chunk using its parent, neighboring, or related elements to provide full context.
- **Libraries/Models:** BM25 (Keyword search), BGE Reranker, Cohere Rerank, LangChain, LlamaIndex

## Deep Dive: Fixed-Size vs. Structure-Aware Chunking

Traditional fixed-size chunking blindly splits text every `N` tokens. This often splits sentences in half, breaks tables across multiple chunks, and completely orphans images from the text that describes them. 

Structure-aware chunking solves this by using the document's logical elements as boundaries.

```mermaid
graph LR
    subgraph "Traditional (Fixed-Size) Chunking"
        direction TB
        Doc1[PDF Document] --> Chunk1["Chunk 1<br>(Tokens 0-500)<br>Contains: Heading + Half of Table"]
        Doc1 --> Chunk2["Chunk 2<br>(Tokens 501-1000)<br>Contains: Other half of Table + Text"]
        style Chunk1 fill:#ffcccc,stroke:#cc0000
        style Chunk2 fill:#ffcccc,stroke:#cc0000
    end

    subgraph "Structure-Aware Chunking"
        direction TB
        Doc2[PDF Document] --> SC1["Chunk 1: Heading & Intro (Text)"]
        Doc2 --> SC2["Chunk 2: Complete Table (Markdown/JSON)"]
        Doc2 --> SC3["Chunk 3: Image + VLM Description"]
        style SC1 fill:#ccffcc,stroke:#009900
        style SC2 fill:#ccffcc,stroke:#009900
        style SC3 fill:#ccffcc,stroke:#009900
    end
```

## Deep Dive: VLM Enrichment and Context Expansion

### Vision Language Model (VLM) Enrichment
When a standard RAG pipeline encounters an image or chart, it usually ignores it. In a multimodal RAG:
1. The parser extracts the image file.
2. The image is passed to a VLM (like GPT-4o or Claude 3.5 Sonnet) with a prompt: *"Describe this chart/image in detail. Extract any data points and explain the trends."*
3. The VLM generates a rich text description.
4. This generated text description is embedded and stored in the Vector DB, linked via metadata to the original image.

### Context Expansion (Parent-Child Retrieval)
If a user asks about a specific row in a table, the vector search might only return that single row (a small chunk). Passing just one row to the LLM often lacks context.
**Context Expansion** fixes this:
1. The Vector DB retrieves Chunk `C` (e.g., the specific row).
2. The system checks `C`'s metadata and finds its `parent_id` (e.g., `Table_A`).
3. The system dynamically fetches the entire `Table_A` (and perhaps the surrounding explanation paragraphs).
4. This fully expanded context is sent to the LLM, resulting in a highly accurate, context-aware answer.

---

## Complete Pipeline Architecture

The complete end-to-end flow of the structure-aware RAG pipeline looks like this:

```mermaid
flowchart TD
    A[PDF Document] --> B(Layout Parsing + OCR<br/>Text/Table/Image Extraction)
    B --> C(VLM Enrichment<br/>Image/Chart Descriptions)
    C --> D(Structure-Aware Chunking)
    D --> E(Relationship Mapping<br/>Metadata Injection)
    E --> F(Embeddings Generation)
    F --> G[(Vector Database)]
    
    Q[User Query] --> H(Hybrid Retrieval + Reranking)
    G --> H
    H --> I(Context Expansion<br/>Fetch Neighbors/Parents)
    I --> J(Large Language Model)
    J --> K[Final Answer]
```
