# 🏛️ PM04-PS05 — Municipal Complaint Automation Bot

> **UiPath RPA Process | Windows Framework**  
> Automates the ingestion, normalization, classification, and reporting of municipal citizen complaints from CSV and PDF sources.

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Features](#-features)
- [Project Structure](#-project-structure)
- [Workflow Architecture](#-workflow-architecture)
- [Prerequisites](#-prerequisites)
- [Input Files](#-input-files)
- [Output Files](#-output-files)
- [Classification Rules](#-classification-rules)
- [Error Handling](#-error-handling)
- [How to Run](#-how-to-run)
- [Known Issues & Limitations](#-known-issues--limitations)

---

## 📌 Overview

The **Municipal Complaint Automation Bot** is a UiPath RPA process that processes citizen complaints submitted to a municipality. It reads complaints from two sources — a **CSV file** and a folder of **PDF documents** — cleans and normalises the data, classifies each complaint by category using keyword matching, writes the results to Excel output files, and sends a daily summary email to a supervisor.

| Field | Detail |
|---|---|
| **Project Name** | PM04-PS05 |
| **Project Type** | Process |
| **Framework** | Windows |
| **Entry Point** | `Main.xaml` |
| **Last Updated** | 2026/04/21 |

---

## ✨ Features

- 📄 **Dual-source ingestion** — reads complaints from both CSV and PDF files
- 🔍 **Document Understanding pipeline** — uses UiPath DU to digitize and extract fields from PDF complaints
- 🧹 **Data normalization** — trims whitespace, validates fields, and standardizes date formats to `yyyy-MM-dd`
- 🏷️ **Keyword-based classification** — categorizes each complaint as Water, Electricity, Refuse, or Other
- 💾 **Excel output** — writes classified complaints and an error log to separate Excel files
- 📧 **Summary email** — sends a daily complaint summary report to a supervisor via Gmail
- 🛡️ **Error logging** — captures normalization and validation failures with timestamps

---

## 📁 Project Structure

```
PM04-PS05/
│
├── Main.xaml                        ← Entry point / orchestrator
│
├── Workflows/
│   ├── ReadComplaints.xaml          ← Reads complaints from CSV
│   ├── ExtractFromPDF.xaml          ← Document Understanding PDF extraction
│   ├── NormalizeData.xaml           ← Cleans and validates complaint rows
│   ├── ClassifyComplaint.xaml       ← Keyword-based complaint classification
│   ├── WriteOutput.xaml             ← Writes results to Excel
│   └── SendSummaryEmail.xaml        ← Sends summary email via Gmail
│
├── DocumentProcessing/
│   └── taxonomy.json                ← DU taxonomy for PDF classification
│
├── Data/
│   ├── Input/
│   │   ├── MUNICIPAL_Complaints.csv ← Source CSV complaints file
│   │   ├── KeywordClassifierLearning.json
│   │   └── SamplePDFs/
│   │       ├── Complaint1.pdf
│   │       ├── Complaint2.pdf
│   │       └── ...
│   └── Output/
│       ├── Out_Classified_Complaints.xlsx  ← Main output
│       └── Error_Log.xlsx                  ← Error records (if any)
│
├── README.md                        ← This file
├── Build_Guide_PM04-PS05.txt        ← Step-by-step build documentation
└── Workflow_Map_PM04-PS05.md        ← Visual workflow diagram (Mermaid)
```

---

## 🔧 Workflow Architecture

The bot follows a **linear orchestration pattern** where `Main.xaml` invokes each sub-workflow in sequence, passing data via arguments.

```
START
  │
  ├─► [ReadComplaints.xaml]     CSV → dt_CSVComplaints
  │       └─► Merge into dt_MasterComplaintsRaw
  │
  ├─► [ExtractFromPDF.xaml]     PDF × N → dt_PDFExtracted
  │       └─► Merge into dt_MasterComplaintsRaw
  │
  ├─► [NormalizeData.xaml]      dt_MasterComplaintsRaw → dt_NormalizedComplaints + dt_Errors
  │
  ├─► [ClassifyComplaint.xaml]  dt_ClassifiedComplaints (adds Bot_Classification column)
  │
  ├─► [WriteOutput.xaml]        → Out_Classified_Complaints.xlsx + Error_Log.xlsx
  │
  └─► [SendSummaryEmail.xaml]   → Gmail summary to supervisor
```

### Sub-Workflow Arguments

| Workflow | Input Arguments | Output Arguments |
|---|---|---|
| `ReadComplaints.xaml` | *(none)* | `out_ComplaintsTable` (DataTable) |
| `ExtractFromPDF.xaml` | `in_PDFFilePath` (String) | `out_ExtractedData` (DataTable) |
| `NormalizeData.xaml` | `in_ComplaintsTable` (DataTable) | `out_NormalizedComplaints` (DataTable), `out_ErrorLog` (DataTable) |
| `ClassifyComplaint.xaml` | `io_ClassifiedComplaints` (In/Out DataTable), `in_ErrorLog` (DataTable) | *(modified in place)* |
| `WriteOutput.xaml` | `in_ClassifiedTable` (DataTable), `in_ErrorLog` (DataTable) | *(none)* |
| `SendSummaryEmail.xaml` | `in_SupervisorEmail` (String), `in_ErrorCount` (Int32), `in_ClassifiedTable` (DataTable) | *(none)* |

---

## ⚙️ Prerequisites

### UiPath Packages Required

| Package | Purpose |
|---|---|
| `UiPath.System.Activities` | Core activities (Assign, If, For Each, etc.) |
| `UiPath.Excel.Activities` | Read CSV, Write Range |
| `UiPath.DocumentUnderstanding.Activities` | Digitize, Classify, Extract |
| `UiPath.GSuite.Activities` | Send Gmail summary email |
| `UiPath.UIAutomation.Activities` | General UI support |

### Connections Required

| Connection | Used In | Purpose |
|---|---|---|
| **Gmail / GSuite** | `SendSummaryEmail.xaml` | Sending the daily summary email |

### Other Requirements

- A valid **DU taxonomy file** at `DocumentProcessing/taxonomy.json`
- PDF complaint files placed in `Data/Input/SamplePDFs/`
- Supervisor email address set in the `strSupervisorEmail` variable in `Main.xaml`

---

## 📥 Input Files

### 1. `Data/Input/MUNICIPAL_Complaints.csv`
The primary CSV source file containing citizen complaints.

**Expected columns:**

| Column | Type | Description |
|---|---|---|
| `Complaint_ID` | String | Unique complaint identifier |
| `Citizen_Name` | String | Name of the citizen |
| `Complaint_Text` | String | Full text description of the complaint |
| `Date_Received` | String | Date the complaint was received |

**Accepted date formats:** `yyyy-MM-dd` or `yyyy/MM/dd HH:mm:ss`

### 2. `Data/Input/SamplePDFs/*.pdf`
PDF complaint documents to be processed through the Document Understanding pipeline. All `.pdf` files in this folder are processed automatically.

---

## 📤 Output Files

| File | Location | Description |
|---|---|---|
| `Out_Classified_Complaints.xlsx` | `Data/Output/` | All complaints with their `Bot_Classification` label |
| `Error_Log.xlsx` | `Data/Output/` | Rows that failed validation (empty text, missing date, etc.) — only created if errors exist |

---

## 🏷️ Classification Rules

Complaints are classified by searching the complaint text (case-insensitive) for the following keywords:

| Category | Keywords |
|---|---|
| 💧 **Water** | `burst pipe`, `no water supply`, `low water pressure` |
| ⚡ **Electricity** | `load shedding`, `transformer exploded`, `no power` |
| 🗑️ **Refuse** | `bin not collected`, `rubbish piling up`, `refuse truck missed` |
| ❓ **Other** | *(default — any complaint not matching the above)* |

> The classification is stored in the `Bot_Classification` column of the output Excel file.

---

## 🛡️ Error Handling

The bot uses a multi-layered error handling approach:

| Layer | Mechanism | Behaviour |
|---|---|---|
| **Missing CSV** | `If` condition in Main | Logs warning, skips CSV ingestion gracefully |
| **No PDFs found** | `If` condition in Main | Logs warning, skips PDF ingestion gracefully |
| **Empty master table** | `If` condition in Main | Logs warning and halts processing |
| **Empty complaint text** | `If` in NormalizeData | Logs row to Error Log, continues processing |
| **Missing/invalid date** | `If` in NormalizeData | Logs row to Error Log, continues processing |
| **Row-level exceptions** | `Try/Catch` in NormalizeData | Captures exception message, logs to Error Log |
| **Empty classified table** | `If` in SendSummaryEmail | Skips email sending gracefully |

All errors are logged to `dt_Errors` and written to `Data/Output/Error_Log.xlsx` at the end of the run.

---

## ▶️ How to Run

1. **Open** the project in UiPath Studio Desktop
2. **Verify** that `Data/Input/MUNICIPAL_Complaints.csv` exists and is correctly formatted
3. **Place** any PDF complaint files into `Data/Input/SamplePDFs/`
4. **Set** the supervisor email in `Main.xaml` → variable `strSupervisorEmail`
5. **Ensure** the Gmail connection is configured and authenticated
6. **Press `F5`** (or click **Run**) to execute the bot from `Main.xaml`
7. **Check outputs** in `Data/Output/` after the run completes

---

## ⚠️ Known Issues & Limitations

| Issue | Detail |
|---|---|
| `ClassifyDocumentScope` has no classifiers | A classifier (e.g. Keyword Classifier) must be manually added inside `ExtractFromPDF.xaml` |
| `DataExtractionScope` missing arguments | `DocumentPath`, `Taxonomy`, and `ClassificationResult` must be wired up in `ExtractFromPDF.xaml` |
| `classificationResult` type mismatch | Variable must be changed to array type `ClassificationResult()` |
| `dsExtracted` type mismatch | Variable typed as `Dataset` (DU type) — may conflict with `System.Data.DataSet` |
| Keyword classification is rule-based | No ML model used — classification accuracy depends entirely on keyword presence in complaint text |

---

*Documentation generated: 2026/04/21 | PM04-PS05 — Municipal Complaint Automation Bot*
