# 1. Use Case for an Agentic System

**Application:** A medical app for families. Each user has a profile, and family members are linked to it.

**Problem:** Users talk naturally, not in commands. They mention symptoms in passing, send photos of prescriptions, ask medical questions, or may be in danger and unable to ask for help. A fixed menu or keyword system cannot handle this. The system needs to *understand* each message and decide what to do.

**Why it needs an agent:** A single message can require several different actions (answer a question, record a symptom, set a reminder, trigger an emergency flow), and the right action depends on meaning and context, not keywords.

| What the user says or sends | What the agent infers | What it does |
|---|---|---|
| "I can't sleep, it's been a week" | Symptom (insomnia, 1 week) | Gives general guidance from the knowledge base, saves it to the history as "reported by user" so the doctor sees it next visit |
| "My dad's sugar was high" | Condition about a family member | Confirms which family member, then records it |
| "My mom had hypertension, and I get headaches" | Family history plus personal symptom | Saves each in the correct place |
| Photo of a prescription | Medication event | Extracts drugs, asks the user to confirm, then creates dose reminders |
| Photo of lab results | Lab event | Extracts values, explains them against reference ranges |
| "What's the first-line treatment for type 2 diabetes?" | Medical question | Retrieves from the RAG, answers with citations |
| "My chest hurts and I can't breathe" | Emergency | Gives the emergency number, quick first steps, notifies the emergency contact |
| Elderly user stops replying after a worrying message | Possible incapacity | Repeated check-ins, then escalates to family |

## Architecture idea

A master agent analyzes each request first, then routes:

- Simple, structured, low-risk tasks go to a cheap worker model.
- Medical answers, complex cases, and routing decisions stay on the master model.
- Images go to a vision-capable model.
- RAG requests use sequential filters: condition, topic, semantic role, population.
- Emergency detection always includes code-level rules, never an LLM alone.
