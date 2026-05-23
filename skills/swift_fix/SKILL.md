---
name: procurement-flow
description: >
  Defines the business logic and document lifecycle for the Swift Fix application.
  Focuses on the "Procurement-based Asset Deployment" workflow including Material Request (MR),
  Request for Quotation (RFQ), Sales Quotation (SQ), Purchase Order (PO), Purchase Receipt (PR),
  Asset, and Asset Capitalization. Use this when implementing features or validations related to
  procurement, serialization, assets, or document flow within the Swift Fix app.
license: MIT
compatibility: "Claude Code, Claude.ai Projects, Claude API. Frappe v14-v16."
metadata:
  author: Swift Fix
  version: "1.1"
---

# Procurement-based Asset Deployment

> Detailed business flow and document lifecycle for the Swift Fix application. Use this document to understand key document definitions, state transitions, validations, and the relationships between procurement documents.

---

## 1. Overview of the Flow

The Swift Fix application operates on a custom business process called **Procurement-based Asset Deployment**. This flow tracks the lifecycle of materials and equipment from initial intent, through negotiations, purchase commitments, receipt of goods, and serialization, to final capitalization/deployment in the field.

```mermaid
flowchart TD
    subgraph Procurement Lifecycle
        MR[Material Request<br><b>Intent of Procuring</b>] --> RFQ[Request for Quotation]
        RFQ --> SQ[Sales Quotation<br><i>Supplier Quotation</i>]
        SQ --> PO[Purchase Order<br><b>Terms of Procuring</b>]
    end

    subgraph Receipt & Serialization
        PO --> PR[Purchase Receipt<br><i>Completion of Mfg</i>]
        PR -->|Simultaneous Creation<br>PR/Item Link| Asset[Asset<br><b>Completion of Procurement</b>]
    end

    subgraph Deployment
        PR & Asset --> AC[Asset Capitalisation<br><b>Deployment of Item</b>]
    end

    style MR fill:#f9f,stroke:#333,stroke-width:2px
    style PO fill:#bbf,stroke:#333,stroke-width:2px
    style Asset fill:#bfb,stroke:#333,stroke-width:2px
    style AC fill:#fbb,stroke:#333,stroke-width:2px
```

---

## 2. Core Procurement Status Landmarks

There are three primary document states that define the status of the entire procurement lifecycle:

| Document | Abbreviation | Core Significance |
| :--- | :--- | :--- |
| **Material Request** | **MR** | Defines the **intent of procuring**. |
| **Purchase Order** | **PO** | Defines the **contractual terms of procuring** with the vendor/supplier. |
| **Asset** | **Asset** | Defines the **completion of procurement**. |

---

## 3. Document Roles and Rules

### 1. Material Request (MR)
* **Definition**: Initiates the procurement lifecycle by stating the intent to acquire a material or service.
* **Hiding Action Buttons**: Standard "Purchase Order" and "Supplier Quotation" action buttons under the `Create` group must be hidden from the Material Request form, as raising POs or SQs directly from the MR is not allowed in this workflow.
* **Custom Processing Status**: Status of MR is tracked via the custom field `custom_processing_status` (e.g. `Draft`, `Shortlisted`, `Cancelled`, `Held`, `Asset Capitalised`).
* **Item Validation**: Only allow items with `is_fixed_asset = 1` to be added to a Material Request. If any item has `is_fixed_asset` different from 1, a validation error must be thrown.

### 2. Request for Quotation (RFQ)
* **Definition**: A document sent to one or more suppliers/vendors to request a quote.
* **Validation**:
  * Links to Material Requests via the `material_request` field in the `items` child table.
  * **Save Restriction**: RFQ cannot be saved/submitted unless there is at least one linked Material Request in the items table, and all linked Material Requests are in **Shortlisted** status.
* **Submit Side-Effect**:
  * Upon submission of an RFQ, a timeline comment must be automatically posted to all unique linked Material Requests:
    > `"A Quotation is requested from Vendor and Recce Process is in Progress"`
* **UI Banner**:
  * Displays linked Material Requests dynamically in the custom HTML field `custom_mr_html` on the RFQ form, supporting multiple MR cards if needed.

