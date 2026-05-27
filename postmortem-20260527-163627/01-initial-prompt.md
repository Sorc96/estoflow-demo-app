# 01 — Initial Prompt (Verbatim)

This session was a continuation of a previous context that was compacted. The session summary
describes the state at handoff. The initial user messages in this session (prior to the postmortem
prompt) are reproduced below as verbatim as the session summary captures them.

---

## Prior session messages (from compacted context summary)

The session continued from a previous conversation (krtekstop-v2, version 11, sci-fi technologies
already added). Four tasks were given in sequence:

### Task 1
> "Ještě uprav data v aktuálním projektu, aby využívala nově přidané sci-fi položky"

*(Translation: "Also update the data in the current project to use the newly added sci-fi items")*

### Task 2
> "Ještě uprav název firmy a jméno jednatele, aby to víc odpovídalo aktuální podobě aplikace"

*(Translation: "Also update the company name and director name to better match the current form of
the application")*

### Task 3
> "Můžeš vytvořit obrázek krtka jako povstaleckého vojáka ze star wars a použít ho jako logo ve
> výsledném pdf"

*(Translation: "Can you create an image of a mole as a rebel soldier from Star Wars and use it as
the logo in the resulting PDF")*

Follow-up clarification on Task 3:
> "zkus svg"

*(Translation: "try svg")*

### Task 4
> "Modely použité pro výpočty by měly mít asociovanou geometrii, která bude vidět ve výkresu"

*(Translation: "Models used for calculations should have associated geometry that will be visible in
the drawing")*

---

## Postmortem prompt

The final message of the session was the postmortem prompt provided by the user (the full
instructions for creating this bundle). See the verbatim text in `02-session.jsonl` — it is the
last human turn before the agent began writing these files.

Key directives from the postmortem prompt:
- Create bundle directory with timestamp
- Write 7 postmortem files (01 through 07)
- Copy session JSONL as final action
- Be honest about gaps; write "I don't recall — see 02-session.jsonl" rather than confabulating
