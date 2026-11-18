# OLJ Job Hunter: an n8n job-matching pipeline

An n8n workflow that finds new job postings on OnlineJobs.ph, filters them against my skills, scores the good ones with an LLM, drafts a tailored application for each strong match, and sends everything to Discord for me to review.

> **Human in the loop:** the workflow never applies to anything. It drafts and notifies, and I review every application before sending.

## How it works

```
Trigger → Fetch listings → Extract → Normalize → Remove duplicates
        → Fetch full descriptions → Rule-based filter
        → AI match scoring → IF score ≥ 70
        → AI application writer → Format → Discord
```

| Stage | n8n nodes | What it does |
|---|---|---|
| Fetch | HTTP Request, HTML | Loads the search results page and pulls each job's title and link with CSS selectors |
| Normalize | Split Out, Edit Fields | Turns one page of arrays into one item per job with a consistent shape (`title`, `url`, `source`) |
| Deduplicate | Remove Duplicates | Drops repeated postings by URL |
| Enrich | HTTP Request (batched), HTML | Fetches each job page and extracts the full description, with a delay between requests |
| Filter | Code, IF | Cheap keyword pass: must-have skills, blocked titles, "5+ years" detection. Runs before any AI call to save quota |
| Score | Basic LLM Chain, Gemini, Structured Output Parser | Compares the job to my resume and returns `score`, `verdict`, `gaps`, `red_flags`, `reason` as JSON |
| Write | Basic LLM Chain, Gemini | Drafts a tailored summary, resume bullets and a short cover letter using only facts from the resume |
| Notify | Code, Discord (webhook) | Splits output into messages under Discord's 2,000-character limit and posts them |

## Design decisions

- **Filter before AI.** The rule-based pass removes obvious mismatches, so the LLM only sees jobs worth reading. This saves rate limit and speeds up runs.
- **Structured output.** A JSON schema on both AI calls means downstream nodes get real fields (a numeric `score`), not free text.
- **Anti-fabrication prompt.** The writer is told to use only facts in the resume, and it returns a `claims_to_verify` list so I can check anything it stretched.
- **Prompt-injection guard.** Job descriptions are untrusted text, so both prompts tell the model to ignore instructions found inside them.
- **Polite scraping.** Requests are batched one at a time with a 1.5 second gap.

## Setup

**Requirements:** Docker Desktop, a Google AI Studio API key (Gemini), a Discord webhook.

```bash
docker volume create n8n_data
docker run -it --rm --name n8n -p 5678:5678 \
  -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

1. Open `http://localhost:5678` and create an owner account.
2. Import `workflow.json` (Workflows → Import from File).
3. Create credentials: **Google Gemini (PaLM) API** and **Discord Webhook**.
4. Open the **Resume** node and replace the placeholder text with your own resume as plain text.
5. Edit the search URL in the first HTTP Request node, and the keyword lists in the filter Code node.
6. Execute the workflow manually before enabling any schedule.

## Customizing

- **Keywords:** `mustHave`, `blockTitle` and `blockText` arrays in the filter Code node.
- **Score threshold:** the IF node after scoring (default 70).
- **Discord divider length:** the `repeat()` number in the formatting Code node.
- **Model:** any Gemini Flash model works for scoring and drafting.

## Limitations

- CSS selectors depend on OnlineJobs.ph's current page layout and will break if it changes.
- Rule-based filtering can miss good jobs or let junk through, so I tune the lists after reviewing the rejected branch.
- LLM scores are a screening aid, not a verdict. Always read the job post yourself.
- Check a site's terms of service before scraping it. This project is for personal use at a low request rate.

## Roadmap

- [ ] Schedule trigger, with deduplication that remembers jobs between runs
- [ ] Error-alert workflow that pings Discord on failure
- [ ] Additional sources (Gmail job alerts for LinkedIn, Indeed, Jobstreet) normalized into the same item shape
- [ ] Log applications to a spreadsheet

## What I learned

- Splitting arrays into per-item data, and reattaching fields lost between nodes
- Debugging n8n expressions and item pairing
- Structured LLM output and prompt design for scoring and writing
- Working within API and message-size limits

## Security

No credentials are stored in this repo. API keys and the Discord webhook live in n8n credentials, and the resume in `workflow.json` is a placeholder.
