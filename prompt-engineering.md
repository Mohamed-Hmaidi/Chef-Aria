# Prompt engineering notes

## Techniques used

| Technique | Where | Purpose |
|---|---|---|
| Role assignment (persona) | Main prompt: "You are Chef Aria..." | Consistent, warm cooking-expert voice |
| One-shot example | Main prompt: one complete Greek salad JSON | Shows the exact format, length and style |
| Few-shot examples | Topic-check prompt: six YES/NO examples | Handles borderline requests such as "a flying car" |
| Chain of thought | `reasoning` is the first JSON field | The model explains why a dish fits before naming it; shown to the user as "Why I picked it" |
| Structured output | JSON shape and rules, JSON mode in the API call, field validation, one retry | The code can split the answer into screen panels reliably |
| Prompt chaining | The main model writes an `image_prompt`, the code adds fixed style words and sends it to the image model | One model writes the prompt for another |
| Code-controlled variation | Python picks a random target average rating and passes it in the user message | Star ratings vary instead of clustering around 4/5 |

The topic check began as a zero-shot prompt (instruction only) and gained examples after borderline requests were misjudged.

## Who writes which prompt

| Goes to | Written by |
|---|---|
| Prompt Guard | The user's text, unchanged |
| Topic check | Fixed system prompt (instruction plus examples) and the user's text |
| Main generation | Fixed system prompt, plus a user message the code builds: the request and a random target rating |
| Image model | The `image_prompt` sentence written by the main model, plus a fixed style line |

## Parameters

| Parameter | Value | Effect |
|---|---|---|
| `temperature` (main) | 0.8 | Varied dishes and wording |
| `temperature` (retry) | 0.4 | Calmer second attempt after malformed output |
| `temperature` (topic check) | 0 | A classifier should be deterministic |
| `top_p` | 0.95 | Removes the very unlikely words before sampling |
| `max_tokens` (main) | 2500 | Room for hidden reasoning plus about 1,000 tokens of JSON, within the 8K tokens/minute limit |
| `max_tokens` (topic check) | 250 | One word plus a little reasoning |
| `reasoning_effort` | low | Simple tasks, so fewer tokens and less waiting |
| Rating target range | 2.8 to 4.9 | Random target average per request |
| Prompt Guard threshold | 0.5 | Scores above this are blocked |

Rough guide to temperature: low values (0 to 0.3) give predictable, repeatable answers, medium values (0.4 to 0.8) give variety that stays sensible, and high values (above 1) become increasingly random.

## Temperature experiment

Request: "something light for dinner", three runs per temperature, same prompt.

| Temperature | Dishes returned | Observation |
|---|---|---|
| 0.2 | Citrus Shrimp & Avocado Salad (3 times) | No variety: the same dish every run |
| 0.8 | Citrus Quinoa Salad; Lemon-Herb Quinoa Veggie Bowl; Lemon-Herb Grilled Salmon with Cucumber-Dill Salad | Three different dishes, all relevant |
| 1.2 | Lemon-Herb Quinoa Salad; Lemon-Herb Quinoa Salad with Grilled Vegetables; Citrus Quinoa Salad | Valid output, but not more varied than 0.8 |

All nine runs returned valid JSON. Three runs per setting is a small sample, so these are patterns and not proof. One possible explanation for 1.2 not being more varied is that `top_p` 0.95 limits how random the output can get; this was not tested.

## Open questions

- **Example bias:** the main prompt's single example is a light salad, and "light dinner" requests returned citrus and quinoa salads in every run. The example may be pulling results toward salads. Testing a second, very different example is a possible next step.
- **Reasoning effort:** only `low` was used. Whether `medium` improves dish choices enough to justify the extra tokens was not measured.
