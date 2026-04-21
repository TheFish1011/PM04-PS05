# 🗺️ Workflow Map — PM04-PS05 Municipal Complaint Automation Bot

> **Tip:** To render this diagram, open this file in **VS Code** with the
> [Markdown Preview Mermaid Support](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid)
> extension, or paste the code block into [https://mermaid.live](https://mermaid.live)

---

## 🔁 Full Process Flow

```mermaid
flowchart TD
    START([🚀 Bot Started\nMain.xaml]) --> BUILD[Build Master\nComplaints Table\ndt_MasterComplaintsRaw]

    BUILD --> CSV_SEQ

    subgraph CSV_SEQ ["📄 CSV Extraction"]
        CSV_CHECK{CSV File\nExists?}
        CSV_READ[Invoke ReadComplaints.xaml\nRead CSV → dt_CSVComplaints]
        CSV_LOOP[For Each Row\nAdd to dt_MasterComplaintsRaw]
        CSV_SKIP[Log: CSV Not Found\nSkip]

        CSV_CHECK -- Yes --> CSV_READ --> CSV_LOOP
        CSV_CHECK -- No  --> CSV_SKIP
    end

    CSV_SEQ --> PDF_SEQ

    subgraph PDF_SEQ ["📑 PDF Extraction"]
        PDF_CHECK{PDF Files\nFound?}
        PDF_LOOP[For Each PDF File]
        PDF_INVOKE[Invoke ExtractFromPDF.xaml\nDU Pipeline → dt_PDFExtracted]
        PDF_ADD[For Each Row\nAdd to dt_MasterComplaintsRaw]
        PDF_SKIP[Log: No PDFs Found\nSkip]

        PDF_CHECK -- Yes --> PDF_LOOP --> PDF_INVOKE --> PDF_ADD
        PDF_CHECK -- No  --> PDF_SKIP
    end

    PDF_SEQ --> GUARD

    GUARD{Master Table\nHas Rows?}
    GUARD -- No --> WARN[Log Warning:\nNo Data Found\n🛑 Stop]
    GUARD -- Yes --> NORM

    subgraph NORM ["🧹 NormalizeData.xaml"]
        N1[Build Error Log Table]
        N2[Clone Master Table\nStructure → dtNormalized]
        N3[For Each Row in\ndt_MasterComplaintsRaw]
        N4[Extract & Trim Fields\nComplaint_ID, Name,\nText, Date]
        N5{Complaint\nText Empty?}
        N6[Log Error Row\nto out_ErrorLog]
        N7{Date\nPresent?}
        N8[Parse & Reformat Date\nyyyy-MM-dd]
        N9[Log Missing\nDate Error]
        N10[Add Clean Row\nto dtNormalized]
        N_CATCH[Catch Exception\nLog to Error Log]

        N1 --> N2 --> N3 --> N4
        N4 --> N5
        N5 -- Yes --> N6 --> N7
        N5 -- No  --> N7
        N7 -- Yes --> N8 --> N10
        N7 -- No  --> N9 --> N10
        N10 --> N3
        N3 -- Done --> N_OUT[Export\nout_NormalizedComplaints]
        N4 -. Exception .-> N_CATCH
    end

    NORM --> COPY[Assign:\ndt_ClassifiedComplaints =\ndt_NormalizedComplaints.Copy]

    COPY --> CLASSIFY

    subgraph CLASSIFY ["🏷️ ClassifyComplaint.xaml"]
        C1[Add Column:\nBot_Classification]
        C2[For Each Row]
        C3[Get Complaint Text\nToLower + Trim]
        C4[Default:\nstrClassification = Other]
        C5{Check\nKeywords}
        C6[Water\nburst pipe\nno water supply\nlow water pressure]
        C7[Electricity\nload shedding\ntransformer exploded\nno power]
        C8[Refuse\nbin not collected\nrubbish piling up\nrefuse truck missed]
        C9[Other\nno keyword match]
        C10[Set CurrentRow\nBot_Classification]

        C1 --> C2 --> C3 --> C4 --> C5
        C5 -- Water Keywords    --> C6 --> C10
        C5 -- Electricity Keywords --> C7 --> C10
        C5 -- Refuse Keywords   --> C8 --> C10
        C5 -- No Match          --> C9 --> C10
        C10 --> C2
    end

    CLASSIFY --> WRITE

    subgraph WRITE ["💾 WriteOutput.xaml"]
        W1[Write Range:\ndt_ClassifiedComplaints\n→ Out_Classified_Complaints.xlsx]
        W2{Error Log\nHas Rows?}
        W3[Write Range:\ndt_Errors\n→ Error_Log.xlsx]
        W4[Log: Output\nWritten Successfully]

        W1 --> W2
        W2 -- Yes --> W3 --> W4
        W2 -- No  --> W4
    end

    WRITE --> EMAIL

    subgraph EMAIL ["📧 SendSummaryEmail.xaml"]
        E1{Classified Table\nEmpty?}
        E2[Log Warning:\nSkip Email]
        E3{Column\nBot_Classification\nExists?}
        E4[Log Warning:\nColumn Missing]
        E5[Count Water /\nElectricity /\nRefuse / Other]
        E6[Build Email Body\nwith Summary Stats]
        E7[Send Gmail\nto Supervisor]

        E1 -- Yes --> E2
        E1 -- No  --> E3
        E3 -- No  --> E4
        E3 -- Yes --> E5 --> E6 --> E7
    end

    EMAIL --> END([✅ Bot Completed\nLog Timestamp])
```

---

## 📦 Sub-Workflow Inputs & Outputs

```mermaid
flowchart LR
    subgraph MAIN ["🏠 Main.xaml"]
        M1([Start])
    end

    subgraph RC ["ReadComplaints.xaml"]
        RC_IN[ ]
        RC_OUT[out_ComplaintsTable\nDataTable]
    end

    subgraph EP ["ExtractFromPDF.xaml"]
        EP_IN[in_PDFFilePath\nString]
        EP_OUT[out_ExtractedData\nDataTable]
    end

    subgraph ND ["NormalizeData.xaml"]
        ND_IN[in_ComplaintsTable\nDataTable]
        ND_OUT1[out_NormalizedComplaints\nDataTable]
        ND_OUT2[out_ErrorLog\nDataTable]
    end

    subgraph CC ["ClassifyComplaint.xaml"]
        CC_INOUT[io_ClassifiedComplaints\nDataTable In/Out]
        CC_IN2[in_ErrorLog\nDataTable]
    end

    subgraph WO ["WriteOutput.xaml"]
        WO_IN1[in_ClassifiedTable\nDataTable]
        WO_IN2[in_ErrorLog\nDataTable]
    end

    subgraph SE ["SendSummaryEmail.xaml"]
        SE_IN1[in_SupervisorEmail\nString]
        SE_IN2[in_ErrorCount\nInt32]
        SE_IN3[in_ClassifiedTable\nDataTable]
    end

    MAIN -->|invokes| RC
    MAIN -->|invokes per PDF| EP
    MAIN -->|invokes| ND
    MAIN -->|invokes| CC
    MAIN -->|invokes| WO
    MAIN -->|invokes| SE
```

---

## 🗂️ Data Tables Reference

| Variable | Type | Created In | Used In |
|---|---|---|---|
| `dt_MasterComplaintsRaw` | DataTable | Main.xaml | NormalizeData |
| `dt_CSVComplaints` | DataTable | ReadComplaints | Main.xaml |
| `dt_PDFExtracted` | DataTable | ExtractFromPDF | Main.xaml |
| `dt_NormalizedComplaints` | DataTable | NormalizeData | Main.xaml |
| `dt_ClassifiedComplaints` | DataTable | Main.xaml (copy) | ClassifyComplaint, WriteOutput, SendSummaryEmail |
| `dt_Errors` | DataTable | NormalizeData | WriteOutput, SendSummaryEmail |

---

## 🏷️ Classification Keywords Reference

| Category | Keywords |
|---|---|
| 💧 Water | `burst pipe`, `no water supply`, `low water pressure` |
| ⚡ Electricity | `load shedding`, `transformer exploded`, `no power` |
| 🗑️ Refuse | `bin not collected`, `rubbish piling up`, `refuse truck missed` |
| ❓ Other | *(anything that does not match the above)* |

---

*Generated: 2026/04/21 | Project: PM04-PS05 — Municipal Complaint Automation Bot*
