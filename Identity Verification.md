# Identity Verification

## End-to-end Identity Proofing Flow (BIAN + Jumio + FinX Glue)

```mermaid
sequenceDiagram
    autonumber
    participant CH as Channel
    participant BIAN as BIAN API
    participant ADP as Jumio Adapter
    participant JAM as Jumio Account API
    participant JFR as Jumio Front Image API
    participant JBK as Jumio Back Image API
    participant JFW as Jumio Finalize Workflow API
    participant JRS as Jumio Fetch Result API
    participant S3 as FinX S3 Upload API
    participant DD as Document Directory
    participant PLM as Party Lifecycle Management Store

    CH->>BIAN: POST /PartyLifecycleManagement/{id}/IdentityProofing/Initiate\n(consent, customerInternalRef, location, ip, region, docDirectoryIds)
    BIAN->>ADP: Initiate Identity Proofing request

    Note over ADP: Uses predefined workflowDefinitionKey = 2

    ADP->>JAM: POST https://account.emea-1.jumio.ai/api/v1/accounts\n(consent + reference + location + ip + region + workflowDefinitionKey=2)
    JAM-->>ADP: accountId, workflowExecutionId, accessToken/tokenId, FRONT/BACK upload URLs

    ADP->>JFR: POST .../accounts/{accountId}/workflow-executions/{workflowExecutionId}/credentials/{tokenId}/parts/FRONT\n(multipart JPG/PNG)
    JFR-->>ADP: FRONT upload accepted

    ADP->>JBK: POST .../accounts/{accountId}/workflow-executions/{workflowExecutionId}/credentials/{tokenId}/parts/BACK\n(multipart JPG/PNG)
    JBK-->>ADP: BACK upload accepted

    ADP->>JFW: POST .../accounts/{accountId}/workflow-executions/{workflowExecutionId}\n(finalize workflow)
    JFW-->>ADP: Workflow finalized

    Note over ADP,JRS: Finalize must occur after BOTH uploads\nElse Fetch Result may return "precondition not fulfilled"

    ADP->>JRS: GET https://retrieval.emea-1.jumio.ai/api/v1/accounts/{accountId}/workflow-executions/{workflowExecutionId}
    JRS-->>ADP: Full assessment result payload (original Jumio result)

    ADP->>S3: POST https://gatewayqa.ustfinx.com/v1/document-directory/s3/upload\n(upload full fetch-result payload as PDF)
    S3-->>ADP: S3 URL for Identity Proofing Log document

    ADP->>DD: POST https://gatewayqa.ustfinx.com/v1/document-directory/register\n(register S3 URL with tags: Identity Proofing Log, PartyId, AssessmentId)
    DD-->>ADP: documentDirectoryId

    ADP->>PLM: Persist assessment result + final decision\nPersist original Jumio result metadata
    ADP->>PLM: Update Identity Proofing BQ documentInstanceReference\nwith documentDirectoryId

    ADP-->>BIAN: Initiate response with assessment + final result + document reference
    BIAN-->>CH: IdentityProofing Initiate response
```

## Notes

- The adapter determines the **final decision** based on the Jumio **original result**.
- Both **assessment result** and **final result** are stored in the **Party Lifecycle Management** store.
- The **original Jumio result** must be converted/stored as a **PDF log document**, registered in Document Directory, and linked back to the Identity Proofing BQ to appear in CAM Documents tab.
