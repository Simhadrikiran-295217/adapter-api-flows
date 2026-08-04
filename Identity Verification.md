# Identity Verification

## End-to-end Identity Proofing Flow (BIAN + Jumio + FinX Glue)

```mermaid
sequenceDiagram
    autonumber
    participant CH as Customer Channel
    participant BIAN as Party Lifecycle Management SD
    participant FG as FinX TM Adapter
    participant ADP as TM Core (Jumio)

    CH->>BIAN: POST /PartyLifecycleManagement/{id}/IdentityProofing/Initiate\n(consent, customerInternalRef, location, ip, region, docDirectoryIds)
    BIAN->>FG: Initiate Identity Proofing request
    FG->>ADP: Jumio Account API POST /v1/accounts\n(consent, customerInternalRef, location, ip, region, workflowDefinitionKey=2)

    Note over ADP: Uses predefined workflowDefinitionKey = 2

    ADP-->>FG: accountId, workflowExecutionId, accessToken/tokenId, FRONT/BACK upload URLs
    FG-->>BIAN: account + workflow + upload details
    BIAN-->>CH: Identity proofing session initialized

    CH->>BIAN: Upload FRONT and BACK document images
    BIAN->>FG: Forward document images
    FG->>ADP: Front Image API POST /v1/accounts/{{accountId}}/\nworkflow-executions/{{workflowExecutionId}}/credentials/{{tokenId}}/parts/FRONT
    ADP-->>FG: FRONT upload accepted

    FG->>ADP: Back Image API POST /v1/accounts/{{accountId}}/\nworkflow-executions/{{workflowExecutionId}}/credentials/{{tokenId}}/parts/BACK
    ADP-->>FG: BACK upload accepted

    FG-->>BIAN: Document upload status
    BIAN-->>CH: Upload accepted

    FG->>ADP: Finalize Workflow API POST /v1/accounts/{{accountId}}/\nworkflow-executions/{{workflowExecutionId}}
    ADP-->>FG: Workflow finalized

    Note over FG,ADP: Finalize must occur after BOTH uploads\nElse Fetch Result may return "precondition not fulfilled"

    FG->>ADP: Fetch Result API GET /v1/accounts/{{accountId}}/\nworkflow-executions/{{workflowExecutionId}}
    ADP-->>FG: Full assessment result payload (original Jumio result)

    FG->>FG: Upload Jumio result as PDF to S3 POST /v1/document-directory/s3/upload
    FG->>FG: Register document in Document Directory POST /v1/document-directory/register\n(returns Document Directory ID)
    FG->>FG: Update documentInstanceReference in Identity Proofing BQ POST /v1/document-directory/register\n(links log document back to BQ)

    FG-->>BIAN: Initiate response with assessment + final result + document reference
    BIAN-->>CH: IdentityProofing Initiate response
```

## Notes

- The adapter determines the **final decision** based on the Jumio **original result**.
- Both **assessment result** and **final result** are stored in the **Party Lifecycle Management** store.
- The **original Jumio result** must be converted/stored as a **PDF log document**, registered in Document Directory, and linked back to the Identity Proofing BQ to appear in CAM Documents tab.
