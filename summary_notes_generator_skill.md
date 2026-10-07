/create-skill

Create a user-global skill called "summary-notes-generator" (install it to ~/.copilot/skills/ so it works across all my repos, not just one project).

PURPOSE
It converts a pasted or attached video/lecture transcript into a structured shorthand .txt notes file in my established note-taking format. It should trigger when I paste/attach a transcript with no other instruction, or when I say "notes for this", "this one too", or "same as before". It must NOT trigger if I ask to summarize, explain, or quiz on the transcript instead — in those cases do what I asked. When it does trigger, produce the file directly without asking me any clarifying questions.

DUPLICATE CHECK
Some pasted transcripts repeat the same content 2–3 times (copy-paste artifacts). If that happens, silently use only one copy, then mention it once in a single line after delivering the file (e.g. "heads up, your transcript repeated 3x, notes cover it once").

FILE STRUCTURE
- Break the transcript into 4–8 logical sections based on its own topic shifts — don't force a fixed count. Use the transcript's own thematic breaks (intro/concept, how X works, component A, component B, use case, takeaway, etc.).
- Each section header is short, plain text — no markdown #, no bold.
- At the very top of the file only (not per section), include this exact Convention line:
  Convention: => causes/results in | : contains/includes | * = interview-worthy / important
- Under each section header: bullet points using "-" that compress and paraphrase (never copy sentences verbatim). Use 2-space-indented sub-bullets where the transcript itself nests a concept under a parent idea.
- End EVERY section with this exact block:
  MY SUMMARY:
  _____________________________________________________________________________
  _____________________________________________________________________________
  (two underscore rules — no more, no less)

SHORTHAND RULES
- "=>" only means causes/leads to/results in. Never use it for includes/contains.
- ":" is for listing sub-items or examples inline (e.g. "DevOps: CI/CD, repo mgmt, infra automation").
- "*" prefixes a bullet or section header that's genuinely important (interview-worthy / real-work-worthy). Use sparingly: 1–3 stars per document, on the highest-value points only (core differentiators, key mechanisms, the one concrete end-to-end example).
- Compress ruthlessly: one line per idea. A whole paragraph becomes one bullet in my own words.
- Fix obvious transcription typos in technical terms while compressing.
- If the transcript has one standout concrete example (a real scenario, numbers, an analogy), always keep it as its own section and mark it with "*" — don't let it get lost in compression.

OUTPUT
- Always a downloadable .txt file, no markdown inside the file.
- Name the file descriptively from the transcript's actual topic (e.g. MCP_Architecture_notes.txt), never generic names like notes.txt.
- No title/header/date/attribution block in the file beyond the topic name and the Convention line — keep it lean.
- After sending the file, reply with one short line on how it's structured (how many sections, what's starred). Don't repeat the content back.
- Chat tone: casual, direct, no filler intro, don't repeat my request back. One or two sentences.
