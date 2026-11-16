# Job Queues, Background Workers and OCR

## 1. Job Queue

A job queue is a system that temporarily holds tasks until a worker is available to process them.

**SearchForge example:**

When three engineers upload three PDFs:
- PDF A and PDF B are picked up by two available workers.
- PDF C waits in the job queue.
- When a worker becomes available, it picks up PDF C.

**Why do we need it?**
- Manage multiple tasks.
- Handle more jobs than available workers.
- Prevent the API from performing all processing synchronously.

## 2. Producer and Consumer

These are the two main roles in a job queue system.

**Producer:** Creates jobs and adds them to the queue.

In SearchForge, the Express API acts as the producer.

**Consumer:** Picks up jobs and processes them.

In SearchForge, the BullMQ worker acts as the consumer.

| Component | Responsibility |
|---|---|
| Express API | Producer |
| Redis | Stores queue data |
| BullMQ | Manages jobs |
| Worker | Consumer |

## 3. Background Workers

A background worker is a separate process that performs time-consuming tasks independently of the user's browser.

**Without a worker:**
1. API receives a PDF.
2. API extracts its text.
3. API creates passages.
4. API indexes the document.
5. User waits for processing to finish.

**With a worker:**
1. API receives the PDF.
2. API creates an indexing job.
3. BullMQ manages the job through Redis.
4. API responds that processing has started.
5. Worker processes the PDF independently.

**Important:** Closing the browser does not stop processing because the worker runs separately. However, the worker/server must remain operational, and failed jobs need recovery mechanisms.

## 4. OCR — Optical Character Recognition

OCR is a technology that recognizes text inside images and converts it into machine-readable text.

**Why does SearchForge need OCR?**

Some PDFs contain scanned pages. Although they look like documents, their content is actually stored as images.

Normal PDF text extraction may not recognize this content.

OCR solves this problem by recognizing the characters inside those images.

**Possible OCR technologies:**
- Tesseract OCR — open-source OCR engine.
- Google Cloud Vision — cloud-based text recognition.
- AWS Textract — text and structured data extraction.

For our initial MVP, Tesseract is a reasonable option to evaluate.

## 5. Hybrid PDF Text Extraction

SearchForge should not unnecessarily run OCR on every page.

Example: A PDF contains 100 pages:
- 80 pages contain selectable text.
- 20 pages contain scanned images.

Our worker should:
1. Extract normal text from each page.
2. Check whether sufficient meaningful text was extracted.
3. Run OCR on pages with insufficient text.
4. Preserve the extracted text and original page number.
5. Continue to segmentation and indexing.

```text
             Uploaded PDF
                  |
          Process each page
                  |
          Normal extraction
                  |
          Is text sufficient?
             /         \
           Yes          No
            |            |
       Keep text      Run OCR
            |            |
            +-----+------+
                  |
          Combined text
                  |
             Segmentation
                  |
             Tokenization
                  |
               Indexing
```

**Why use this approach?**
- Avoid unnecessary OCR processing.
- Reduce processing time and resource consumption.
- Support both normal and scanned PDFs.
- Preserve page-level source references for citations.

OCR may produce errors with blurry scans, handwriting and complex layouts. SearchForge should preserve extraction quality information where practical.

## 6. Key Takeaways

1. A job queue holds tasks until workers are available.
2. The producer creates jobs; the consumer executes them.
3. BullMQ manages jobs, Redis stores queue data, and workers process jobs.
4. Background workers allow processing to continue independently of the browser.
5. OCR extracts text from scanned images.
6. Hybrid extraction uses normal extraction where possible and OCR where necessary.
7. Page numbers must be preserved for accurate source citations.

## 7. Interview Questions

**Q1. What is a job queue?**

A system that holds tasks until workers are available to process them.

**Q2. What is the difference between a producer and a consumer?**

A producer creates jobs and adds them to the queue. A consumer retrieves and executes those jobs.

**Q3. Why do we use background workers?**

To execute time-consuming tasks independently of the main API request.

**Q4. What happens if a user closes the browser during PDF processing?**

Processing can continue because the background worker runs independently of the browser, provided the worker service remains operational.

**Q5. What is OCR?**

Optical Character Recognition converts text inside images into machine-readable text.

**Q6. Why shouldn't we run OCR on every PDF page?**

Because normal text extraction is sufficient for pages containing selectable text. OCR only needs to be used where normal extraction is insufficient.

---

**Research references:**
- [BullMQ Introduction](https://docs.bullmq.io/guide/introduction)
- [BullMQ Workers](https://docs.bullmq.io/guide/workers)
- [Tesseract OCR](https://github.com/tesseract-ocr/tesseract)
