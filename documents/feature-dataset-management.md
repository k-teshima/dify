# Dataset Management

## 1. Feature Overview

The Dataset Management feature provides a comprehensive system for creating, managing, and utilizing knowledge datasets. It supports various data sources such as file uploads (PDF, TXT, Markdown, DOCX, CSV, Excel), Notion imports, and website scraping. Datasets undergo a processing pipeline including content extraction, cleaning, segmentation, and embedding (for vector search) or keyword indexing (for keyword search). These indexed datasets are then made available for Retrieval Augmented Generation (RAG) in AI applications, enabling them to answer queries based on the ingested knowledge.

## 2. Architecture Diagram

```mermaid
flowchart TD
    subgraph UserInput [Data Sources]
        A[File Upload] --> C
        B[Notion Import] --> C
        B1[Website Crawl] --> C
    end

    subgraph APIConsole [API/Console Layer]
        C{Dataset & Document Operations via UI/API} --> D["/console/datasets/*"]
        D -- Create/Update Dataset --> DatasetService[DatasetService]
        D -- Add/Update Document --> DocumentService[DocumentService]
        D -- Manage Segments --> SegmentService[SegmentService]
    end

    subgraph ServiceLayer [Service Layer]
        DatasetService --> DatasetModel[Dataset Model]
        DocumentService --> DocumentModel[Document Model]
        DocumentService --> IndexingTask[Async Indexing Task (Celery)]
        SegmentService --> VectorDB[VectorService]
        SegmentService --> SegmentModel[DocumentSegment Model]
    end

    subgraph IndexingPipeline [Core RAG & Indexing Pipeline (Async)]
        IndexingTask --> Extract[ExtractProcessor]
        Extract --> Clean[Cleaners]
        Clean --> Split[TextSplitter]
        Split --> EmbedKeyword{Embedding / Keyword Indexing}
        EmbedKeyword -- High Quality --> EmbeddingModel[Embedding Models]
        EmbeddingModel --> VectorDB
        EmbedKeyword -- Economy --> KeywordTable[DatasetKeywordTable]
        VectorDB --> VectorStore[(Vector Store e.g., Qdrant, Milvus)]
    end
    
    subgraph RetrievalLayer [Retrieval Layer]
        App[Application RAG] --> Retrieval[DatasetRetrieval Service]
        Retrieval --> VectorStore
        Retrieval --> KeywordTable
    end

    subgraph Database [Storage - PostgreSQL & Vector Store]
        DatasetModel --> DatasetsTable[(Datasets Table - datasets)]
        DocumentModel --> DocumentsTable[(Documents Table - documents)]
        SegmentModel --> SegmentsTable[(Segments Table - document_segments)]
        KeywordTable --> KeywordStorage[(Keyword Store - dataset_keyword_tables)]
    end
```

**Diagram Explanation:**

*   **Data Sources:** Users can create datasets from various sources like file uploads, Notion, or website crawls.
*   **API/Console Layer:** Handles user requests for dataset and document operations through REST APIs, typically invoked by the admin console.
*   **Service Layer:**
    *   `DatasetService`: Manages the lifecycle (CRUD) of datasets, including their configuration (name, description, indexing technique, embedding models).
    *   `DocumentService`: Manages documents within datasets, including adding new documents from different sources and triggering their indexing.
    *   `SegmentService`: Manages document segments (chunks) and their corresponding vector embeddings or keyword indexes.
    *   `VectorService`: Abstracts interactions with the underlying vector database.
*   **Core RAG & Indexing Pipeline (Async):** This is typically an asynchronous process.
    *   `ExtractProcessor`: Extracts text content from various source types.
    *   `Cleaners`: (Implied) Cleans the extracted text.
    *   `TextSplitter`: Divides the cleaned text into manageable segments or chunks.
    *   `Embedding Models / Keyword Indexing`:
        *   For "high_quality" indexing, embedding models convert text segments into vector embeddings.
        *   For "economy" indexing, keywords are extracted.
    *   `Vector Store / KeywordTable`: Stores the embeddings or keywords for efficient retrieval.
