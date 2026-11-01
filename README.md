## Data and licence

The source documents are taken from GOV.UK, the official UK government website.

- **Source:** GOV.UK guidance on flexible working: [Flexible working](https://www.gov.uk/flexible-working), including its sections on types of flexible working, applying, the application process, and appeals. The text was taken from the [printable version](https://www.gov.uk/flexible-working/print).
- **Retrieved:** 01/11/2026
- **Licence:** Crown copyright, published under the [Open Government Licence v3.0](https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/)

The project does not include any personal data or non-public government information.

# Flexible Working Q&A

A question-answering tool that answers questions about UK flexible working rules using GOV.UK guidance, and shows which section of the guide each answer comes from.

## Problem and audience

Flexible working rules are spread across several pages of official guidance, covering who can ask, how to apply, what an employer can refuse, and how appeals work. Finding a specific answer means reading across those pages and checking the details yourself.

This project is for employees and HR staff who want a quick, sourced answer to a specific question about their rights or responsibilities. It's also a working example of how retrieval can be evaluated before adding a language model on top.

## Evaluation

### Question set

The evaluation set in `data/eval.csv` has 19 questions drawn from the GOV.UK flexible working guide. Each question has a known answer and the section where that answer appears. Questions cover all five sections: Overview (8), Types of flexible working (3), Applying for flexible working (5), After the application (0), and Appeals (3).

### Method

Each question is run through retrieval. The top-1 score checks whether the first retrieved passage comes from the expected section. The top-3 score checks whether the expected section appears anywhere in the top three passages. MRR (mean reciprocal rank) averages the inverse of the rank at which the expected section first appears. NDCG@3 is a discounted ranking score over the top three results, with one relevant section per question. The index is built with `all-MiniLM-L6-v2` embeddings in Chroma. Each chunk size is logged as an MLflow run.

### Chunk size comparison

| Chunk size (words) | Overlap | Chunks | Top-1       | Top-3       | MRR   | NDCG@3 |
|--------------------|---------|--------|-------------|-------------|-------|--------|
| 120                | 20      | 14     | 68% (13/19) | 89% (17/19) | 0.789 | 0.817  |
| 200                | 40      | 10     | 58% (11/19) | 89% (17/19) | 0.728 | 0.771  |
| 350                | 50      | 5      | 63% (12/19) | 95% (18/19) | 0.763 | 0.810  |

The differences are one or two questions out of 19, so no chunk size is clearly better on this set. The 120-word setting with 20 words of overlap is the default, since it leads on top-1, MRR, and NDCG@3. The 350-word setting has the highest top-3 rate, so it may suit a later step that uses several passages at once.

Full results are in [results/chunk_size_comparison_v2_19q_mrr_ndcg.xlsx](https://github.com/rohinerd/document-qa-rag/blob/main/results/chunk_size_comparison_v2_19q_mrr_ndcg.xlsx). The earlier 11-question run is kept in `results/` for reference.

### Answer step

Answers are generated with Gemini (`gemini-flash-latest`) from the top three retrieved passages, with a prompt that asks the model to answer only from those passages and cite the section in square brackets.

The answer step was scored by hand on all 19 evaluation questions. Judgements were made against the GOV.UK guide.

| Metric | Result |
|---|---|
| Answer accuracy (substantively correct) | 84% (16/19) |
| Citation accuracy (correct section cited) | 89% (17/19) |

Two answers were refused because retrieval missed the source passage (questions 3 and 6). One answer (question 18) was cut off before its final list item. Scores were logged to MLflow in the `answer-step-test` experiment.

This is a manual check on a small set, not a benchmark. The Overview judgements were made from earlier extracts and should be rechecked against the live page.

### Known weaknesses

- **Questions with no retrieval hit:** Question 3 ("From when can an employee make a request?") and question 6 ("good business reason") never placed the correct section in the top three. Both answers are in the Overview, but the search ranked Applying and Appeals passages higher. In the answer step, this caused both refusals.
- **Confusion between related sections:** Several Applying questions were ranked correctly in the top three but not at the top, and "After the application" was the most common rival.
- **Answer step:** Two refusals trace back to retrieval misses, and one answer was truncated. Scoring was manual, on 19 questions, so the figures are indicative.

## Architecture

The pipeline has five stages:

1. **Load:** the GOV.UK guide is saved as plain text, with navigation and cookie banners removed.
2. **Split by section:** the text is divided at its five headings, so every chunk knows which section it came from.
3. **Chunk:** each section is split into overlapping windows of words (120 words with 20 overlap by default).
4. **Embed and index:** each chunk is converted to a vector with `all-MiniLM-L6-v2` from sentence-transformers, and stored in Chroma with its section label.
5. **Retrieve:** a question is embedded the same way, and the closest chunks are returned.

The embedding model is a third-party open model, downloaded automatically when the notebook runs. It was chosen because it's small enough to run in a free Colab session and produces good results for English text. The model isn't stored in this repository.

Answer generation with a language model and citations is not yet built.

## How to run

The project runs in Google Colab, and needs no local installation or cloud account.

1. Open the RAG notebook in Colab.
2. Run the cells in order from the top: install the libraries, load the guide text from this repository, split and chunk it, build the Chroma index, and score retrieval against `data/eval.csv`.
3. Run the MRR and NDCG cell and the comparison cell to reproduce the chunk-size results. Runs are logged to MLflow in the notebook session.

The guide text is loaded from `data/flexible-working.txt`, and the evaluation questions are in `data/eval.csv`. Colab sessions reset, so rerun the cells in order if you return later.

## Limitations

- **Not legal advice.** The tool answers from the GOV.UK guidance only. It doesn't interpret an individual's circumstances, and it isn't a substitute for advice from Acas, a solicitor, or an employment adviser.
- **Narrow scope.** It covers one GOV.UK guide, and answers may be incomplete where the guide points to other pages.
- **Point-in-time text.** The guide was saved on 1 November 2026. Changes to the GOV.UK page won't appear until the text is updated.
- **Retrieval only.** Until answer generation is added, the project shows whether the right section is found, not whether a generated answer is correct.
- **Small sample:** 19 questions means one answer changes the score by about 5 percentage points. Treat the results as indicative, not definitive.
- **No answer generation yet:** This measures whether the right section is found, not whether the final answer is correct.
