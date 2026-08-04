```mermaid
sequenceDiagram

    participant Customer as Customer

    participant Channel as Customer Channel

    participant PD as Product Directory SD

    participant Adapter as FinX TM Adapter

    participant Core as TM Core


    Customer->>Channel: Access Product Onboarding Journey


    Channel->>PD: GET /product-directory/products/retrieve


    PD->>Adapter: GET /product-directory/products/retrieve


    Adapter->>Core: GET /v1/products?is_internal=false

    Core-->>Adapter: 200 OK Product Groups


    Adapter-->>PD: 200 OK ProductListing[]


    PD-->>Channel: 200 OK ProductListing[]


    Channel->>PD: GET /product-directory/{product-directory-id}/retrieve


    PD->>Adapter: GET /product-directory/{product-directory-id}/retrieve


    Adapter->>Core: GET /v1/product-versions<br/>?product_id={product-directory-id}<br/>&view=PRODUCT_VERSION_VIEW_INCLUDE_TAGS


    Core-->>Adapter: 200 OK Product Versions


    Adapter-->>PD: 200 OK ProductDirectoryEntry<br/>ProductReference[]


    PD-->>Channel: 200 OK ProductDirectoryEntry<br/>ProductReference[]


    Note right of Channel: Filter ProductReference[] where<br/>ProductPriority == LEVEL_VARIANT


    Channel-->>Customer: Render Available Products<br/>Savings Standard<br/>Savings Premium<br/>Youth Savings


    Customer->>Channel: Select Product


    Channel->>PD: GET /product-directory/{product-directory-id}/sales-and-marketing/{sales-and-marketing-id}/retrieve


    PD->>Adapter: GET /product-directory/{product-directory-id}/sales-and-marketing/{sales-and-marketing-id}/retrieve


    Adapter->>Core: GET /v1/product-versions:batchGet<br/>?version_ids={product_version_id}<br/>&view=PRODUCT_VERSION_VIEW_INCLUDE_PARAMETERS


    Core-->>Adapter: 200 OK Product Parameters


    Adapter-->>PD: 200 OK SalesandMarketing


    PD-->>Channel: 200 OK SalesandMarketing


    Channel-->>Customer: Render Product Features<br/>Benefits<br/>Rates<br/>Limits<br/>Eligibility


    Customer->>Channel: Click Apply


    Note right of Channel: Continue to Account Opening Journey
```
