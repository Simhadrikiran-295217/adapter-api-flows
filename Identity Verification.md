# Identity Verification

## End-to-end Identity Proofing Flow (BIAN + Jumio + FinX Glue)

```mermaid
sequenceDiagram
    autonumber
    participant CH as Customer Channel
    participant BIAN as Party Lifecycle Management SD
    participant FG as FinX TM Adapter
    participant ADP as TM Core

    CH->>BIAN: POST /PartyLifecycleManagement/{id}/IdentityProofing/Initiate\n(consent, customerInternalRef, location, ip, region, docDirectoryIds)
    BIAN->>FG: Initiate Identity Proofing request
    FG->>ADP: Forward request to Jumio Adapter

    Note over ADP: Uses predefined workflowDefinitionKey = 2

    ADP-->>FG: accountId, workflowExecutionId, accessToken/tokenId, FRONT/BACK upload URLs
    FG-->>BIAN: account + workflow + upload details
    BIAN-->>CH: Identity proofing session initialized

    CH->>BIAN: Upload FRONT and BACK document images
    BIAN->>FG: Forward document images
    FG->>ADP: Upload FRONT/BACK to Jumio

    ADP-->>FG: FRONT/BACK upload accepted
    FG-->>BIAN: Document upload status
    BIAN-->>CH: Upload accepted

    FG->>ADP: Finalize workflow after both uploads
    ADP-->>FG: Workflow finalized

    Note over FG,ADP: Finalize must occur after BOTH uploads\nElse Fetch Result may return "precondition not fulfilled"

    FG->>ADP: Fetch final assessment result
    ADP-->>FG: Full assessment result payload (original Jumio result)

    FG->>FG: Upload full fetch-result payload as PDF log document
    FG->>FG: Register document in Document Directory\nand persist assessment/final decision in PLM

    ADP-->>FG: Final assessment + metadata confirmed
    FG-->>BIAN: Initiate response with assessment + final result + document reference
    BIAN-->>CH: IdentityProofing Initiate response
```

## Notes

- The adapter determines the **final decision** based on the Jumio **original result**.
- Both **assessment result** and **final result** are stored in the **Party Lifecycle Management** store.
- The **original Jumio result** must be converted/stored as a **PDF log document**, registered in Document Directory, and linked back to the Identity Proofing BQ to appear in CAM Documents tab.
