# React Fallacifier Extension Server

Backend service for the **React Fallacifier browser extension**.  
This server receives a user claim, gathers web search evidence, computes source trust scores, and asks an LLM to return a structured fact-check-style verdict.

---

## Table of Contents

- [Overview](#overview)
- [What This Server Does](#what-this-server-does)
- [Repository Structure](#repository-structure)
- [Tech Stack](#tech-stack)
- [High-Level Request Flow](#high-level-request-flow)
- [API Reference](#api-reference)
  - [GET /activate](#get-activate)
  - [POST /claim](#post-claim)
- [Environment Variables](#environment-variables)
- [Setup & Installation](#setup--installation)
- [Running the Server](#running-the-server)
- [Trust Scoring Logic](#trust-scoring-logic)
- [Prompting / LLM Behavior](#prompting--llm-behavior)
- [Data Storage (Firebase Realtime Database)](#data-storage-firebase-realtime-database)
- [Expected Request/Response Examples](#expected-requestresponse-examples)
- [Error Handling Notes](#error-handling-notes)
- [Current Limitations](#current-limitations)
- [Security Notes](#security-notes)
- [Performance Notes](#performance-notes)
- [Development Notes & Suggested Improvements](#development-notes--suggested-improvements)
- [Deployment Notes](#deployment-notes)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Overview

This project is a Node.js + Express backend (100% JavaScript) that powers claim analysis for the Fallacifier ecosystem.

Core capabilities:

1. Accept a claim from a client.
2. Search the web (Brave Search API) for relevant sources.
3. Assign each source a **domain trust score** (cached in Firebase Realtime Database).
4. Construct a structured prompt containing source snippets + trust metadata.
5. Send the prompt to an NVIDIA-hosted LLM endpoint (via OpenAI SDK-compatible API).
6. Return a parsed JSON response and top-ranked sources to the client.

---

## What This Server Does

When a client sends a claim to `/claim`, the server:

- pulls web results for that claim,
- computes `Trust_score` for each result URL,
- sorts results descending by trust,
- generates a long-form analysis prompt,
- asks the model for a machine-readable JSON verdict,
- repairs malformed JSON if needed (`jsonrepair`),
- returns the final structured payload with `Top_Sources`.

---

## Repository Structure

```text
.
├── .env.example         # Sample environment variables
├── bravesearch.js       # Brave Search integration + result shaping + trust scoring per URL
├── givescores.js        # Domain trust scoring logic + Firebase cache (get or create score)
├── gpt.cjs              # LLM call wrapper (NVIDIA endpoint through OpenAI SDK)
├── server.js            # Express app + routes + orchestration
├── package.json
└── package-lock.json
```

---

## Tech Stack

- **Runtime:** Node.js
- **Server Framework:** Express `^5.1.0`
- **CORS:** `cors`
- **Environment Management:** `dotenv`
- **Search Provider:** Brave Search API
- **LLM Client:** OpenAI Node SDK targeting NVIDIA base URL
- **Model Endpoint:** `mistralai/mistral-nemotron` (as configured)
- **JSON Recovery:** `jsonrepair`
- **Domain/TLD Parsing:** `tldts`
- **Data Store:** Firebase Realtime Database (compat SDK)

---

## High-Level Request Flow

1. Client sends `POST /claim` with body:
   ```json
   { "claim": "..." }
   ```

2. `server.js` calls `searches(claim)` from `bravesearch.js`.

3. `bravesearch.js`:
   - requests Brave search results,
   - maps each result URL through `givescores.js`,
   - returns normalized objects:
     - `domain`
     - `link`
     - `title`
     - `content`
     - `Trust_score`

4. Results are sorted by trust score.

5. `makeprompt(...)` builds a large instruction prompt.

6. `gpt.cjs` sends prompt to NVIDIA API and receives completion text.

7. Server attempts:
   - `jsonrepair(response)`
   - `JSON.parse(...)`

8. Server responds with parsed AI output + `Top_Sources` (top 3 trusted links).

---

## API Reference

### GET /activate

Health-check / keep-alive endpoint.

**Response (text):**
```text
The server is Now active!
```

---

### POST /claim

Accepts a user claim and returns analysis output.

#### Request Body

```json
{
  "claim": "The earth is flat"
}
```

#### Success Behavior

- Searches the web for evidence.
- Scores each source.
- Sends combined evidence to LLM.
- Returns parsed response object with `Top_Sources`.

#### Response Shape (inferred)

Since LLM fields are model/prompt-dependent, response includes:
- model-generated keys from parsed JSON
- mandatory server-added key:
  - `Top_Sources`: array of top 3 scored sources

---

## Environment Variables

From `.env.example`:

```dotenv
# Brave Search API
Brave_API_KEY=

# Nvidia API
Nvidia_API_KEY=

# Firebase Configuration
firebase_API_KEY=
firebase_AuthDomain=
firebase_ProjectID=
firebase_StorageBucket=
firebase_messagingSenderId=
firebase_AppID=
firebase_measurementId=
firebase_databaseURL=
```

### Notes

- `Nvidia_API_KEY` is used as API key in OpenAI SDK client with:
  - `baseURL: https://integrate.api.nvidia.com/v1`
- Firebase vars are required for score caching reads/writes.
- Missing env vars can lead to runtime failures in API calls or DB initialization.

---

## Setup & Installation

### 1) Clone

```bash
git clone https://github.com/AkshaTGA/React-Fallacifier-Extension-Server.git
cd React-Fallacifier-Extension-Server
```

### 2) Install dependencies

```bash
npm install
```

### 3) Configure environment

Create `.env` in project root (copy from `.env.example`) and fill all required keys.

```bash
cp .env.example .env
```

Then edit `.env` with real credentials.

---

## Running the Server

There is currently no `scripts.start` in `package.json`, so run directly:

```bash
node server.js
```

Server listens on:

- **Port:** `8080`
- **Base URL (local):** `http://localhost:8080`

---

## Trust Scoring Logic

Implemented in `givescores.js`:

### Score Inputs

- Protocol bonus:
  - `https:` → `+1`
- TLD-based weighting:
  - High-trust: `.gov`, `.gov.in`, `.ac.uk`, `.int`, `.mil`, `.nic.in` → `+6`
  - `.edu` → `+3`
  - `.org` → `+2`
  - `.com`, `.in`, `.dev`, `.io` → `+1`
  - suspicious/low-signal (`.xyz`, `.abcd`, `.tk`, `.ml`, `.cf`) → `-1`
  - unlisted suffixes → `-1`
- Deep subdomain penalty:
  - more than 4 dot segments in host/domain string → `-1`
- Keyword penalty in domain:
  - if domain matches `(blog|blogspot|wordpress|weebly|free|host|unofficial|altnews|clickbait)` → `-2`

### Score Clamp

Final score is clamped to:

- minimum `0`
- maximum `10`

### Caching Strategy

- Domain key is transformed by replacing `.` with `_`.
- Score path: `domains/<domainKey>` in Firebase Realtime Database.
- On lookup:
  - if score exists -> return cached value
  - else compute score, persist, then return

---

## Prompting / LLM Behavior

`makeprompt(...)` in `server.js` builds a custom prompt containing:

- original claim
- enumerated evidence snippets
- each source’s `domain`, `title`, `content`, `DomainTrustScore`
- explicit instructions for verdicting and source mention behavior

The prompt appears designed to force structured JSON output, later parsed and repaired on server side.

---

## Data Storage (Firebase Realtime Database)

This project stores **domain-level trust scores** as reusable cache values.

### Benefits

- avoids recomputing score for frequently seen domains,
- keeps scoring deterministic for repeated requests,
- lowers compute overhead during claim processing.

### Stored Data Shape (conceptual)

```json
{
  "domains": {
    "https://example_com": 4,
    "https://wikipedia_org": 3
  }
}
```

---

## Expected Request/Response Examples

### Example Request

```bash
curl -X POST http://localhost:8080/claim \
  -H "Content-Type: application/json" \
  -d '{"claim":"Drinking water cures all diseases"}'
```

### Example Response (illustrative)

```json
{
  "Verdict": "Refuted",
  "Confidence": "Medium",
  "Reasoning": "Available evidence does not support universal cure claims.",
  "Mention": "yes",
  "Top_Sources": [
    {
      "domain": "who.int",
      "link": "https://www.who.int/...",
      "title": "...",
      "content": "...",
      "Trust_score": 7
    },
    {
      "domain": "nih.gov",
      "link": "https://www.nih.gov/...",
      "title": "...",
      "content": "...",
      "Trust_score": 7
    },
    {
      "domain": "example.org",
      "link": "https://example.org/...",
      "title": "...",
      "content": "...",
      "Trust_score": 3
    }
  ]
}
```

---

## Error Handling Notes

Current behavior in `server.js` includes:

- parse failure path:
  - responds with text: `"parsing failed" + err.message`
- LLM call error path:
  - checks if message includes `400`
  - logs guidance message to console
  - does not always send structured error response to client

Brave/Firebase failures bubble via promise failures and may produce unhandled behaviors if not fully wrapped.

---

## Current Limitations

1. `package.json` has no `scripts` for start/dev/test.
2. No explicit validation for missing `claim` field.
3. No request timeout/retry logic for external APIs.
4. No authentication/rate-limiting on public endpoints.
5. No structured error schema for clients.
6. Prompt is monolithic and hard-coded in `server.js`.
7. No tests (unit/integration/e2e) included.
8. Caching strategy only stores heuristic score, not freshness metadata.

---

## Security Notes

- Never commit `.env` or credentials.
- Restrict CORS origin list in production.
- Add rate limiting (`express-rate-limit`) to protect `/claim`.
- Validate and sanitize request input.
- Use secret management in production (not plain env files when possible).
- Consider response filtering to avoid exposing unintended model output.

---

## Performance Notes

- Each claim currently triggers:
  - one Brave query,
  - up to 10 trust-score lookups/updates in Firebase,
  - one LLM completion call.
- End-to-end latency is dominated by external network/API calls.
- Potential optimization:
  - parallelize/restrict search count,
  - add claim-level memoization,
  - add timeouts + fallback behavior.

---

## Development Notes & Suggested Improvements

### Recommended `package.json` scripts

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "node server.js"
  }
}
```

### Recommended enhancements

- Add input schema validation (e.g., `zod`, `joi`, or manual guardrails).
- Add centralized error middleware.
- Return consistent JSON errors:
  ```json
  { "error": { "code": "PARSE_ERROR", "message": "..." } }
  ```
- Move prompt template to dedicated module/file.
- Add logging with request IDs.
- Add tests:
  - `giveurlscores` unit tests
  - `/claim` integration tests (mock Brave + LLM + Firebase)
- Pin Node version via `.nvmrc` and engines field.

---

## Deployment Notes

For production deployment:

1. Set all environment variables in hosting provider.
2. Ensure outbound HTTPS access to:
   - Brave API
   - NVIDIA API
   - Firebase Realtime Database
3. Set process manager or platform health checks to `/activate`.
4. Run server with:
   ```bash
   node server.js
   ```
5. Expose port `8080` or map host port accordingly.

---

## Troubleshooting

### Server starts but `/claim` fails

- Verify `Brave_API_KEY` and `Nvidia_API_KEY`.
- Verify Firebase credentials and DB URL.
- Confirm network egress not blocked by host environment.

### JSON parsing failures from model output

- Check logs for malformed response.
- Consider stricter prompt JSON constraints.
- Keep `jsonrepair` but add fallback schema validation.

### Trust scores not persisting

- Verify Firebase Realtime Database rules and URL.
- Confirm service/project config is correct.
- Check if initialized app credentials match target project.

---

## License

No explicit license file detected in this repository.  
If this is intended for open-source use, add a `LICENSE` file (e.g., MIT/Apache-2.0) and update this section accordingly.