*   **Retrieval Layer:**
    *   `DatasetRetrieval Service`: Used by applications to query datasets. It fetches relevant segments from the vector store or keyword table based on the query.
*   **Storage:**
    *   PostgreSQL stores metadata for datasets, documents, segments, processing rules, etc.
    *   Vector Store (e.g., Qdrant, Milvus, Weaviate) stores the vector embeddings of document segments.

## 3. Related Modules

| Path                                               | Role                                                                    | Major Classes/Functions                                                              |
| -------------------------------------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `api/services/dataset_service.py`                  | Core dataset/document/segment lifecycle, indexing triggers, permissions. | `DatasetService`, `DocumentService`, `SegmentService`, `create_empty_dataset`, `save_document_with_dataset_id`, `create_segment` |
| `api/controllers/console/datasets/datasets.py`     | Console API endpoints for dataset CRUD and general settings.              | `DatasetListApi`, `DatasetApi`                                                       |
| `api/controllers/console/datasets/datasets_document.py` | Console API endpoints for document CRUD within datasets.                | `DocumentListApi`, `DocumentDeleteApi`, `DocumentDetailApi`                          |
| `api/models/dataset.py`                            | Database models for datasets, documents, segments, rules, etc.          | `Dataset`, `Document`, `DocumentSegment`, `DatasetProcessRule`, `DatasetKeywordTable`, `Embedding`, `DatasetCollectionBinding` |
| `api/core/rag/extractor/extract_processor.py`      | Selects and runs appropriate content extractors for data sources.       | `ExtractProcessor`, `extract`                                                        |
| `api/core/rag/extractor/`                          | Specific content extractors for various file types and sources.         | `PdfExtractor`, `WordExtractor`, `NotionExtractor`, `FirecrawlWebExtractor`          |
| `api/core/rag/splitter/text_splitter.py`           | Document segmentation strategies (character, recursive, token-based).   | `TextSplitter`, `RecursiveCharacterTextSplitter`, `TokenTextSplitter`                |
| `api/core/rag/embedding/embedding_base.py`         | Abstract base class for embedding models.                               | `Embeddings`                                                                         |
| `api/core/rag/embedding/cached_embedding.py`       | Caches embeddings to avoid re-computation.                              | `CachedEmbeddings`                                                                   |
| `api/core/rag/datasource/vdb/vector_base.py`       | Abstract base class for vector database interactions.                   | `BaseVector`                                                                         |
| `api/core/rag/datasource/vdb/`                     | Implementations for specific vector databases (Qdrant, Milvus, etc.).   | `QdrantVector`, `MilvusVector`                                                       |
| `api/core/rag/retrieval/dataset_retrieval.py`      | Service for retrieving relevant documents/segments from datasets.       | `DatasetRetrieval`, `retrieve`                                                       |
| `tasks/document_indexing_task.py`                  | Celery task for asynchronous document indexing.                         | `document_indexing_task`                                                             |
| `core/indexing_runner.py`                          | Orchestrates the document indexing pipeline (extract, split, embed).    | `IndexingRunner` (Note: Not in the provided list but inferred from task calls)       |

## 4. Data Flow

### a. Document Upload and Indexing ("high_quality" technique)

1.  **Input:** User uploads a PDF file to an existing dataset via the Console UI. The dataset is configured for "high_quality" indexing.
2.  **API Request:** The UI sends a `POST` request to `/console/datasets/<dataset_id>/documents`.
3.  **Controller & Service Processing:**
    *   `api/controllers/console/datasets/datasets_document.py -> DocumentListApi.post()` receives the request.
    *   It calls `api/services/dataset_service.py -> DocumentService.save_document_with_dataset_id()`.
    *   `DocumentService` creates a `Document` record in the database, linking it to the specified `Dataset`. It records data source information (e.g., upload file ID) and sets the initial `indexing_status` to "waiting".
    *   A `DatasetProcessRule` (either default or custom for the dataset) is associated with the document.
    *   An asynchronous Celery task, `tasks.document_indexing_task.document_indexing_task`, is invoked with the `document_id` and `dataset_id`.
