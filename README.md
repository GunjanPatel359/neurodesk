# NeuroDesk

LLM-driven ticket assignment for a service desk. A ticket arrives, the system reads it,
works out which technical skills the work actually needs, picks the technician best suited
to it, and then measures whether that was the right call.

```
ticket ──► skill extraction ──► technician selection ──► assignment
             (Gemini)              (Gemini)                  │
                                                             ▼
                                              resolution, SLA + sentiment
                                                             │
                           technician skill ratings ◄── evaluation
```

The loop at the bottom is the point: assignment quality is scored after the fact and fed
back into each technician's skill ratings, so the next selection is made on measured
performance rather than on what the profile claims.

## How it works

**1. Skill extraction** — the ticket's subject and body go to the model with the catalog of
skills the organisation already tracks. It returns the skills this ticket requires, and may
propose a new skill when nothing in the catalog fits.

**2. Technician selection** — candidates are filtered to those available, then the model
chooses between them against the required skills and returns the chosen technician with its
reasoning, so a human can see why.

**3. Evaluation** — once a ticket is resolved, the evaluation service computes resolution
time against the SLA target for that priority, runs sentiment analysis over customer
feedback, scores per-skill performance, and updates the technician's skill ratings.

## Stack

| Layer | Built with |
|---|---|
| AI service | Python, Flask, LangChain, Google Gemini 2.5 Flash, Pydantic |
| Web app | Next.js (App Router), TypeScript, Tailwind, Radix UI |
| Data | PostgreSQL via Prisma |

Tickets, skills and technicians are modelled as typed Pydantic objects mirroring the Prisma
schema, so the AI service and the web app agree on shape.

## Layout

```
ai-backend2/          the LLM service
  app.py              Flask API
  models/             ticket, skill, technician
  services/           skill_extraction, technician_selection, assignment, evaluation
nextfrontend/         Next.js app — admin, technician and end-user areas
  prisma/             schema and seed
```

## API

| Method | Route | Does |
|---|---|---|
| `POST` | `/api/ticket-assignment` | extract skills, select a technician, return the choice and the reasoning |
| `POST` | `/api/evaluate-technician` | score a resolved ticket and update skill ratings |
| `GET` | `/health` | liveness |

## Running it

The AI service:

```sh
cd ai-backend2
pip install -r requirements.txt
cp sample.env .env        # add GOOGLE_API_KEY
python app.py             # :5000
```

The web app:

```sh
cd nextfrontend
pnpm install
pnpm prisma migrate dev
pnpm dev                  # :3000
```

## Status

A working prototype, not production software. The assignment and evaluation paths run end to
end; the model is reached through LangChain, so swapping Gemini for another provider is a
configuration change.
