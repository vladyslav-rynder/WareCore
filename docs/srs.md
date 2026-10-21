# Software Requirements Specification (SRS)
## Project: WareCore (WMS) v1.0.0

## 1. Introduction

### 1.1 Purpose

The purpose of WareCore is to automate core warehouse inventory operations for a single facility. The system enables warehouse topology configuration, reliable item/location identification, cycle counting with discrepancy approval workflows, and immutable transaction audit logging.

### 1.2 Scope (MVP)

The initial scope (v1.0.0) focuses on single-warehouse operations and includes:
- **Warehouse Topology & User Management:** Configuration of structural zones, racks, and storage bins, alongside Role-Based Access Control (RBAC) for Workers, Managers, and Administrators.
- **Product & Bin Identification:** Manual lookup and resolution for SKUs, barcodes, and bin locations with input validation.
- **Cycle Counting:** Execution of physical bin counts and automated discrepancy calculation.
- **Inventory Adjustment Workflow:** Managerial approval or rejection queue for stock adjustments with concurrency guards.
- **Immutable Audit Trail:** Append-only event logging for all stock changes and supervisory decisions.

## 2. System Roles (RBAC)

1. **Warehouse Worker:** Manages inbound goods, executes putaway, and performs manual item identification and counting.
2. **Manager:** Oversees stock levels, initiates cycle counts, reviews discrepancies, and approves or rejects inventory adjustments.
3. **Administrator:** Manages users, roles, and warehouse structural topology (zones, racks, and bins).

## 3. Functional Requirements (User Stories & Acceptance Criteria)

### FR-00: Warehouse Topology & User Management (Administrator Setup)

#### User Story 1: Warehouse Topology Setup

**User Story**

As an Administrator,  
I want to create and manage warehouse structural units (zones, racks, and storage bins),  
So that the system accurately mirrors physical storage space for inventory management.

**Acceptance Criteria**

- **AC-01: Create Storage Bin**
  - Given an Administrator is on the topology configuration screen,
  - When they submit a unique bin identifier (e.g., `A-01-02-01`), zone, and capacity constraints,
  - Then the system validates uniqueness and creates the storage location in an active state.

- **AC-02: Prevent Duplicate Bins**
  - Given a storage bin identifier already exists in the system,
  - When an Administrator attempts to create another bin with the same identifier,
  - Then the system rejects the operation and displays a duplicate location error.

- **AC-03: Deactivate / Lock Bin**
  - Given an existing storage bin,
  - When an Administrator marks the bin as inactive or locked for maintenance,
  - Then the system prevents new inventory tasks from assigning stock to that bin until it is reactivated.

#### User Story 2: User Role Assignment & Access Control (RBAC)

**User Story**

As an Administrator,  
I want to create user accounts and assign specific roles (Worker, Manager, Admin),  
So that access to operational and managerial functions is strictly controlled.

**Acceptance Criteria**

- **AC-01: User Creation & Role Assignment**
  - Given an Administrator,
  - When they create a new user account and assign a role (`Warehouse Worker`, `Manager`, or `Administrator`),
  - Then the system grants access rights corresponding strictly to that assigned role.

- **AC-02: Revoke or Update Role**
  - Given an existing user account,
  - When an Administrator changes or revokes the assigned role,
  - Then the updated permission rules apply immediately to subsequent system actions.

### FR-01: Manual Barcode / SKU & Bin Identification Fallback

**User Story**

As a Warehouse Worker,  
I want to manually input a barcode/SKU string or storage bin code,  
So that I can identify items and locations when scanner devices fail or labels are damaged.

**Acceptance Criteria**

- **AC-01: Valid Product Identification**
  - Given the user is on an execution screen,
  - When they submit a registered SKU or barcode,
  - Then the system resolves the item, displays its core metadata, and allows proceeding to the next step.

- **AC-02: Invalid Product Identification**
  - Given an unregistered SKU or barcode is submitted,
  - When the input is processed,
  - Then the system rejects the operation and displays an "Item Not Found" error message.

- **AC-03: Storage Bin Validation**
  - Given an operation requiring a storage location,
  - When a user submits a bin code,
  - Then the system validates that the bin exists, is active, and is not locked for maintenance/counting.

### FR-02: Inventory Cycle Counting

**User Story**

As a Manager or Worker,  
I want to register physical count results for a specific bin and product,  
So that stock discrepancies can be identified and submitted for approval.

**Acceptance Criteria**

- **AC-01: Submit Physical Count**
  - Given an active bin and product selection,
  - When the user submits a non-negative physical count quantity,
  - Then the system calculates the discrepancy as physical count minus system quantity.

- **AC-02: Discrepancy Handling (Pending Approval)**
  - Given a cycle count yields a non-zero discrepancy,
  - When the count is submitted,
  - Then the system creates an Inventory Adjustment request with status `PendingApproval`, keeping current recorded stock unchanged.

- **AC-03: Zero Discrepancy Auto-Resolution**
  - Given a physical count matches the system quantity exactly,
  - When the count is submitted,
  - Then the system updates the last-counted timestamp for the bin/product and logs a verified audit entry without creating an adjustment request.

