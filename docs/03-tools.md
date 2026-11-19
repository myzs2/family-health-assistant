# 3. Tools

## Knowledge base (RAG)
| Tool | Purpose |
|---|---|
| `get_section_index(condition_group)` | Returns the available topics so the model picks from a closed list instead of inventing |
| `search_medical_kb(query, condition_group, topic, semantic_role[], population, top_k)` | Filtered hybrid search over medical chunks |
| `lookup_drug_info(drug_name, population)` | Dose, contraindications, interactions |
| `lookup_lab_reference(test_name, age, sex)` | Normal ranges |

## Patient and family records
| Tool | Purpose |
|---|---|
| `get_patient_profile()` | Demographics, chronic conditions, allergies, current medications |
| `get_family_members()` | Linked family members and relationships |
| `get_medical_history(person_id)` | Existing history |
| `log_symptom(person_id, symptom, duration, severity, user_words)` | Saves a symptom, flagged as user-reported and unconfirmed |
| `add_history_entry(person_id, type, details)` | Diagnoses or conditions (requires confirmation) |
| `add_family_history(relative_id, condition)` | Family medical history (requires confirmation) |

## Images and reminders
| Tool | Purpose |
|---|---|
| `extract_prescription(image)` | Returns drug, dose, frequency, duration as JSON |
| `extract_lab_report(image)` | Returns test, value, unit, reference range as JSON |
| `create_dose_reminders(items[])` | Creates reminders (only after the user confirms the extraction) |

## Emergency and follow-up
| Tool | Purpose |
|---|---|
| `get_emergency_info(country)` | Local emergency number and nearest hospital |
| `notify_emergency_contact(message, severity)` | Contacts the saved emergency contact (requires prior opt-in) |
| `schedule_checkin(person_id, interval, max_attempts)` | Schedules repeated "are you okay?" messages |
| `escalate_if_no_response(checkin_id)` | Escalates to contacts if no reply |

## Worker delegation
| Tool | Purpose |
|---|---|
| `delegate_to_worker(task, input, schema)` | Sends a low-risk structured task (query expansion, symptom extraction, formatting) to the cheap worker model and validates the JSON output against the schema |

## Safety rules on tools
- Self-reported symptoms are saved automatically, with a visible notice and an undo option.
- Anything about another person, any diagnosis, any family history, and any reminders require confirmation first.
- `notify_emergency_contact` runs automatically only if the user opted in beforehand.
- No writing to another person's record without permission.
