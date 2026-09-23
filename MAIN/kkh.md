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

Khon Kaen Hospital develops and operates its own hospital information system (HIS) with an internal IT team, rather than licensing a vendor product.

- It runs the open-source Qwen3.5 122B A10B model on its own H200 hardware, so it controls the context instead of relying on an external provider.
- Its main asset is the large volume of data it accumulates.
- Its primary use case is ICD10 code suggestion: a physician may overlook a code that the patient history supports, and every extra recorded code can recover roughly a million in reimbursement.

The AI is advisory only — it suggests and generates dashboards, while physicians make the final call.

The real challenge is the data pipeline, not the model: delivering the right data to the right place at the right time is what matters.

## Demo session: referral system

- A hospital referring an OPD patient submits the patient's data to a central system, and the receiving hospital works from that submission.
- KKH pulls all data from the central API into its HIS, then uses AI to triage the referral and decide which department and ward should handle the patient.
- Bottlenecks observed:
  - Inference is slow; a smaller model might be needed to generate predictions faster.
  - Triage quality is only as good as the referral detail it is based on.
  - Nurses have triage criteria and a training system, but it is not disclosed how that knowledge is injected into the AI context.

## Feedback

- The system does relieve the nurses' triage burden by cutting down form filling and button clicking.
