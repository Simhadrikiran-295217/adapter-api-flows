```mermaid
sequenceDiagram
    participant channel as Channel
    participant plm as Party Lifecycle Management
    participant finx as FinX Glue Adapter
    participant ca as Comply Advantage


    channel->>plm: POST /PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/Initiate
    Note right of channel: Before party creation, customer must pass AML/PEP screening

    plm->>finx: Initiate Qualification assessment(POST /v1/PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/Initiate)
    Note right of finx: QualificationTaskRecord.Task contains full customer profile serialized as JSON string

    finx->>ca: POST /v2/workflows/sync/create-and-screen\n(payload from QualificationTaskRecord.Task)
    ca-->>finx: Workflow response\n(workflow_instance_identifier, status, risk/scoring/step results)


    finx-->>plm: Qualification Initiate response (persisted assessment record)
    plm-->>channel: Qualification initiated + screening outcome snapshot

    channel->>plm: GET /PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/{qualificationid}/Retrieve
    plm->>finx: Retrieve qualification assessment by qualificationid (GET /v1/PartyLifecycleManagement/{partylifecyclemanagementId}/Qualification/{qualificationid}/Retrieve)
    Note right of finx: qualificationid is independently addressable\nSupports multiple instances under same lifecycle CR\n(e.g., initial + periodic re-screening)
        finx->>ca: GET /v2/workflows/{workflow_instance_identifier}
        ca-->>finx: Latest workflow status/details
        finx->>finx: Refresh persisted instruction/result fields
    end

    finx-->>plm: Qualification assessment record
    plm-->>channel: GET Retrieve response
```
