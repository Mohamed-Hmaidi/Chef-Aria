# Architecture

How Chef Aria is put together, which model does what, which limits shaped the design, and how problems were solved. Numbers marked *(estimate)* are calculations, not measurements.

## Pipeline

Each request passes through these steps. A step can stop the request early, which saves the tokens and the image that later steps would use.

| Step | What happens | Done by |
|---|---|---|
| 0 | Empty or over-300-character input is rejected (no AI call) | Python |
| 1 | Security check: risk score for prompt injection and jailbreaks, blocked above 0.5 | Llama Prompt Guard 2 (86M), Groq |
| 2 | Topic check: is the request about food? YES or NO | gpt-oss-20b, Groq |
| 3 | Recipe package as JSON: reasoning, dish name, description, image prompt, ingredients, steps, history, 3 example reviews | gpt-oss-120b, Groq |
| 4 | Validation, rating clamped to 1 to 5, average rating, text formatting | Python |
| 5 | Dish picture from the image prompt written in step 3 | Pollinations.ai |
| 6 | Result shown; the button and textbox are locked while steps 1 to 5 run | Gradio |

Off-topic and empty requests show a smiling chef drawn with Pillow and a friendly message. If the image service fails, a placeholder card is drawn and the recipe is still shown.

## Models and why each was chosen

The size of each model is matched to the difficulty of its job. The cheap checks run first.

| Model | Why this one |
|---|---|
| Llama Prompt Guard 2 (86M) | A tiny classifier built for one job, detecting prompt attacks. It has no prompt and returns a score. Its free limits are the most generous. |
| gpt-oss-20b | The topic check is a one-word answer, so a large model is not needed. It has its own rate-limit budget, so it does not use up the main model's tokens. |
| gpt-oss-120b | The hardest job: one long answer in strict JSON that follows many rules at once (persona, review ratings matching a target, an image prompt). It is the largest model available on the account. |

These choices rest on size, purpose and rate limits. They were not benchmarked against the alternatives.

## Rate limits and capacity

Free-tier limits on the account used for development:

| Model | Requests / min | Requests / day | Tokens / min | Tokens / day |
|---|---|---|---|---|
| gpt-oss-120b | 30 | 1K | 8K | 200K |
| gpt-oss-20b | 30 | 1K | 8K | 200K |
| llama-prompt-guard-2-86m | 30 | 14.4K | 15K | 500K |

Pollinations without a key allows about one image per 15 seconds per IP (reported by several sources and observed in testing, not guaranteed).

One recipe uses roughly 2,000 to 2,500 tokens on the main model *(estimate)*: about 1,250 in and 700 to 1,300 out. That allows about 3 recipes per minute and about 80 per day *(estimate)*. Token limits, not request limits, are the constraint. A typical request takes 10 to 25 seconds, mostly because of the image *(estimate)*.

## Reliability

| Situation | Behaviour |
|---|---|
| Empty input | Chef picture and a greeting, no AI call |
| Over 300 characters | Plain message, no AI call |
| Prompt Guard score above 0.5 | "Flagged as unsafe" message |
| Topic check says NO | Chef picture and one of several friendly messages |
| A check itself fails (error, rate limit) | Fails open: the request continues, and the error is printed to the console |
| Groq rate limit, timeout, connection error | The client retries twice, then a friendly message |
| Malformed or incomplete JSON | Tolerant parsing, field validation, one retry at a lower temperature |
| Image throttled or failed | Waits, retries up to 3 times, then a placeholder card |
| Duplicate submit | Button and textbox locked until the result is shown |
| Any unexpected exception | Top-level handler shows a message instead of crashing |

Failing open means a broken safety check never blocks real users, but it also means a broken check protects nothing. This trade-off was chosen deliberately for a small project.

## Problems solved

| Problem | Cause | Solution |
|---|---|---|
| Image could not be displayed | Raw bytes were returned, but the interface needs an image object | Convert to a PIL image |
| Safety check never blocked anything | The first version looked for a text label, but Prompt Guard returns a number | Read the score and compare it with a threshold |
| JSON cut off mid-answer | Hidden reasoning tokens count toward `max_tokens` | Raised the cap, then tuned it to 2,500 for the 8K tokens/minute limit |
| Off-topic requests received dishes | The topic model was unavailable on the account (404) and the check failed open | Switched to gpt-oss-20b |
| Images rejected with HTTP 402 | Pollinations returns this for requests faster than about one per 15 s | Minimum 16-second wait after the last image, retries, placeholder |
| Duplicate requests | Repeated clicks or Enter presses | Locked the button and textbox, added a waiting message |
| Star ratings clustered around 4/5 | Models tend to write positive reviews | Random target average chosen by the code |
| Descriptions written in the third person | The model reasoned about "the user" | Reasoning addressed to "you", plus a rule against "the user" |
| Generic pictures | Image prompt was built from the dish name and description | The model writes a dedicated, detailed image prompt |

## Not in scope

- A permanent hosted deployment
- Verifying the historical accuracy of the generated text
- Real user reviews: the reviews are invented examples
