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

### Known weaknesses

- **Questions with no retrieval hit:** Question 3 ("From when can an employee make a request?") and question 6 ("good business reason") never placed the correct section in the top three. Both answers are in the Overview, but the search ranked Applying and Appeals passages higher.
- **Confusion between related sections:** Several Applying questions were ranked correctly in the top three but not at the top, and "After the application" was the most common rival.
- **Small sample:** 19 questions means one answer changes the score by about 5 percentage points. Treat the results as indicative, not definitive.
- **No answer generation yet:** This measures whether the right section is found, not whether the final answer is correct.
