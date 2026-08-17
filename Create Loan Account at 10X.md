```mermaid
sequenceDiagram
    autonumber
    participant Channel
    participant LoanSD as Loan SD
    participant FinX as FinX Glue
    participant Core as 10X Core

    %% =========================
    %% FLOW 1: INITIATE LOAN
    %% =========================
    rect rgb(235, 245, 255)
    Note over Channel,Core: Flow 1 — Initiate Loan Account

    Channel->>LoanSD: POST /Loan/Initiate (BIAN)\nLoanAmount, Currency, PartyReference,\nProductIdentifier, Repayment Terms,\nAccount Number (human-readable)
    LoanSD->>FinX: Validate & transform BIAN -> 10X adapter payload

    FinX->>Core: POST /v2/arrangements\n{creditLimit, currency, partyKey, productKey,\nexternalReference, termLength, termType,\nfrequency, startDate, state=OFFERED}
    Core-->>FinX: 201 Created + arrangementKey
    FinX-->>LoanSD: arrangementKey

    Note over FinX,Core: 10X only permits OFFERED at creation; accept is mandatory before disbursement

    FinX->>Core: POST /v2/arrangements/{arrangementKey}/accept
    alt Accept success
        Core-->>FinX: 204 No Content
    else Validation failure
        Core-->>FinX: 404 + referenceId
    end

    FinX->>Core: GET /v3/arrangements/{arrangementKey}
    Core-->>FinX: arrangement details\nproductKey, productVersion, creditLimit,\ncurrency, partyKey, startDate, externalReference,\nsubscriptionKey

    FinX->>Core: GET /v3/subscriptions/{subscriptionKey}
    Core-->>FinX: subscriptionStatus=Active,\ncreatedDate, subscriptionName

    FinX->>Core: PUT /v2/transactions/{correlationKey}?balanceCheck=true\n(single balanced transaction)\nLeg1: DEBIT loan account\ntransactionId=<human-readable>,\ntransactionCode=PMNT.IRCT.DMCT, status=POSTED,\nnarrative=Loan Disbursement\nLeg2: CREDIT customer receiving account\ntransactionId=<human-readable>,\ntransactionCode=PMNT.IRCT.DMCT, status=POSTED,\nnarrative=Loan Disbursement
    Core-->>FinX: Disbursement posted
    FinX-->>LoanSD: Initiate response (loan created + disbursed)
    LoanSD-->>Channel: 200/201 Success response
    end

    %% =========================
    %% FLOW 2: RETRIEVE LOAN
    %% =========================
    rect rgb(245, 255, 235)
    Note over Channel,Core: Flow 2 — Retrieve Loan Account

    Channel->>LoanSD: GET /Loan/{LoanId}/Retrieve (BIAN)
    LoanSD->>FinX: Retrieve request with LoanId

    FinX->>Core: Search arrangement by externalReference = LoanId
    Core-->>FinX: arrangement found\n(creditLimit, currency, startDate, externalReference,\nsubscriptionKey, repaymentScheduleKey)

    FinX->>Core: GET /v3/subscriptions/{subscriptionKey}
    Core-->>FinX: status, productKey, productVersion,\npartyKey, createdDate, subscriptionName

    FinX->>Core: Get outstanding balance by subscriptionKey
    Core-->>FinX: outstanding balance

    FinX->>Core: Get repayment schedule by repaymentScheduleKey
    Core-->>FinX: term length, term type, interest rate

    FinX-->>LoanSD: Aggregated retrieve response
    LoanSD-->>Channel: Loan details response
    end

    %% =========================
    %% ASSUMPTIONS / NOTES
    %% =========================
    Note over Channel,Core: Assumptions: customer receiving account already exists.
    Note over Channel,Core: externalReference carries human-readable LoanId/Account Number.
    Note over Channel,Core: Quote generation omitted in demo scope.
```