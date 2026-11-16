# Engineering log: problems and fixes

Everything that went wrong while building this app, in the order it happened: what was seen, why it happened, how it was fixed and what was learned. The diagnosis mistakes are here too, because they teach as much as the system errors.

Quoted model answers and log messages are translated from Spanish; the original Spanish text is in the code and in [`pruebas/resultados.md`](../pruebas/resultados.md).

## Summary

| # | Problem | Type | Status |
| --- | --- | --- | --- |
| 1 | No access to Anthropic models in the account | Platform | Solved: Amazon Nova |
| 2 | `aws-cdk-lib` does not install on Python 3.9 | Environment | Solved: uv with Python 3.12 |
| 3 | Manuals with unreadable text or links that are not PDFs | Data | Solved: verified corpus |
| 4 | The account blocks public buckets | Security | Solved: CloudFront with OAC |
| 5 | A stopped `cdk deploy` kept deploying | Tooling | Solved |
| 6 | Reorganizing the code forced a stack teardown | IaC | Solved |
| 7 | `aws login` does not work with boto3 | Environment | Solved: `boto3[crt]` |
| 8 | Maintenance tables lost their columns | RAG: parsing | Solved: vision parser |
| 9 | Changing the parser did not guarantee reprocessing | RAG: ingestion | Solved: new data source |
| 10 | Ingestion took more than 30 minutes | Operations | Explained: quota, not cost |
| 11 | Token metrics showed 0 | Costs | Explained |
| 12 | **The parser invented a table on the index page** | RAG: hallucination | Solved: prompt without examples |
| 13 | The Lambda function was reachable without going through CloudFront | Security | Solved: OAC and `AWS_IAM` |
| 14 | The demo stayed exposed overnight | Security | Solved: switch and geo restriction |
| 15 | The "take it to a mechanic" warning showed up too often | Product | Solved: symptom-based rules |
| 16 | A guardrail version that does not update itself | IaC | Solved |
| 17 | **Wrong diagnosis:** an "invented" value that was actually there | Process | Corrected |
| 18 | **No alarm reached Slack** | Observability | Solved: SNS policy |
| 19 | The golden set does not pass: 8 out of 10 | Evaluation | Open |
| 20 | Supported but incomplete answer | RAG: generation | Open: the instruction was not enough |
| 21 | **Manual pages that never reached the index** | RAG: ingestion | Open: cause to be confirmed |
| 22 | A "not found" phrased differently passed as an answer | Product | Solved: no citations = not found |
| 23 | **A more capable model answered worse** | Model | Reverted to Nova 2 Lite |

---

## 1. No access to Anthropic models in the account

- **Symptom:** the initial plan used Claude as the generator, but the account had no access to Anthropic models.
- **Cause:** Anthropic model access was not enabled on the account, and it was not confirmed whether the account's AWS credits would cover third-party models, so the build stayed on AWS's own models.
- **Fix:** Amazon Nova 2 Lite as the generator and Nova Lite as the parser. They are AWS models: they read images, they are cheap and credits cover them. Nova 2 Lite is invoked through the `us.amazon.nova-2-lite-v1:0` inference profile, because it does not support direct invocation in the region.
- **Lesson:** the model is a replaceable part if the rest of the design does not depend on it.

## 2. `aws-cdk-lib` does not install on Python 3.9

