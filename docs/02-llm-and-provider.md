# 2. Choose LLM and Identify Provider

## Strategy: hybrid (enterprise master + cheap open-weight workers)

**Master agent:** OpenAI (GPT-5 class model via the OpenAI API or Azure OpenAI for enterprise terms).
**Worker models:** an open-weight model such as GPT-OSS-120B, served by Groq.

Model names and prices change often. Verify them on each provider's pricing page before committing.

## Why this split

The master makes the high-stakes decisions: emergency screening, request analysis, routing, and the final medical answer. The workers handle high-volume, low-risk tasks where speed and cost matter more than peak quality.

| Role | Model | Provider | Reason |
|---|---|---|---|
| Master agent (analysis, routing, final medical answer, complex cases) | GPT-5 class (enterprise tier) | OpenAI / Azure OpenAI | Strong reasoning and tool use, enterprise data agreements |
| Query expansion and RAG filter selection | GPT-OSS-120B | Groq | Fast, cheap, structured task (pick from closed lists) |
| Symptom extraction and history logging | GPT-OSS-120B | Groq | Structured JSON extraction, low risk |
| Small talk, formatting, summaries | GPT-OSS-120B | Groq | No need for a premium model |
| Prescription and lab image reading | Vision-capable model | OpenAI | GPT-OSS-120B is text-only |
| Emergency screen | Rules in code + master model | Own backend + OpenAI | Never rely on a cheap model alone |

## Important points

1. **Vision:** GPT-OSS-120B does not read images. Use a vision-capable model for prescriptions and labs, where accuracy matters most.
2. **Emergency safety:** keep keyword and rule checks in your own code.
3. **Privacy:** health data goes to third-party providers. Check data retention, training policy, and DPAs for each. Review Groq's terms separately.
4. **Two vendors, two failure points:** add fallbacks (if Groq is down, route workers to a cheaper OpenAI model, and the reverse).
5. **Quality gate:** the final patient-facing medical answer stays on the master. Workers are used only for steps the user never reads directly, or where output is validated (JSON schema, closed-vocabulary checks).

## Cost control

- Master is called only for analysis and final answers.
- Workers handle roughly everything else.
- Cache the system prompt and repeated question analyses.
- Retrieve only the top 5 to 8 chunks.
- Use batch processing for non-urgent work such as pre-visit history summaries.

## Flow

```
User message
   |
Rule-based emergency check (code)
   |
Master agent: analyze request, choose route
   |
   +-- simple / structured task --> GPT-OSS-120B on Groq
   +-- RAG --> worker picks filters from closed lists --> retrieve --> master writes answer
   +-- image --> vision model --> extraction --> user confirms
   +-- emergency --> protocol (code + master)
```
