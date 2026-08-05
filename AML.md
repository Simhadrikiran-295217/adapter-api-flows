```mermaid
sequenceDiagram
    participant channel as Channel
    participant plm as Party Lifecycle Management
    participant finx as FinX Glue Adapter
    participant ca as Comply Advantage

    channel->>plm: POST /PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/Initiate
    Note right of channel: Before party creation, customer must pass AML/PEP screening

    plm->>finx: Initiate Qualification assessment POST (/v1/PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/Initiate)
    Note right of finx: QualificationTaskRecord.Task contains full customer profile serialized as JSON string

    finx->>ca: POST /v2/workflows/sync/create-and-screen\n(payload from QualificationTaskRecord.Task)
    ca-->>finx: Workflow response\n(workflow_instance_identifier, status, risk/scoring/step results)

    finx-->>plm: Qualification Initiate response (persisted assessment record)
    plm-->>channel: Qualification initiated + screening outcome snapshot

    channel->>plm: GET /PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/{qualificationid}/Retrieve
    plm->>finx: Retrieve qualification assessment by qualificationid (GET /v1/PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/{qualificationid}/Retrieve)

    finx->>ca: GET /v2/workflows/{workflow_instance_identifier}
    ca-->>finx: Latest workflow status/details

    finx-->>plm: Qualification assessment record
    plm-->>channel: GET Retrieve response
```

Explanation:

Initiate AML/PEP qualification from channel

API: POST /PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/Initiate

Trigger: Before party creation, channel triggers qualification to ensure customer passes AML/PEP screening. The request is sent with the party lifecycle context.

Transfer request from PLM BIAN service to FinX Glue Adapter

API: POST /v1/PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/Initiate

PLM service receives the qualification initiation call and forwards it to FinX Glue Adapter. QualificationTaskRecord.Task contains the full customer profile serialized as a JSON string.

Integration to Comply Advantage screening workflow

API: POST /v2/workflows/sync/create-and-screen

Adapter maps the qualification payload to Comply Advantage format and calls create-and-screen for synchronous AML/PEP evaluation.

Comply Advantage returns workflow outcome

API Response: workflow_instance_identifier, status, and risk/scoring/step results

Adapter persists the assessment snapshot and returns the qualification initiate response back to PLM, which then returns qualification initiated outcome to channel.

Retrieve qualification status/details from channel

API: GET /PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/{qualificationid}/Retrieve

Trigger: Channel calls retrieve to fetch latest qualification status for the same party lifecycle and qualification id.

Transfer retrieve request from PLM BIAN service to FinX Glue Adapter

API: GET /v1/PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/{qualificationid}/Retrieve

PLM forwards the retrieve request to adapter service to obtain latest screening details.

Integration to Comply Advantage workflow retrieval

API: GET /v2/workflows/{workflow_instance_identifier}

Adapter resolves the stored workflow_instance_identifier for the qualification and fetches latest workflow state/details from Comply Advantage.

Output:

200 OK with qualification assessment record and latest AML/PEP screening status pushed back to channel via PLM and FinX Glue Adapter.