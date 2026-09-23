---
title: Khon Kaen Hospital
description: Notes on Khon Kaen Hospital's in-house HIS, internal IT team, local model serving on H200, and AI-as-suggestion approach.
tags:
  - healthcare
  - hospital
  - llm
  - qwen
---
# Khon Kaen Hospital

Khon Kaen Hospital runs everything itself: its own hospital information system (HIS), built and maintained by an in-house IT team rather than bought from a vendor.

- Hosts an open-source model — Qwen3.5 122B A10B — on local H200 hardware, shaping the context itself instead of calling an outside provider.
- Sits on a large body of data, which is its central advantage.
- The main point is ICD10 code suggestion, doctor may miss some icd10 code base on patient history, 1 recorded icd10 code help generate like a million in reimburse.

Its AI never decides anything on its own. It suggests, and it builds dashboards, while the doctors keep the final call.

The real work is the data pipeline, not the AI: getting the right data to the right place at the right time is what counts.

## Refereal system

- When hospital want to refer OPD patient to the hospital, they will apply patient data into central system and destination hospital will get the data to work on
- KKH hospital will load all data from central api into HIS, and use AI to triage  from refer detail which department and ward to attend
- bottleneck found
  - Slow generation, maybe model need to be smaller so can generate faster prediction.
  - Do ease a burden on nurse that do a triage, reduce form filling and button clicking.
  - Quality of triage is direct impact from referal detail
  - Nurse do have a criterial, and training system to train personnel on triage, but detail on use that info into AI context is not disclose.
