# Retrieve Loan Account - BIAN ~ 10X Adapter

```mermaid
sequenceDiagram
    participant C as Channel
    participant L as Loan SD
    participant G as FinX Glue
    participant X as 10X Core

    C->>L: GET /Loan/{LoanId}/Retrieve
    L->>G: GET /v1/Loan/{LoanId}/Retrieve

    G->>X: POST /v3/arrangements/search?limit=1 (externalReference: LoanId)
    X-->>G: arrangement[0] (repaymentScheduleKey, subscriptionKey, creditLimit, currency, externalReference)

    G->>X: GET /v3/subscriptions/{subscriptionKey}
    X-->>G: subscription details (status, productKey, productVersion, partyKey, createdDate, subscriptionName)

    G->>X: GET /v2/products/{productKey}/versions/{productVersion}/summary
    X-->>G: product details (interestType inputs, productGroup, productSegment) Derive LoanType: LOANS + Personal => Consumer Loan,LOANS + Corporate/Business => Corporate Loan

    G->>X: POST /v3/subscriptions/{subscriptionKey}/settlement-quote
    X-->>G: Outstanding Balance

    G->>X: GET /v1/schedules/{repaymentScheduleKey}/versions/latest
    X-->>G: repayment schedule (term, termType, interestRate, repaymentStartDate, Maturity Date),Repayment Start Date = dueDate where period=1, Maturity Date = last dueDate (based on number of payments)

    G-->>L: Fetched Loan Details (BIAN mapping)
    L-->>C: 200 OK Loan Retrieve response
```
