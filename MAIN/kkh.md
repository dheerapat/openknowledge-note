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

Khon Kaen Hospital builds and runs its own hospital information system (HIS) through an internal IT team, keeping development and maintenance in-house instead of buying from a vendor.

- It self-hosts the open-source Qwen3.5 122B A10B model on local H200 hardware, controlling its own context rather than calling an outside provider.
- Its central advantage is the large body of data it holds.
- Its core use case is ICD10 code suggestion: a doctor may miss a code supported by the patient history, and each additional recorded code can unlock roughly a million in reimbursement.

The AI never acts on its own. It only offers suggestions and builds dashboards, while doctors retain the final decision.

The real work is the data pipeline, not the AI: moving the right data to the right place at the right time is what actually counts.

## Referral system

- When a hospital wants to refer an OPD patient, it submits the patient's data to a central system, and the receiving hospital works from that data.
- KKH loads all data from the central API into its HIS and uses AI to triage referral details, determining which department and ward should attend to the patient.
- Bottlenecks identified:
  - Generation is slow; a smaller model may be needed to produce faster predictions.
  - Triage quality depends directly on the quality of the referral detail.
  - Nurses have triage criteria and a training system, but how that information is folded into the AI context is not disclosed.

Feed