### 3. Sales Quotation (SQ)
* **Definition**: The pricing quotation received back from the supplier/vendor (sometimes referred to as the Supplier Quotation).
* **Role**: Details vendor pricing, dimensions, and specifications for evaluation against the RFQ.
* **UI Banner**:
  * Displays linked Material Requests dynamically in the custom HTML field `custom_mr_html`.
  * Resolves Material Requests by checking `material_request` in its own items child table, falling back to checking linked `request_for_quotation` items if direct links are absent.

### 4. Purchase Order (PO)
* **Definition**: Formal contract issued to the supplier specifying terms, pricing, and quantities.
* **Role**: Locks in the terms of the procurement.
* **State Constraint**: If an active (submitted and not Closed or Completed) Purchase Order exists for a Material Request, the MR status cannot be changed to `Cancelled` or `Held`.
* **Submit Hook (`on_po_submit`)**:
  * Upon submission of a PO, if no Purchase Receipts or Assets have been generated for it yet, the linked Material Request's `custom_processing_status` must automatically transition to **Under Process**.
  * A timeline comment is posted on the MR: `"Status updated to Under Process upon submission of Purchase Order [PO_Name]"`.

### 5. Purchase Receipt (PR)
* **Definition**: Signifies the completion of the manufacture/supply of the item based on the specifications.
* **Serialization**:
  * A unique **Serial Number** must be generated for the item upon completion/submission of the Purchase Receipt.
* **Simultaneous Creation**:
  * At the same time the Purchase Receipt is created/submitted, an associated **Asset** document must be created automatically.
  * The new Asset is linked directly to the Purchase Receipt (PR) and Purchase Receipt Item.
* **Submit Hook (`on_pr_submit`)**:
  * Upon submission of a PR, the associated PO's status is forced to **Completed**.
  * The linked Material Request's `custom_processing_status` automatically transitions to **Item Received**.
  * A timeline comment is posted on the MR: `"Status updated to Item Received as Purchase Order [PO_Name] is completed via Purchase Receipt [PR_Name]"`.

### 6. Asset
* **Definition**: The resulting asset record representing completion of the procurement phase.
* **State**: Contains serialization information, is linked to the Purchase Receipt, and displays the linked Material Request, Purchase Order, Purchase Receipt, and Asset Capitalization details dynamically via a premium HTML widget (`custom_procurement_html`).

### 7. Asset Capitalisation
* **Definition**: Signifies the actual **deployment** of the item received from the Purchase Receipt.
* **Validation / Linkage**:
  * Matches the received item from the PR to an Asset that shares the same Purchase Order (PO) reference.
* **Submit Hook (`on_asset_capitalization_submit`)**:
  * Upon submission of an Asset Capitalization, the linked Material Request's `custom_processing_status` automatically transitions to **Asset Capitalised**.
  * A timeline comment is posted on the MR: `"Status updated to Asset Capitalised upon submission of Asset Capitalization [AC_Name]"`.

### 8. Stock-led Asset Flow (Non-Procurement Flow)
* **Definition**: A simplified workflow where assets are created from stocked items (e.g. consumables/parts with `maintain_stock = 1`, `is_fixed_asset = 0`) instead of being purchased through the procurement pipeline (MR->RFQ->SQ->PO->PR).
* **Process**:
  1. Items are stocked in a warehouse via **Stock Entry** (SE) or Purchase Invoice.
  2. **Asset Capitalization** is created. Under the stock-items child table, the user selects the stocked item, consumption quantity, and source warehouse.
  3. Upon save/validation, the system enforces that a `target_asset_location` is supplied.
  4. If validation passes, a corresponding draft **Asset** document is automatically generated at the target location.
  5. Upon submitting the Asset Capitalization, the target asset is automatically submitted.
  6. Upon cancelling the Asset Capitalization, the target asset is automatically cancelled.
* **UI Dynamic Behavior**:
  * The global detail popup widget dynamically detects if the asset flow is stock-led (`is_stock_led = true` / absence of PO/PR references).
  * In stock-led flows, procurement-specific timeline cards (RFQ, SQ, PO, PR) are completely hidden, providing a clean user interface focusing only on consumption, capitalization details (including dimensions and installation photos), and asset creation.