- **AC-04: Location Concurrency Guard**
  - Given an active `PendingApproval` adjustment exists for a product in a bin,
  - When another user attempts to initiate a new count or stock movement for the same bin/product,
  - Then the system blocks the operation until the pending adjustment is resolved.

- **AC-05: Input Validation**
  - Given a physical count entry form,
  - When a user submits a negative number or non-numeric value,
  - Then the system prevents submission and displays a validation error.

### FR-03: Inventory Adjustment Approval Workflow

**User Story**

As a Manager,  
I want to review pending inventory adjustments and either approve or reject them,  
So that stock balances stay accurate while maintaining operational control.

**Acceptance Criteria**

- **AC-01: Pending Adjustment Queue**
  - Given unprocessed cycle count discrepancies exist,
  - When a Manager opens the adjustment management interface,
  - Then the system presents a list of `PendingApproval` requests with details including product, bin, system quantity, physical quantity, discrepancy, submitted by, and timestamp.

- **AC-02: Approve Adjustment**
  - Given a pending adjustment request,
  - When an authorized Manager approves it,
  - Then the system updates the recorded inventory quantity to the physical count, marks the adjustment as `Approved`, releases any location locks, and logs the decision.

- **AC-03: Reject Adjustment**
  - Given a pending adjustment request,
  - When an authorized Manager rejects it and provides a mandatory reason,
  - Then the system marks the adjustment as `Rejected`, preserves the original system inventory, releases location locks, and logs the reason.

- **AC-04: Idempotency / Duplicate Execution Guard**
  - Given an adjustment request has already been marked as `Approved` or `Rejected`,
  - When a concurrent or duplicate request attempts to change its state,
  - Then the system rejects the request with a state-conflict notification without making further inventory modifications.

### FR-04: Immutable Audit Logging

**User Story**

As a Manager or Administrator,  
I want the system to maintain an unalterable trail of all stock movements and adjustments,  
So that full traceability is guaranteed for compliance and investigation.

**Acceptance Criteria**

- **AC-01: Operation Event Capture**
  - Given any stock-modifying operation or approval decision succeeds,
  - When the transaction commits,
  - Then the system writes an event record containing event ID, event type, user ID, product ID, bin ID, quantity change, resulting quantity, reason if applicable, and timestamp.

- **AC-02: Immutability Guarantee**
  - Given an audit record has been written,
  - When system state changes or inventory records are modified,
  - Then existing audit log entries remain unmodified and cannot be updated or deleted via application interfaces.

### FR-05: Product Catalog Management

**User Story**

As a Manager,  
I want to create and manage product records,  
So that warehouse operations can identify and track products consistently.

**Acceptance Criteria**

- **AC-01: Create Product**
  - Given a Manager is on the product management interface,
  - When they submit a unique SKU, product name, and unit of measure,
  - Then the system creates the product record in an active state.

- **AC-02: Prevent Duplicate SKU**
  - Given a product with the submitted SKU already exists,
  - When a Manager attempts to create another product with the same SKU,
  - Then the system rejects the operation and displays a duplicate SKU error.

- **AC-03: Barcode Assignment**
  - Given a product record exists,
  - When a Manager assigns a barcode to the product,
  - Then the system validates barcode uniqueness and saves the barcode if it is not assigned to another product.

- **AC-04: Update Product Details**
  - Given an existing product record,
  - When a Manager updates its name, barcode, or other permitted attributes,
  - Then the system saves the updated details without altering existing inventory quantities or historical records.

- **AC-05: Deactivate Product**
  - Given an existing product record,
  - When a Manager deactivates the product,
  - Then the system prevents the product from being used in new inventory operations while preserving its existing inventory records and historical data.

### FR-06: Inventory Visibility

**User Story**

As a Warehouse Worker or Manager,  
I want to view and search current inventory by product and storage bin,  
So that I can identify available stock quantities and their storage locations.

**Acceptance Criteria**

- **AC-01: View Inventory**
  - Given the user has permission to view inventory,
  - When they open the inventory interface,
  - Then the system displays inventory records including product SKU, product name, storage bin, and recorded quantity.

- **AC-02: Search Inventory**
  - Given inventory records exist,
  - When the user searches by product SKU, barcode, or product name,
  - Then the system displays matching inventory records.

- **AC-03: View Product Across Multiple Bins**
  - Given a product is stored in multiple bins,
  - When the user views that product's inventory,
  - Then the system displays the recorded quantity for each bin separately.

- **AC-04: View Total Product Quantity**
  - Given a product has inventory records in multiple bins,
  - When the user requests the product's total recorded quantity,
  - Then the system displays the sum of its recorded quantities across all bins.

- **AC-05: Empty Search Results**
  - Given no inventory records match the search criteria,
  - When the search is submitted,
  - Then the system displays an appropriate message indicating that no matching inventory was found.

### FR-07: Goods Receiving

**User Story**

As a Warehouse Worker,  
I want to register incoming goods and their quantities,  
So that received stock is recorded and made available for subsequent warehouse operations.

