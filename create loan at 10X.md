# Create loan at 10X

```mermaid
sequenceDiagram
    participant Channel
    participant LoanSD as Loan SD
    participant FinXGlue as FinX Glue
    participant Core10X as 10X Core

    Channel->>LoanSD: Loan/Initiate
    LoanSD->>FinXGlue: POST /v1/Loan/Initiate

    rect rgb(245, 245, 245)
        Note over FinXGlue,Core10X: Step 1 - Create loan arrangement
        FinXGlue->>Core10X: POST /v2/arrangements (creditLimit, currency, partyKey,productKey, externalReference, repaymentTerms)
        Core10X-->>FinXGlue: 201 Created (ArrangementKey Returned)
    end

    rect rgb(235, 245, 255)
        Note over FinXGlue,Core10X: Step 2 - Accept arrangement
        FinXGlue->>Core10X: POST /v2/arrangements/{arrangementKey}/accept
        alt Success
            Core10X-->>FinXGlue: 204 No Content
        else Validation failure
            Core10X-->>FinXGlue: 404 Not Found\nreferenceId
        end
    end

    rect rgb(245, 255, 245)
        Note over FinXGlue,Core10X: Step 3 - Get arrangement details
        FinXGlue->>Core10X: GET /v3/arrangements/{arrangementKey}
        Core10X-->>FinXGlue: arrangement details\nproductKey, version, creditLimit, currency,\npartyKey, startDate, externalReference, subscriptionKey
    end

    rect rgb(255, 248, 235)
        Note over FinXGlue,Core10X: Step 4 - Get subscription details
        FinXGlue->>Core10X: GET /v3/subscriptions/{subscriptionKey}
        Core10X-->>FinXGlue: subscription details\nstatus = Active, creationDate, name
    end

    rect rgb(255, 240, 245)
        Note over FinXGlue,Core10X: Step 5 - Disburse loan amount
        FinXGlue->>Core10X: PUT /v2/transactions/{correlationKey}?balanceCheck=true\nBalanced transaction with 2 legs
        Note over FinXGlue,Core10X: Leg 1: Debit loan account\nLeg 2: Credit customer receiving account\nEach leg includes transactionId, code = PMNT.IRCT. DMCT,\nstatus = POSTED, narrative = Loan Disbursement
        Core10X-->>FinXGlue: Transaction posted successfully
    end

    FinXGlue-->>LoanSD: Loan initiated and disbursed
    LoanSD-->>Channel: Loan subscription created successfully
```

## Flow summary

Creating a loan subscription (account) in 10X involves these mandatory steps:

1. **Create arrangement** using loan amount, currency, customer Party Key, product key, external reference, and repayment terms.
2. **Accept arrangement** using the returned Arrangement Key.
3. **Get arrangement details** to confirm creation and retrieve the Subscription Key.
4. **Get subscription details** to verify the subscription is Active.
5. **Disburse the loan amount** using a balanced transaction with debit and credit legs submitted together.
