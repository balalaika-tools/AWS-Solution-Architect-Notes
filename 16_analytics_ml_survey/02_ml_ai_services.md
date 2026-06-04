# AI/ML Services — Survey

> **Who this is for**: Engineers prepping for SAA-C03 who need **recognition-level** coverage of AWS's managed AI/ML services. The classic exam pattern is a one-sentence use case ("extract text from scanned forms", "detect objects in images") that maps to exactly one managed service. You don't need to build models — you need to **match the use case to the API**. SageMaker is the one to understand best.

> **Survey scope**: One short paragraph + one table row per service. Memorize the keyword trigger, not the implementation. Services marked **adjacent context** are useful AWS knowledge but are not in the current SAA-C03 in-scope service list checked during this review.

---

## 1. Vision & Document AI

**Amazon Rekognition** — *Image and video analysis.* Detects objects, scenes, faces, text-in-images, celebrities, and **unsafe/inappropriate content (content moderation)**; does facial comparison and analysis. Use whenever the input is a **picture or video** and the task is "what's in it."
> 💡 **Exam clue**: "**detect objects / faces in images or video**", "**content moderation**", "facial recognition" → **Rekognition**.

**Amazon Textract** — *OCR that extracts text, forms, and tables from scanned documents.* Goes beyond raw OCR: understands key-value pairs (forms) and table structure (e.g., invoices, IDs, financial statements). Use for digitizing scanned **documents**.
> 💡 **Exam clue**: "**extract text / data from scanned documents / forms / PDFs**", "OCR" → **Textract**.

---

## 2. Language & Speech

**Amazon Comprehend** — *Natural Language Processing (NLP).* Finds **sentiment**, key phrases, entities, language, and PII in text; can do topic modeling. Use to analyze the *meaning* of text (reviews, support tickets, social posts).
> 💡 **Exam clue**: "**sentiment analysis**", "extract **entities / key phrases** from text", "NLP" → **Comprehend**.

**Amazon Transcribe** — *Speech-to-text.* Converts audio/video speech into text transcripts (supports speaker identification, custom vocabulary, medical variant). Use for call-center transcripts, captions, meeting notes.
> 💡 **Exam clue**: "**speech → text**", "transcribe audio", "generate captions/subtitles" → **Transcribe**.

**Amazon Polly** — *Text-to-speech.* Turns text into lifelike spoken audio (many voices/languages, neural voices). Use for voice responses, audio articles, IVR prompts.
> 💡 **Exam clue**: "**text → speech**", "convert text to lifelike voice/audio" → **Polly**.

**Amazon Translate** — *Neural machine translation between languages.* Use for localizing content or translating user input/text in real time.
> 💡 **Exam clue**: "**translate** between languages", "localize content" → **Translate**.

**Amazon Lex** — *Build conversational chatbots and voice bots* (the engine behind Alexa: ASR + NLU). Use to create chatbots/IVR that understand intent and respond.
> 💡 **Exam clue**: "**chatbot / conversational interface / voice bot**" → **Lex**.

---

## 3. Search, Recommendations & Forecasting

**Amazon Kendra** — *Intelligent, ML-powered enterprise search.* Natural-language search across internal documents/knowledge sources (SharePoint, S3, etc.) returning precise answers. Use for an internal "search our docs and get answers" use case.
> 💡 **Exam clue**: "**intelligent / enterprise search**", "natural-language search over internal documents/knowledge base" → **Kendra**.

**Amazon Personalize** (**adjacent context**) — *Real-time personalized recommendations* (the same tech behind Amazon.com). Use for "customers who viewed this also..." / product or content recommendations.
> 💡 **Exam clue**: "**recommendations**", "personalized product/content suggestions" → **Personalize**.

**Amazon Forecast** (**legacy / adjacent context**) — *Time-series forecasting* using ML. It is **not
available to new customers**; existing customers can continue. Recognize it in older exam material for
demand planning, inventory, capacity, or financial forecasts from historical time-series data. For new
designs, use SageMaker/AutoGluon or a domain-specific planning service where appropriate.
> 💡 **Exam clue**: "**forecast**", "predict future **time-series** demand/inventory" → **Forecast** in older material; **SageMaker/AutoGluon** for custom current designs.

**Amazon Fraud Detector** (**legacy / adjacent context**) — *Managed fraud-detection* for online
activity (fake accounts, payment fraud). As of **November 7, 2025**, it is **not open to new
customers**. Existing customers can continue. For new designs, AWS points toward AutoGluon/SageMaker
for custom fraud models, and **AWS WAF Fraud Control** for login/sign-up abuse patterns.
> 💡 **Exam clue**: "**detect online fraud**", "fraudulent transactions/accounts" → **Fraud Detector** in older material; **SageMaker/AutoGluon** or **WAF Fraud Control** for current greenfield designs.

---

## 4. Build-Your-Own & Generative AI

