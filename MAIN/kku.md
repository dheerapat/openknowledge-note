---
title: Khon Kaen University
description: Notes on KKU Critical Care's "From Bedside Data to AI" initiative — ICU data sources, the Smart ICU mission, and the data-to-care pipeline.
tags:
  - healthcare
  - icu
  - ai
---
# From Bedside Data to AI — KKU Critical Care

KKU Critical Care's initiative to turn high-frequency ICU data into AI-assisted patient care. The medical ICU already generates a massive volume of data every minute; the goal is to capture, integrate, and act on it.

## Medical ICU data sources

The medical ICU streams data continuously from:

- **Bedside monitors** — heart rate (HR), blood pressure (BP), oxygen saturation (SpO2), respiratory rate (RR), temperature, and ECG
- **Ventilators**
- **Infusion pumps**
- **CRRT** (continuous renal replacement therapy) and urinalysis
- **ECMO** (extracorporeal membrane oxygenation)
- **IABP** (intra-aortic balloon pump)

## Smart ICU

**Mission**

- Support staff
- Improve patient safety
- Build AI-ready capability

> Smart = safe, monitored, AI-driven, reliable technology.

The data pipeline runs end to end:

```mermaid
flowchart LR
  A[Gather data] --> B[Integrate many sources<br/>lab, EHR, monitors] --> C[AI context] --> D[Analysis] --> E[Patient care]
```

## Bottlenecks

| Bottleneck | Detail |
| --- | --- |
| Missing data ports | Some medical equipment lacks an external port to send data out. From now on this must be specified in the TOR before equipment is handed over. |
| Manual labeling | Only raw data is collected, so what happened to the patient during each time frame still has to be labeled by hand. |
| Immature EHR | If the hospital still lacks electronic doctor notes, orders, and patient history, the system will not perform well. |
| Millisecond streams | Some data streams arrive at millisecond intervals, and how to collect them is still unknown. |
| External validation | No other hospital collect as much data as KKU does, so model develop from KKU can't really validate externally |