- **Symptom:** `pip` only found old versions of `aws-cdk-lib`.
- **Cause:** recent versions require Python 3.10 or later, and the system had 3.9.
- **Fix:** [uv](https://docs.astral.sh/uv/) with a uv-managed Python 3.12, without touching the system Python. The project lives in `infra/pyproject.toml` with `uv.lock`.
- **Lesson:** pin the interpreter version in the project; do not depend on the one on the machine.

## 3. Manuals with unreadable text or links that are not PDFs

- **Symptom:** some manual links returned a web page instead of a PDF, and the manuals from another brand extracted as meaningless characters.
- **Cause:** sites that answer any path with the web application, and PDFs with fonts without a character map.
- **Fix:** before accepting a manual, check that the download is `application/pdf` and that the extracted text contains expected words (for example, "aceite", oil). The ones that failed were dropped.
- **Lesson:** validate data quality **before** ingesting it. Garbage in, garbage out.

## 4. The account blocks public buckets

- **Symptom:** the plan to serve the web app from a public S3 bucket was not viable.
- **Cause:** the account has S3 Block Public Access turned on at the account level.
- **Fix:** a private bucket served by CloudFront with Origin Access Control (OAC). As a bonus, it gives HTTPS to the phone.
- **Lesson:** the block was right; the design was what had to change.

## 5. A stopped `cdk deploy` kept deploying

- **Symptom:** after stopping a deploy, the next `cdk deploy` said *"Other CLIs are currently reading from cdk.out"*, and the stack showed up in `CREATE_IN_PROGRESS`.
- **Cause:** when the terminal was stopped, the CDK Node process stayed alive and kept deploying.
- **Fix:** let it finish (it was the same stack) and kill the orphan process to release the `cdk.out` lock.
- **Lesson:** stopping the CLI does not cancel CloudFormation. Check the stack status, not just the terminal.

## 6. Reorganizing the code forced a stack teardown

- **Symptom:** splitting the stack into one file per lane changed the CloudFormation logical IDs.
- **Cause:** with new IDs, CloudFormation creates the new resources before deleting the old ones, and the knowledge base and the guardrail have fixed names: they would collide.
- **Fix:** `cdk destroy` and `cdk deploy` from scratch, before any data was ingested. As a side effect, `destroy` got tested.
- **Lesson:** refactoring IaC is not free: logical IDs are part of the contract with the deployed resources.

## 7. `aws login` does not work with boto3

- **Symptom:** `ingestar.py` failed with *"Using the login credential provider requires an additional dependency"*.
- **Cause:** `aws login` credentials need botocore's `crt` extra. The AWS CLI ships it; boto3 does not.
- **Fix:** `uv add "boto3[crt]"`.
- **Lesson:** the CLI working does not guarantee that the SDK works with the same credentials.

## 8. Maintenance tables lost their columns

- **Symptom:** to "how often do I change the oil?" the assistant answered, correctly, that it could not find it.
- **Cause:** the default parser only extracts text. The maintenance chart row came out as `Aceite de motor / Cambiar / √√√√√√` ("Engine oil / Change / √√√√√√"): the check marks lost their kilometer column. **No chunking or embedding can recover what the parser already lost.**
- **Fix:** parsing with a vision model (Nova Lite), which reads each page as an image and transcribes tables to Markdown.
- **Cost:** tokens per page, only at ingestion; the README [cost estimate](../README.md#costs) puts it at about USD 0.26 per full ingestion at list price. Not yet confirmed against Cost Explorer.
- **Lesson:** parsing is the most underestimated decision in a RAG.

## 9. Changing the parser did not guarantee reprocessing

- **Symptom:** `cdk diff` showed that the data source would be updated in place.
- **Cause:** [syncing is incremental](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-data-source-sync-ingest.html): when a document has no content or metadata changes, the sync skips it. With the same PDFs, a parser change alone would not reprocess them. Also, the old data source had a `RETAIN` policy: its vectors would have stayed alive.
- **Fix:** a new data source with a versioned logical ID (`FuenteDatosV2`) and a `DELETE` policy. When it is replaced, the old one is deleted with its vectors.
- **Lesson:** changing how knowledge is processed forces reprocessing all of it, and it is best to force it explicitly.

## 10. Ingestion took more than 30 minutes

- **Symptom:** about 1,100 pages took between 33 and 40 minutes to ingest.
- **Cause:** the requests-per-minute quota for the parser model on a new account. The knowledge base waits between calls.
- **Fix:** none needed. Ingestion runs in AWS: the script only polls the status and can be closed.
- **Lesson:** time is not money here. You pay per token, not per minute.

## 11. Token metrics showed 0

- **Symptom:** the `AWS/Bedrock` token metrics showed 0 for Nova Lite and Titan after a full ingestion.
- **Cause:** not confirmed. The working hypothesis is that the model calls the knowledge base makes during ingestion are not published as account invocations in those metrics, but no AWS documentation was found that states this.
- **Fix:** measure the real cost in Cost Explorer (with a one-day delay) and cover the risk with the AWS Budgets budget.
- **Lesson:** know where each number comes from before promising it can be measured.

## 12. The parser invented a table on the index page

- **Symptom:** the top result for "how often do I change the oil?" was "page 89", with an oil change table and the symbols `√ ✓ ● X`. In the real PDF, page 89 is the **alphabetical index**.
- **Cause:** the parser prompt included a concrete example ("Engine oil: change at 500 km, 3000 km...") and a list of symbols. Faced with a page without tables, the model **copied the example as if it were content**.
- **How it was caught:** by checking the cited page against the source PDF.
- **Fix:** a prompt with no content examples and explicit rules: do not create tables that do not exist, transcribe the index as is and never copy the instructions. Full re-ingestion (see problem 9).
- **Verification:** after re-ingestion, page 89 no longer shows up and no chunk contains the example's symbols.
- **Lesson:** hallucinations also happen at **ingestion**, not only when answering. And an example in a prompt can turn into content.

## 13. The Lambda function was reachable without going through CloudFront

- **Symptom:** the Lambda Function URL was public (`auth NONE`): anyone with the URL could call it directly.
- **Cause:** the initial design put CloudFront only in front of the web app, not the API.
- **Fix:** CloudFront as the only door. `/*` goes to S3 and `/api/*` goes to the Lambda function, both with OAC. The Function URL requires a signature (`AWS_IAM`) and only CloudFront signs. For `POST` requests, the browser sends the SHA-256 of the body in the `x-amz-content-sha256` header, as Lambda requires. CORS also goes away, because everything is on the same domain.
- **Lesson:** the backend must not be reachable without going through the edge, which is where WAF and limits go.

## 14. The demo stayed exposed overnight

- **Symptom:** the web app stayed public while nobody was watching it, and every question spends tokens.
- **Fix:** a CDK switch (`--context web_activa=false`) that disables the distribution without deleting anything, and a permanent geographic restriction to Colombia.
- **Lesson:** turning it off costs nothing. For a demo, the cheapest control is not being exposed when nobody uses it.

## 15. The "take it to a mechanic" warning showed up too often

- **Symptom:** the warning appeared even for "how much oil does the engine take?".
- **Cause:** the rule looked for the word "aceite" (oil) in the question **and in the model's answer**.
- **Fix:** the rule fires only on **symptoms** in the question or the photo (red or lit warning lights, brakes, temperature, leaks, noises, smoke), not on what the model answers. Tested with 11 cases.
- **Lesson:** a warning that always shows up stops being useful.

## 16. A guardrail version that does not update itself

- **Symptom:** when rules were added to the guardrail, the Lambda function would have kept using the previous version.
- **Cause:** the `AWS::Bedrock::GuardrailVersion` resource is created once; changing the guardrail only changes the draft.
- **Fix:** changing the version description forces CloudFormation to create a new one. It is documented in `guardrail.py`.
- **Lesson:** read the `cdk diff` before deploying: it showed `replace` exactly where it had to.

## 17. Wrong diagnosis: an "invented" value that was actually there

- **Symptom:** the assistant answered *"the oil must be changed every 3000 km (p. 44)"*, and the conclusion was that "3000 km" was invented because the chunk seemed to contain only the check marks, without the kilometers.
- **Cause of the diagnosis mistake:** while inspecting the chunk, the text was trimmed around the word "aceite", and the header with the kilometers, higher up in the same chunk, was left out of view.
- **What the measurement showed:** with contextual grounding active, that answer scored 0.92: it was supported. Reading the full chunk revealed the header `500 km | 3000 km (1900 mi) | 6000 km | ...`.
- **The real problem:** the answer is supported but **incomplete**. The chart marks the first change at 500 km, and the answer leaves it out.
- **Lesson:** read the full evidence before concluding. And grounding measures that what is said is in the source, **not that it is complete**: completeness is checked by the golden set with expected values.

## 18. No alarm reached Slack

- **Symptom:** when an alarm was forced into the ALARM state, its history said *"CloudWatch Alarms is not authorized to perform: SNS:Publish"*. Nothing reached Slack.
- **Cause:** to let AWS Budgets publish to the SNS topic, a statement was added to the topic policy. That policy **replaces** the default SNS policy, which was what allowed CloudWatch to publish. Only the Budgets permission was left.
- **How it was caught:** by forcing an alarm by hand (`set-alarm-state`) and reading the action history, instead of assuming it worked.
- **Fix:** the topic policy explicitly declares CloudWatch and Budgets, limited to the account (`aws:SourceAccount`).
- **Verification:** with the fix deployed, the same forced alarm logs *"Successfully executed action"* to the SNS topic, where it used to say *"Failed to execute action"*, and the message reaches the Slack channel.
- **Lesson:** an alert that has never been tested is not an alert. Force the failure.

## 19. The golden set does not pass: 8 out of 10

- **What was done:** 10 questions with the answer verified by hand against the PDF, covering the 6 manuals, both languages, one safety case and three traps. See [`pruebas/`](../pruebas/).
- **Result:** 8 out of 10 in both runs. All three traps pass: wrong vehicle, outside the manual and injection. The threshold requires 9 out of 10. Between runs, the quality of the failure changed: the Pulsar case went from a "not found" disguised as an answer to saying clearly that it is not there.
- **Failures:** problems 20 and 21. Problem 22 came up while reviewing the answer details.
- **Lesson:** the golden set found in 30 seconds two failures that manual testing had missed. That is why it is the gate before any change.

## 20. Supported but incomplete answer

- **Symptom:** to "how often do I change the oil?" (Yamaha XTZ125) it answers "every 3000 km". The maintenance chart marks the first change at **500 km** and then every 3000.
- **Cause:** grounding passes (the value is in the source), but nothing forces the answer to include **all** the relevant values of the chunk.
- **Attempt 1:** the generator instructions were told to include all the relevant values of a table row (first interval and frequency). **It did not work:** the golden set failed again. Nova 2 Lite answered that the row marks "Change" in the 3000 km column, when the row has marks in all six columns. The model misreads the table; it is not an instructions problem.
- **Attempt 2:** move the generator up to Nova Pro, as the Inception rule says (move up only if the golden set shows the need). **It did not work either** and made another case worse (see problem 23).
- **Lesson:** grounding does not measure completeness; the expected values in the golden set do. And not everything is fixed with a prompt: when the golden set still fails after adjusting the instructions, the evidence justifies changing the model.

## 21. Manual pages that never reached the index

- **Symptom:** for "what oil do I use and how often do I change it?" (Bajaj Pulsar NS125) it does not find the value. The page with the specification (SAE 20W50, change every 10,000 km) exists in the PDF.
- **What was measured:**
  - The same question **in English** does not retrieve that page either, so the language is not the cause.
  - Asking for up to 100 chunks from that manual returns 58 chunks starting on 50 different pages. **10 pages have no chunk at all**, including the one with the oil specification.
  - Also, the search returns fewer chunks than requested (3 or 4 out of 8). The cause is not yet known.
- **Possible causes, to be confirmed:** the vision parser returned empty output for image-heavy pages, or the text of those pages ended up inside a chunk that starts on the previous page.
- **Pending:** review the parser output per page and, if confirmed, raise the number of retrieved chunks and validate during ingestion that every page produces text.
- **Lesson:** an ingestion that says "6 of 6 documents indexed" does not mean **every page** is in the index. Measure coverage per page.

## 22. A "not found" phrased differently passed as an answer

- **Symptom:** in the Pulsar case, the model answered *"I did not find specific information about which type of oil to use..."*, with no citations, and the app delivered it as a normal answer.
- **Cause:** the code detected "not found" by looking for the exact phrase from the instructions. The model said it in other words.
- **Fix:** any answer that does not cite any page is treated as "not found". It is a rule in code and does not depend on the model's wording.
- **Verification:** in the second golden set run, the same case answers *"I did not find that in your vehicle's manual."* (the app's fixed Spanish message). It still lacks the value (problem 21), but now it fails safely.
- **Lesson:** do not trust the model to repeat an exact phrase. Rules should check a structural signal (whether there are citations or not).

---

## 23. A more capable model answered worse

- **Hypothesis:** if Nova 2 Lite misreads the interval table (problem 20), a more capable model would read it correctly.
- **Experiment:** the generator was switched to Nova Pro (one line in `stack.py`), deployed, and the golden set was run.
- **Result:** 8 out of 10, as before, but with a **worse** failure:
  - The oil interval was still incomplete ("every 3000 km").
  - On the Pulsar, where Lite honestly answered "not found", Pro stated: *"Use SAE 90 and change it every 500 km (p. 46)"*. In the manual, SAE 90 and 500 km are **for chain care** (clean it with diesel and SAE 90, lubricate it with spray every 500 km), not for the engine oil.
- **Why grounding did not catch it:** both values are in the source; they are just attributed to another part. Grounding checks that the value exists in the source, **not that it is attributed to the right thing**.
- **Decision:** go back to Nova 2 Lite. Pro costs more (USD 0.80 / 3.20 per 1M input / output tokens versus USD 0.33 / 2.75 for Nova 2 Lite, us-east-1 public pricing checked on 2026-11-16; about twice as much per question with this prompt size), did not fix completeness and turned a safe failure into a dangerous one. It was confirmed with another run: back to 8 out of 10, with the Pulsar answering "not found".
- **Lesson:** a bigger model is not automatically safer. Changing the model is a change that goes through the evaluation gate, and the evaluation must be able to **revert** it. An honest "I don't know" is worth more than a confident wrong answer.

---

**Note on page numbers (observed in this corpus, not taken from AWS documentation):** each chunk carries a single page number (`x-amz-bedrock-kb-document-page-number`), and it matches the page where the chunk **starts**. If it starts at the end of a page, the citation can land one page before the value (in one case, the citation said 50 and the value is on page 51 of the PDF viewer). That is why the golden set accepts a one-page difference.