**Amazon SageMaker** — *Fully managed platform to build, train, tune, and deploy **custom** ML models* end to end (notebooks, training jobs, hosted inference endpoints, pipelines, feature store). This is the service to know best: when the managed AI APIs above *don't* fit and you must build/train a **custom model** with your own data and algorithm, the answer is SageMaker.
> 💡 **Exam clue**: "**build / train / deploy a custom ML model**", "data scientists need a managed ML platform", "host an inference endpoint" → **SageMaker**. (Contrast: pre-built task APIs → Rekognition/Comprehend/etc.)

**Amazon Bedrock** (**adjacent context**) — *Fully managed access to foundation models (FMs) for generative AI* via a single API (models from Anthropic, Amazon, Meta, and others). Serverless, no model hosting; supports retrieval-augmented generation (knowledge bases), agents, and fine-tuning/customization of FMs. Use for GenAI features — chat, summarization, content generation — without managing models or infrastructure.
> 💡 **Exam clue**: "**generative AI / foundation models / LLM** without managing infrastructure" → **Bedrock**.

---

## 5. Summary Table

| Service | What it is | When to use | Exam keyword |
|---|---|---|---|
| **Rekognition** | Image/video analysis | Detect objects/faces, content moderation | "objects/faces in images or video", "moderation" |
| **Textract** | OCR + form/table extraction | Digitize scanned docs/forms/invoices | "extract text from scanned documents/forms" |
| **Comprehend** | NLP (sentiment, entities) | Analyze meaning of text | "sentiment", "entities / key phrases", "NLP" |
| **Transcribe** | Speech-to-text | Audio → transcripts, captions | "speech to text", "transcribe audio" |
| **Polly** | Text-to-speech | Generate lifelike voice audio | "text to speech", "lifelike voice" |
| **Translate** | Language translation | Localize / translate text | "translate between languages" |
| **Lex** | Chatbots / voice bots | Conversational interfaces | "chatbot", "conversational / voice bot" |
| **Kendra** | Intelligent enterprise search | NL search over internal docs | "intelligent / enterprise search" |
| **Personalize** *(adjacent)* | Recommendations | Personalized product/content recs | "recommendations" |
| **Forecast** *(legacy/adjacent)* | Time-series forecasting | Existing customers / older recognition | "forecast time-series" |
| **Fraud Detector** *(legacy/adjacent)* | Managed fraud detection | Existing customers / older recognition | "online fraud detection" |
| **SageMaker** | Build/train/deploy custom ML | Custom models, managed ML platform | "build/train/deploy custom ML model" |
| **Bedrock** *(adjacent)* | Managed foundation models (GenAI) | LLM/GenAI features, no infra | "generative AI / foundation models / LLM" |

---

## 6. Key Exam Points (Use Case → Service)

The single most important skill here: match a one-sentence use case to the right **pre-built** API — and recognize when no pre-built API fits, so the answer is the **custom** path (SageMaker).

| Use case in the question stem | Pick |
|---|---|
| "Detect objects/people/text **in images or video**" | Rekognition |
| "**Moderate** uploaded images/video for inappropriate content" | Rekognition |
| "**Extract text/data from scanned forms / invoices / PDFs**" | Textract |
| "Determine **sentiment** / extract **entities** from text/reviews" | Comprehend |
| "Convert **speech to text** / generate captions" | Transcribe |
| "Convert **text to speech** / generate voice audio" | Polly |
| "**Translate** content into other languages" | Translate |
| "Build a **chatbot / voice assistant**" | Lex |
| "**Natural-language search** across internal documents" | Kendra |
| "Provide product / content **recommendations**" | Personalize |
| "**Forecast** future demand from historical data" | Forecast in older material; SageMaker/AutoGluon for current custom builds |
| "Detect **fraudulent** transactions/accounts" | Fraud Detector in older material; SageMaker/AutoGluon or WAF Fraud Control for current builds |
| "**Build and train a custom model** with our own data" | SageMaker |
| "Add **generative AI / LLM** features, managed, no infra" | Bedrock |

> 💡 **Mental model**: AWS sells **pre-built task APIs** (Rekognition, Textract, Comprehend,
> Transcribe, Polly, Translate, Lex, Kendra, Personalize) for *common* AI tasks. Some older
> recognition services such as Forecast and Fraud Detector are now legacy/existing-customer only.
> Reach for **SageMaker** when the task is *custom* and no current ready-made API matches. Reach for
> **Bedrock** when the requirement is *generative AI / foundation models*.

> ⚠️ **Classic traps**:
> - Text **inside an image/photo** → Rekognition (image text). Text **in a scanned document/form** → Textract. Analyzing the **meaning** of plain text → Comprehend.
> - "Build/train a model" → SageMaker. "Use a ready model for X task" → the matching managed API. Don't default to SageMaker for tasks a pre-built service already solves (more work, wrong answer).

---

**Next**: [../17_exam_patterns/01_architecture_decision_guides.md — Architecture Decision Guides](../17_exam_patterns/01_architecture_decision_guides.md)