**Acceptance Criteria**

- **AC-01: Register Goods Receipt**
  - Given an active product exists,
  - When a Warehouse Worker submits a goods receipt with a valid product identifier and positive quantity,
  - Then the system records the receipt with the product, quantity, responsible user, and timestamp.

- **AC-02: Validate Received Product**
  - Given a Warehouse Worker submits a goods receipt,
  - When the specified product does not exist or is inactive,
  - Then the system rejects the operation and displays an appropriate validation error.

- **AC-03: Assign Storage Bin**
  - Given a valid goods receipt has been initiated,
  - When the Warehouse Worker selects an active and available storage bin,
  - Then the system accepts the bin assignment for the received goods.

- **AC-04: Update Inventory**
  - Given a valid goods receipt has been completed and a storage bin assigned,
  - When the receipt is confirmed,
  - Then the system increases the recorded inventory quantity for the specified product and bin by the received quantity.

- **AC-05: Validate Received Quantity**
  - Given a goods receipt form,
  - When a user submits a zero, negative, or invalid quantity,
  - Then the system rejects the submission and displays a validation error.

- **AC-06: Audit Goods Receipt**
  - Given a goods receipt is successfully completed,
  - When the inventory quantity is updated,
  - Then the system records an immutable audit entry containing the operation type, product, storage bin, quantity received, responsible user, and timestamp.

### FR-08: Stock Putaway & Transfer

**User Story**

As a Warehouse Worker,  
I want to put away received goods and transfer stock between storage bins,  
So that inventory quantities accurately reflect the physical distribution of products within the warehouse.

**Acceptance Criteria**

- **AC-01: Put Away Received Goods**
  - Given received goods are awaiting storage placement,
  - When a Warehouse Worker assigns the goods to an active and available storage bin and confirms the operation,
  - Then the system records the goods in the assigned bin and updates the corresponding inventory quantity.

- **AC-02: Transfer Stock Between Bins**
  - Given a product has a positive recorded quantity in a source bin,
  - When a Warehouse Worker submits a transfer with a valid source bin, destination bin, and positive quantity,
  - Then the system decreases the source bin quantity and increases the destination bin quantity by the same amount.

- **AC-03: Validate Available Quantity**
  - Given a transfer request is submitted,
  - When the requested transfer quantity exceeds the recorded quantity in the source bin,
  - Then the system rejects the operation and leaves both inventory quantities unchanged.

- **AC-04: Validate Storage Locations**
  - Given a transfer request specifies source and destination bins,
  - When either bin does not exist, is inactive, or is unavailable for the operation,
  - Then the system rejects the transfer and displays an appropriate validation error.

- **AC-05: Prevent Invalid Transfers**
  - Given a transfer request is submitted,
  - When the source and destination bins are identical or the requested quantity is invalid,
  - Then the system rejects the operation without changing inventory quantities.

- **AC-06: Atomic Stock Transfer**
  - Given a valid stock transfer is initiated,
  - When the transfer cannot be completed successfully,
  - Then the system ensures that neither the source nor the destination inventory quantity is partially updated.

- **AC-07: Audit Stock Movement**
  - Given a stock transfer is successfully completed,
  - When the inventory quantities are updated,
  - Then the system records an immutable audit entry containing the product, source bin, destination bin, transferred quantity, responsible user, and timestamp.

### FR-09: Inventory Integrity Rules

**User Story**

As a Manager,  
I want inventory-changing operations to follow consistent integrity rules,  
So that recorded stock quantities remain accurate and warehouse operations do not create inconsistent inventory states.

**Acceptance Criteria**

- **AC-01: Prevent Negative Inventory**
  - Given an inventory-changing operation is requested,
  - When the operation would result in a negative recorded quantity in any affected bin,
  - Then the system rejects the operation and preserves the existing inventory quantities.

- **AC-02: Restrict Inventory Modifications**
  - Given a user attempts to modify inventory quantities,
  - When the requested change is not part of an authorized inventory operation,
  - Then the system rejects the operation without changing the recorded inventory.

- **AC-03: Atomic Inventory Updates**
  - Given an operation requires changes to multiple inventory records,
  - When any part of the operation fails,
  - Then the system ensures that all inventory changes from that operation are rolled back.

- **AC-04: Enforce Pending Adjustment Restrictions**
  - Given a `PendingApproval` inventory adjustment exists for a specific product and bin,
  - When a user attempts another inventory count or stock movement affecting that product and bin,
  - Then the system rejects the conflicting operation until the adjustment is approved or rejected.

- **AC-05: Maintain Inventory Audit Trail**
  - Given an inventory-changing operation is successfully completed,
  - When the operation changes any recorded inventory quantity,
  - Then the system creates an immutable audit entry identifying the operation, affected product and bin, quantity change, responsible user, and timestamp.

- **AC-06: Preserve Inventory on Failed Operations**
  - Given an inventory-changing operation fails validation or cannot be completed,
  - When the operation is rejected,
  - Then the system preserves the inventory quantities and does not record the operation as successfully completed.