4.  **Asynchronous Indexing Pipeline (executed by Celery worker):**
    *   The `document_indexing_task` likely calls an `IndexingRunner` or similar orchestrator.
    *   **Extraction:** `api/core/rag/extractor/extract_processor.py -> ExtractProcessor.extract()` is called. It identifies the file type (PDF) and uses `PdfExtractor` to get the text content.
    *   **Cleaning:** (Implied) The extracted text may undergo cleaning steps (e.g., removing extra spaces).
    *   **Splitting:** Based on the `DatasetProcessRule` (e.g., segmentation strategy, chunk size), `api/core/rag/splitter/text_splitter.py -> RecursiveCharacterTextSplitter` (or another splitter) divides the document content into multiple segments.
    *   **Embedding:**
        *   For each segment, the configured embedding model (e.g., from OpenAI, specified in `Dataset.embedding_model` and `Dataset.embedding_model_provider`) is retrieved using `ModelManager`.
        *   `api/core/rag/embedding/embedding_base.py` (and its implementations) generates a vector embedding for the segment's content. `CachedEmbeddings` might be used to avoid re-computing.
    *   **Storage & Indexing:**
        *   `api/services/dataset_service.py -> SegmentService` (or `VectorService` directly) is used.
        *   For each segment, a `DocumentSegment` record is created in the database, storing its content, token count, and position.
        *   The generated vector embedding and the `DocumentSegment.id` (or a unique `index_node_id`) are stored in the vector database (e.g., Qdrant, Milvus) associated with the dataset's collection (identified by `Dataset.collection_binding_id`). This is handled by an implementation of `api/core/rag/datasource/vdb/vector_base.py -> BaseVector`.
    *   **Status Update:** The `Document`'s `indexing_status` is updated to "completed". `word_count` and `tokens` are also updated.
5.  **Output:** The document is processed, segmented, and its embeddings are stored in the vector database. The document becomes queryable for RAG.

## 5. Extension Points

*   **Adding a New Data Source:**
    1.  Implement a new extractor class in `api/core/rag/extractor/` (e.g., `MyNewDataSourceExtractor`) inheriting from `BaseExtractor`.
    2.  Update `api/core/rag/extractor/entity/datasource_type.py -> DatasourceType` enum if needed.
    3.  Modify `api/core/rag/extractor/extract_processor.py -> ExtractProcessor.extract()` to recognize and use the new extractor based on `ExtractSetting`.
    4.  Update `api/controllers/console/datasets/datasets_document.py` and frontend components to support the new data source type in the UI.
*   **Supporting a New File Type for Upload:**
    1.  Create a new extractor class in `api/core/rag/extractor/` (e.g., `MyNewFileExtractor`) for the file type.
    2.  Update `ExtractProcessor.extract()` to use this new extractor based on the file extension.
*   **Integrating a New Vector Database:**
    1.  Implement a new vector store client in `api/core/rag/datasource/vdb/` (e.g., `my_new_vdb_vector.py`) inheriting from `BaseVector`.
    2.  Add the new vector type to `api/core/rag/datasource/vdb/vector_type.py -> VectorType`.
    3.  Update `api/core/rag/datasource/vdb/vector_factory.py -> VectorFactory` to instantiate the new client.
    4.  Add configuration options for the new vector DB in `configs/`.
*   **Adding a New Text Splitter Strategy:**
    1.  Create a new splitter class in `api/core/rag/splitter/` inheriting from `TextSplitter`.
    2.  Update `DatasetProcessRule` and the UI to allow selection and configuration of this new splitter.
*   **Supporting a New Embedding Model:**
    1.  This is primarily handled by the Model Management feature.
    2.  Ensure the `Embedding` interface in `api/core/rag/embedding/embedding_base.py` is compatible or extend it if necessary.
    3.  The new embedding model should be selectable in the dataset configuration UI.
```