---

## 4. Implementation Guidelines & File Locations

When maintaining or extending this flow, ensure compliance with these components:

* **Server-Side Hooks & Validation**:
  * **Unified Utilities**: Centralized in [utils.py](file:///workspace/development/frappe-bench/apps/swift_fix/swift_fix/setup/utils.py). Houses the core business logic, including `get_asset_html`, `get_historic_flow_details`, `get_procurement_details`, and state verification rules.
  * **Event Handlers**: Implemented in [popr_utils.py](file:///workspace/development/frappe-bench/apps/swift_fix/swift_fix/setup/popr_utils.py). Handles hooks and validations such as `on_po_submit`, `on_pr_submit`, `on_asset_capitalization_submit`, `generate_asset_qr`, `create_purchase_receipt_serial_nos`, and `check_purchase_invoice_capitalization` by delegating helper logic to `utils.py`.
  * **RFQ Validation & Handlers**: Located in [rfq_utils.py](file:///workspace/development/frappe-bench/apps/swift_fix/swift_fix/setup/rfq_utils.py). Handles RFQ save restrictions and timeline comments.
  * **Backend Utilities**: Located in [mr_utils.py](file:///workspace/development/frappe-bench/apps/swift_fix/swift_fix/setup/mr_utils.py). Contains functions such as `change_mr_status`, `has_active_po`, `has_completed_po`, `analyze_mr`, and `validate_mr`.
  * **Hooks Registry**: Registered in [hooks.py](file:///workspace/development/frappe-bench/apps/swift_fix/swift_fix/hooks.py) under the `doc_events` section for the `Material Request`, `Request for Quotation`, `Purchase Order`, `Purchase Receipt`, `Asset Capitalization`, `Asset`, and `Purchase Invoice` Doctypes.
* **Client-Side Scripts**:
  * Located in [client_script.json](file:///workspace/development/frappe-bench/apps/swift_fix/swift_fix/fixtures/client_script.json).
  * Controls dynamic UI banner updates (showing Material Request status in the HTML field without marking the form dirty) and client-side save/action button behaviors.
  * **Button Hiding Rules**: If a linked Purchase Order is in `'Completed'` status, the `'Change Status'` group containing `'Cancel'`, `'Hold'`, and `'Shortlist'` buttons is hidden completely on the Material Request form.
  * **Dynamic Analysis HTML**: Displays a summary of other Material Requests, assets, and a table of linked Purchase Orders with their names, transaction dates, statuses, and grand totals.
* **Custom Fields**:
  * Configured in [custom_field.json](file:///workspace/development/frappe-bench/apps/swift_fix/swift_fix/fixtures/custom_field.json).

---

## 5. REST API Integration & Postman Testing

To facilitate mobile client integration and testing, separate Postman API collections are provided for the two workflows:
* **Procurement-led Flow Collection**: [swift_fix_procurement_flow.postman_collection.json](file:///workspace/development/frappe-bench/agenticdocs/skills/swift_fix/resources/swift_fix_procurement_flow.postman_collection.json) or [local copy](file:///workspace/development/frappe-bench/.agents/skills/procurement-flow/resources/swift_fix_procurement_flow.postman_collection.json)
* **Stock-led Flow Collection**: [swift_fix_stock_led_flow.postman_collection.json](file:///workspace/development/frappe-bench/agenticdocs/skills/swift_fix/resources/swift_fix_stock_led_flow.postman_collection.json) or [local copy](file:///workspace/development/frappe-bench/.agents/skills/procurement-flow/resources/swift_fix_stock_led_flow.postman_collection.json)

### Collection Variables

The collection relies on the following Postman environment/collection variables:
* `baseUrl`: The base URL of the CMMS instance (default: `http://cmms.localhost`).
* `apiKey`: The API Key of the CMMS user.
* `apiSecret`: The API Secret of the CMMS user.
* `mrName`, `rfqName`, `poName`, `poItemName`, `prName`, `acName`: Dynamic resource names to chain workflow steps sequentially.

### Authentication

Token-based authentication is configured at the **Collection Level**. The collection automatically adds the standard header:
`Authorization: token {{apiKey}}:{{apiSecret}}`
to every request. Ensure `apiKey` and `apiSecret` variables are correctly populated in your environment before executing requests.

### Endpoint Categories
1. **Company**:
   * Create Company: `POST /api/resource/Company`
   * Create Supplier: `POST /api/resource/Supplier`
   * Create Warehouse: `POST /api/resource/Warehouse`
2. **Accounts**:
   * Create Asset Received Account: `POST /api/resource/Account`
   * Update Company Asset Received Account: `PUT /api/resource/Company/Sravi Enterprises - Assets Kolapakkam`
3. **Cost Center**:
   * Get Cost Centers: `GET /api/resource/Cost Center`
4. **Location**:
   * Create Location: `POST /api/resource/Location`
5. **Stock Item (with Serial Number)**:
   * Create Asset Category: `POST /api/resource/Asset Category`
   * Create Service Item: `POST /api/resource/Item`
   * Create Sample Item: `POST /api/resource/Item`
6. **MR**:
   * Create MR: `POST /api/resource/Material Request`
   * Submit MR: `PUT /api/resource/Material Request/{{mrName}}`
   * Get MR Status Details: `GET /api/method/swift_fix.setup.mr_utils.get_mr_status_details`
   * Change MR Status: `POST /api/method/swift_fix.setup.mr_utils.change_mr_status`
   * Analyze MR Location: `GET /api/method/swift_fix.setup.mr_utils.analyze_mr`
7. **SQ**:
   * Create Request for Quotation: `POST /api/resource/Request for Quotation`
   * Submit Request for Quotation: `PUT /api/resource/Request for Quotation/{{rfqName}}`
   * RFQ Update Dimensions: `POST /api/method/swift_fix.setup.rfq_utils.rfq_update_dimensions`
   * RFQ Update Dimensions with Images: `POST /api/method/swift_fix.setup.rfq_utils.rfq_update_dimensions_with_images`
   * RFQ Change Recce Status: `POST /api/method/swift_fix.setup.rfq_utils.rfq_change_recce_status`
   * Create Supplier Quotation: `POST /api/resource/Supplier Quotation`
   * Submit Supplier Quotation: `PUT /api/resource/Supplier Quotation/{{sqName}}`
8. **PO**:
   * Create Purchase Order: `POST /api/resource/Purchase Order`
   * Submit Purchase Order: `PUT /api/resource/Purchase Order/{{poName}}`
9. **PR**:
   * Create Purchase Receipt: `POST /api/resource/Purchase Receipt`
   * Submit Purchase Receipt: `PUT /api/resource/Purchase Receipt/{{prName}}`
10. **Asset Capitalization**:
   * Create Asset Capitalization: `POST /api/resource/Asset Capitalization`
   * Submit Asset Capitalization: `PUT /api/resource/Asset Capitalization/{{acName}}`
11. **Asset**:
   * Get Assets by PR Reference: `GET /api/resource/Asset?filters=[["purchase_receipt", "=", "{{prName}}"]]`
   * Submit Asset manually: `PUT /api/resource/Asset/{{assetName}}`
   * Get Stock Ledger Entries for Receipt: `GET /api/resource/Stock Ledger Entry?filters=[["voucher_no", "=", "{{prName}}"]]`
   * Get Item details: `GET /api/resource/Item/MBLIT`
   * Get Procurement HTML Widget: `GET /api/method/swift_fix.setup.utils.get_procurement_details?asset_name={{assetName}}`
12. **Stock-led Flow (Stock Entry & Capitalization)**:
   * Create Stock Entry (Material Receipt): `POST /api/resource/Stock Entry`
   * Submit Stock Entry: `PUT /api/resource/Stock Entry/{{seName}}`
   * Create Stock-led Asset Capitalization: `POST /api/resource/Asset Capitalization`
   * Submit Stock-led Asset Capitalization: `PUT /api/resource/Asset Capitalization/{{acName}}`
