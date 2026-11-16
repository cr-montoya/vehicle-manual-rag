# Phase 0 inception: vehicle manual assistant

AI-DLC: the AI proposes and asks, the human decides. Everything is validated before writing code. Scope: one or two 90-minute blocks.

> This is the original planning document, kept as written at the time (translated to English). Some decisions changed during the build: the generator is Amazon Nova 2 Lite instead of Claude Haiku (see problem 1 and problem 23 in the [engineering log](bitacora.md)), the PDFs are parsed with a vision model, and the corpus uses real manufacturer manuals that are downloaded, not redistributed.

## 1. Intent

A non-technical person asks about their motorcycle or car ("how often do I change the oil?", or a photo of the dashboard: "what does this light mean?") and gets what the manual of **that** vehicle says, with the page cited. If the topic is safety related (brakes, temperature, oil), the answer ends with "take it to a mechanic".

## 2. Criteria (in order of weight)

1. **Never answer from the wrong manual.** This is the main risk.
2. **Traceable answer:** always with a document and page citation, or "it is not in the manual".
3. **Usable from a phone** by a non-technical person.
4. **Near-zero cost and destroyable** with `cdk destroy`.

## 3. Autonomy level

**Level 2: RAG**, with one multimodal call to read the photo. No agent is needed: there are no actions and no path that depends on the case. Rejected alternative: an agent with tools (level 4). That level adds latency and risk without adding value here.

## 4. Proposed design (the human decides)

| Component | Proposal | Rejected alternative and why |
| --- | --- | --- |
| Source | S3 with one `.metadata.json` per PDF: `tipo`, `marca`, `modelo`, `anio` (type, make, model, year) | One index per vehicle: does not scale and duplicates infrastructure |
| RAG | Amazon Bedrock Knowledge Base | Custom pipeline with LangChain: more code and operations |
| Vector store | **Amazon S3 Vectors** (GA since December 2025, integrated with Knowledge Bases) | Amazon OpenSearch Serverless: bills compute in OCU-hours (USD 0.24 per OCU-hour in us-east-1, about USD 175 per month for each OCU kept running), and a [minimum capacity](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-scaling.html) above 0 is billed even when idle; unjustified for 3 PDFs |
| Embeddings | Amazon Titan Text Embeddings V2 | Cohere: adds nothing for 3 documents |
| Generator | **Claude Haiku 5.5** (vision, speed, cost); move up to Sonnet 5.5 only if the golden set shows it misreads photos | Sonnet from the start: more expensive with no evidence it is needed |
| Safety | Basic Amazon Bedrock Guardrails: out-of-scope topics and prompt injection | No guardrail: a non-technical person will use it with free-form questions |
| Interface | Minimal web page (S3 and CloudFront) with a Lambda function behind it: vehicle selector, text and photo | Streamlit on a server: it has to be operated and is not serverless |
| Access | Shared link with a usage limit (throttling) and a budget alarm | Amazon Cognito: excessive for a one-day demo |

**Flow:** the user picks their vehicle, or names it in the text. With no identified vehicle, the system asks before answering. With a vehicle, the search is filtered by `marca`, `modelo` and `anio` before comparing vectors. If there is a photo, Haiku describes it first ("oil pressure warning light") and that description is used as the query. The answer includes the citation and, if the topic is safety related, the mechanic warning.

## 5. Assumptions

| Assumption | If it turns out false, this changes |
| --- | --- |
| Personal account with AWS credits in us-east-1 | The region and the model IDs |
| The PDFs have extractable text, they are not scans | Parsing with a foundation model or Amazon Textract would be needed at ingestion |
| The page citation comes from the location metadata the knowledge base returns (confirmed: the Lambda function reads `x-amz-bedrock-kb-document-page-number` from each retrieved chunk) | Each PDF is split by page before ingestion |
| The "take it to a mechanic" rule lives in the prompt, plus a keyword list in the Lambda function | It moves to a deterministic rule in code, outside the model (this is how it was implemented) |

## 6. Acceptance criteria: a golden set of 10 questions

- **6 text questions answered in the manual**, such as oil interval, tire pressure or fuel tank capacity. They pass if the citation points to the right page.
- **2 dashboard photos.** They pass if they identify the warning light and answer with a citation and the safety warning.
- **Trap 1, wrong vehicle:** ask about the motorcycle with the car selected. Passes if it does not answer from the other manual.
- **Trap 2, outside the manuals:** for example, "how much does insurance cost?". Passes if it answers "it is not in the manual".

Proposed threshold: 9 out of 10, with both traps required. Decided: 9 out of 10 with every trap required (the final golden set has three traps).

## 7. Open questions for the human

1. **Corpus:** real or fictional manuals? Official manufacturer manuals are copyrighted, and I did not find a license that allows using them in a demo. Fictional ones can be shown and versioned without that risk. Decided: real manuals, downloaded from each manufacturer's official URL and never redistributed in the repository.
2. Threshold of 9 out of 10?
3. Haiku 5.5 as the initial generator?
