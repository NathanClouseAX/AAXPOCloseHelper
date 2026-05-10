# AAXPOCloseHelper

A Dynamics 365 Finance and Operations (D365FO) model that provides bulk cancellation of remaining physical quantities on open purchase order lines. It allows users to find open (backorder) purchase lines filtered by date range and cancel their remaining quantities in bulk, advancing the purchase order status without manually editing each line.

**Publisher:** www.atomicax.com
**Version:** 1.0.0
**License:** GPL-3.0

## Overview

In standard D365FO, cancelling the remainder on purchase order lines must be done one line at a time through the "Update remaining quantity" form. This model adds an **Open Purchase Line Maintenance** form that displays all backorder purchase lines with non-zero remaining physical quantities, and provides a **Cancel Remainder** action that can be applied to one or many selected lines at once.

The cancellation process handles:
- **Intercompany synchronization** via `InterCompanyUpdateRemPhys::synchronize`
- **Landed Cost (ITM) ship line updates** via reflection-based call to `ITMUpdateShipLine`
- **MCR Call Center order cancellation** workflows including line cancellation, order cancellation, and payment adjustments
- **Drop-shipment scenarios** by zeroing all remaining quantity fields
- **Per-line error isolation** so that failures on individual lines do not block processing of other selected lines

## Module Dependencies

ApplicationCommon, ApplicationFoundation, ApplicationPlatform, ApplicationSuite, ContactPerson, Directory, FiscalBooks, Ledger, Retail, SourceDocumentation, SourceDocumentationTypes, Tax

## Navigation

The form is accessible from two locations in the D365FO menu:

- **Accounts Payable > Periodic Tasks > Cleanup > Open Purchase Line Maintenance**
- **Procurement and Sourcing > Cleanup > Open Purchase Line Maintenance**

## Source Code Reference

### Classes

#### AAXPurchLineClose (`AxClass/AAXPurchLineClose.xml`)

Provides bulk cancellation of remaining physical quantities on open purchase order lines. Invoked as an action menu item from the AAXPurchLineClose form, supports multi-select to cancel remainders across many lines in a single operation. Handles intercompany synchronization, Landed Cost (ITM) ship line updates, and MCR Call Center order cancellation workflows.

| Method | Visibility | Description |
|---|---|---|
| `construct()` | public static | Constructs a new instance of the `AAXPurchLineClose` class. |
| `main(Args args)` | public static | Entry point invoked by the AAXPurchLineClose action menu item. Iterates over all selected records in the AAXPurchLineOpen form datasource using `MultiSelectionHelper` and cancels the remaining physical quantity on each corresponding `PurchLine`. Refreshes the calling form after processing. |
| `runPurchLine(PurchLine)` | public | Cancels the remaining physical quantity on a single purchase order line. Sets `RemainPurchPhysical` to zero, recalculates `RemainInventPhysical` via `calcQtyOrdered`, synchronizes intercompany remaining quantities via `InterCompanyUpdateRemPhys::synchronize`, writes the updated line, and conditionally triggers a Landed Cost (ITM) ship line update via reflection. Each line is processed in its own `ttsbegin`/`ttscommit` transaction; errors are caught and logged as warnings so that other selected lines continue processing. |
| `updateRemainPhysical(PurchLine)` | protected | Updates the remaining physical quantity on a purchase line with full intercompany and MCR Call Center cancellation support. Synchronizes intercompany remaining quantities, handles drop-shipment scenarios by zeroing all remaining fields (`RemainPurchPhysical`, `RemainInventPhysical`, `PdsCWRemainInventPhysical`), writes the line, updates the purchase order back-order status, and processes MCR sales order cancellations and payment adjustments. |
| `mcrSalesOrderCancelInit(TradeInventTransId)` | private | Initializes the MCR Call Center sales order cancellation process. If the MCR Call Center configuration key is enabled and the purchase line is linked to a sales line, creates an `MCRSalesOrderCancellation` instance and captures the pre-cancellation state of the sales order. Returns null if MCR is disabled or no linked sales line exists. |
| `mcrSalesOrderCancel(MCRSalesOrderCancellation, TradeInventTransId)` | private | Completes the MCR Call Center sales order cancellation workflow after a purchase line remainder has been cancelled. If the linked sales line status has become `Canceled`, posts the line-level cancellation. If the entire sales order is also `Canceled`, posts the order-level cancellation. Always posts the cancellation payment adjustment to handle any payment holds or refunds on the sales order. |

#### PurchLine_AAXPOCloseHelper_Extension (`AxClass/PurchLine_AAXPOCloseHelper_Extension.xml`)

Extension of the `PurchLine` table (using the `[ExtensionOf(tableStr(PurchLine))]` pattern). Provides a stub method for Landed Cost (ITM) ship line update integration that can be enabled when the ITM module is present.

| Method | Visibility | Description |
|---|---|---|
| `AAXUpdateShipLine(PurchLine, PurchLine, boolean)` | static | Placeholder for Landed Cost (ITM) ship line synchronization. When the ITM module is installed, uncomment the inner call to `PurchLine::ITMUpdateShipLine` to synchronize ship line quantities when purchase line remainders are cancelled. Parameters: the updated `PurchLine` buffer, the original `PurchLine` buffer before modification, and a force flag to override change detection. |

### View

#### AAXPurchLineOpen (`AxView/AAXPurchLineOpen.xml`)

Database view that surfaces open (backorder) purchase lines with their associated purchase order header data. Filtered to `PurchStatus = Backorder`. Provides computed columns for expected delivery date and display methods for item name and catch weight unit. Used as the primary data source for the AAXPurchLineClose bulk cancellation form.

**Data Sources:** `PurchLine` (root) joined to `PurchTable` (on `PurchId`).

**Filter:** `PurchLine.PurchStatus = Backorder`

| Field | Type | Source | Description |
|---|---|---|---|
| `PurchId` | Bound | PurchLine | Purchase order number |
| `InventTransId` | Bound | PurchLine | Inventory transaction ID (lot ID) |
| `InventDimId` | Bound | PurchLine | Inventory dimension ID |
| `ConfirmedDlv` | Bound | PurchLine | Confirmed delivery date |
| `ProjId` | Bound | PurchLine | Project ID |
| `VendAccount` | Bound | PurchLine | Vendor account number |
| `ItemId` | Bound | PurchLine | Item number |
| `ProcurementCategory` | Bound | PurchLine | Procurement category |
| `PurchUnit` | Bound | PurchLine | Purchase unit of measure |
| `PurchQty` | Bound | PurchLine | Original purchase quantity |
| `RemainPurchPhysical` | Bound | PurchLine | Remaining physical quantity in purchase units |
| `LineNumber` | Bound | PurchLine | Line number |
| `PdsCWQty` | Bound | PurchLine | Catch weight quantity |
| `PdsCWRemainInventPhysical` | Bound | PurchLine | Catch weight remaining inventory physical |
| `PurchName` | Bound | PurchTable | Vendor name from purchase order header |
| `ExpectedDate` | Computed | `expectedDateDefinition()` | Returns `ConfirmedDlv` if non-null, otherwise `DeliveryDate` |
| `DeliveryDate` | Bound | PurchLine | Requested delivery date |
| `PurchLineRecId` | Bound | PurchLine.RecId | Record ID of the purchase line (used to look up the actual `PurchLine` record for updates) |

**Methods:**

| Method | Type | Description |
|---|---|---|
| `expectedDateDefinition()` | Computed column | Defines the `ExpectedDate` computed column. Returns the confirmed delivery date of the purchase order line if there is one defined; otherwise, returns the requested delivery date. |
| `itemName()` | Display | Gets the order line item name according to its inventory dimensions via `InventTable.itemName()`. |
| `pdsCWUnitId()` | Display | Gets the catch weight unit for the item via `PdsCatchWeight::cwUnitId()`. |

### Query

#### AAXPurchLineOpen (`AxQuery/AAXPurchLineOpen.xml`)

Query definition backing the AAXPurchLineOpen view. Selects from `PurchLine` (filtered to `PurchStatus = Backorder`) joined to `PurchTable` on `PurchId`. Exposes fields from both tables to support filtering, display, and lookup in the AAXPurchLineClose form.

### Form

#### AAXPurchLineClose (`AxForm/AAXPurchLineClose.xml`)

Simple List form that displays open (backorder) purchase lines with non-zero remaining physical quantities. Provides date range filters and a Cancel Remainder action for bulk cancellation. Uses the AAXPurchLineOpen view as its primary data source. Default date filter shows lines with expected dates up to 3 months before today.

**Form Variables:**

| Variable | Type | Default | Description |
|---|---|---|---|
| `fromDate` | Date | `dateNull()` (no lower bound) | Start of the expected date filter range |
| `toDate` | Date | 3 months ago from today | End of the expected date filter range |
| `orderTypeListPage` | PurchLineDeliveryType | `Receipts` | Delivery type filter (not currently applied as a range) |
| `inventDimFormSetup` | InventDimCtrl_Frm_ActiveRightClick | null | Inventory dimension display controller |

**Data Sources:**

| Name | Table | Join | Create | Delete | Insert If Empty |
|---|---|---|---|---|---|
| `AAXPurchLineOpen` | AAXPurchLineOpen (view) | Root | No | No | No |
| `InventDim` | InventDim | Inner join to AAXPurchLineOpen | No | No | - |
| `PurchTable` | PurchTable | Inner join to AAXPurchLineOpen | - | - | - |

**Form Methods:**

| Method | Description |
|---|---|
| `init()` | Initializes the form and sets up the inventory dimension display controls via `updateDesign`. |
| `updateDesign(InventDimFormDesignUpdate)` | Configures the inventory dimension grid display controls based on the specified update mode. On initialization, sets up the `InventDimCtrl_Frm_ActiveRightClick` controller with default dimension visibility from inventory transaction parameters. |
| `inventDimSetupObject()` | Returns the inventory dimension form setup controller object. Required by the D365FO inventory dimension framework for coordinating dimension display across the form. |

**Data Source Methods (AAXPurchLineOpen):**

| Method | Description |
|---|---|
| `init()` | Initializes the AAXPurchLineOpen data source. Sets default values for the from/to date filter controls and applies the initial hidden query ranges for expected date and non-zero remaining quantity. |
| `research(boolean)` | Re-applies the date and remaining quantity filter ranges on the runtime query before executing the research. Ensures filters reflect the current `fromDate` and `toDate` values after user changes. |
| `findOrCreateDateAndOrderTypeRanges(QueryBuildDataSource)` | Creates or updates hidden query ranges on the AAXPurchLineOpen data source for the expected date range (`fromDate` to `toDate`) and remaining physical quantity (non-zero only). Clears and recreates ranges if they do not already exist. Both ranges are hidden from the user to prevent manual modification. |

**Filter Controls:**

| Control | Type | Description |
|---|---|---|
| `QuickFilterControl` | Quick Filter | Filters the `GridBackOrderLines` grid; defaults to the `VendAccount` column. |
| `OrderType` | ComboBox (`PurchLineDeliveryType`) | Updates the delivery type filter value and refreshes the data source query on selection change. |
| `BackorderDateFromDateEdit` | Date (`SalesLineDlvDate`) | Updates the `fromDate` variable and refreshes the data source query on modification. |
| `BackorderDateToDateEdit` | Date (`SalesLineDlvDate`) | Updates the `toDate` variable and refreshes the data source query on modification. |

**Grid Columns:** Purchase ID, Line Number, Expected Date, Delivery Date, Confirmed Delivery, Vendor Account, Purchase Name (PurchTable), Sales ID, Item ID, Inventory Dimensions (config, size, color, style, site, warehouse, batch, WMS location, serial, status, license plate, owner, profile, GTD), Catch Weight group, Item Name, Sales Unit, Purchase Qty, Remaining Physical Qty.

**Action Pane:**

| Tab | Group | Button | Description |
|---|---|---|---|
| Purchase | Maintain | Edit | Opens the purchase order form in edit mode via `PurchTableFromPurchBackOrderLinesForEdit`. |
| Purchase | Maintain | Cancel Remainder | Invokes `AAXPurchLineClose` action menu item. Supports multi-select. |
| Purchase | Inventory | On-hand | Opens the on-hand inventory form (`InventOnhand`). |
| Purchase | Inventory | Inventory transactions | Opens inventory transactions (`InventTrans`). |
| Purchase | Item information | Lot | Opens lot/batch details (`InventLot`). |
| General | Setup | Dimensions display | Opens the inventory dimension display configuration (`InventDimParmFixed`). |

### Menu Items

#### AAXPurchLineClose (`AxMenuItemAction/AAXPurchLineClose.xml`)

Action menu item that invokes the `AAXPurchLineClose` class.

| Property | Value |
|---|---|
| Type | Action |
| Label | Cancel Remainder |
| Help Text | Cancel remainder on line purchase line to advance the status |
| Object Type | Class |
| Object | AAXPurchLineClose |
| Subscriber Read Access | Allow |

#### AAXPurchLineCleanup (`AxMenuItemDisplay/AAXPurchLineCleanup.xml`)

Display menu item that opens the `AAXPurchLineClose` form.

| Property | Value |
|---|---|
| Type | Display |
| Label | Open Purchase Line Maintenance |
| Help Text | Perform Bulk Operations on Open Purchase Lines |
| Object | AAXPurchLineClose |
| Subscriber Read Access | Allow |

### Menu Extensions

#### AccountsPayable.AAXPOCloseHelper (`AxMenuExtension/AccountsPayable.AAXPOCloseHelper.xml`)

Adds the `AAXPurchLineCleanup` display menu item under **Accounts Payable > Periodic Tasks > Cleanup** as a submenu named "AAXCleanup".

#### ProcurementAndSourcing.AAXPOCloseHelper (`AxMenuExtension/ProcurementAndSourcing.AAXPOCloseHelper.xml`)

Adds the `AAXPurchLineCleanup` display menu item directly under **Procurement and Sourcing > Cleanup**.

### Security

#### AAXPurchLineClose Privilege (`AxSecurityPrivilege/AAXPurchLineClose.xml`)

A security privilege granting full access (Read, Create, Update, Delete, Correct) to both entry points:

| Entry Point | Object Type | Object Name | Form |
|---|---|---|---|
| AAXPurchLineCleanup | MenuItemDisplay | AAXPurchLineCleanup | AAXPurchLineClose |
| AAXPurchLineClose | MenuItemAction | AAXPurchLineClose | - |

#### PurchOrderMaintain.AAXPOCloseHelper Duty Extension (`AxSecurityDutyExtension/PurchOrderMaintain.AAXPOCloseHelper.xml`)

Extends the standard `PurchOrderMaintain` security duty to include the `AAXPurchLineClose` privilege. Users with the "Maintain purchase orders" duty automatically gain access to this feature.

### Labels

#### AAXPOCloseHelper (`AxLabelFile/AAXPOCloseHelper_en-US.xml`)

English (en-US) label file. Label file ID: `AAXPOCloseHelper`.

| Label ID | Text |
|---|---|
| `AAXPOCloseHelperCancelRemainder` | Cancel Remainder |
| `AAXPOCloseHelperCancelRemainderHelp` | Cancel remainder on line purchase line to advance the status |
| `AAXPOCloseHelperCancelError` | Purchase Order %1, Line %2, Item %3, Lot Id %4 encountered an error and could not be cancelled. |
| `AAXPOCloseHelperOpenLineMaintenance` | Open Purchase Line Maintenance |
| `AAXPOCloseHelperOpenLineMaintenanceHelp` | Perform Bulk Operations on Open Purchase Lines |

## Project Structure

```
AAXPOCloseHelper/
  AxClass/
    AAXPurchLineClose.xml              # Main cancel-remainder business logic
    PurchLine_AAXPOCloseHelper_Extension.xml  # PurchLine table extension (ITM stub)
  AxForm/
    AAXPurchLineClose.xml              # Simple List form for viewing/cancelling lines
  AxLabelFile/
    AAXPOCloseHelper_en-US.xml         # Label file metadata
    LabelResources/en-US/
      AAXPOCloseHelper.en-US.label.txt # English label strings
  AxMenuExtension/
    AccountsPayable.AAXPOCloseHelper.xml       # AP menu integration
    ProcurementAndSourcing.AAXPOCloseHelper.xml # P&S menu integration
  AxMenuItemAction/
    AAXPurchLineClose.xml              # Action menu item (triggers class)
  AxMenuItemDisplay/
    AAXPurchLineCleanup.xml            # Display menu item (opens form)
  AxQuery/
    AAXPurchLineOpen.xml               # Query: backorder PurchLines + PurchTable
  AxSecurityDutyExtension/
    PurchOrderMaintain.AAXPOCloseHelper.xml  # Duty extension for PurchOrderMaintain
  AxSecurityPrivilege/
    AAXPurchLineClose.xml              # Security privilege with full CRUD access
  AxView/
    AAXPurchLineOpen.xml               # View over backorder purchase lines
Descriptor/
  AAXPOCloseHelper.xml                 # Model descriptor (ID: 895972602, Layer: 14)
DeployablePackage/
  AXDeployablePackage_*.zip            # Pre-built deployable package
Model/
  AAXPOCloseHelper-www.atomicax.com.axmodel  # Compiled model file
Projects/
  AAXPOCloseHelper/                    # Visual Studio solution and project files
axpp/
  AAXPOCloseHelper.axpp               # Exportable project package
```

## Installation

1. **From Source:** Copy the `AAXPOCloseHelper` folder into your `PackagesLocalDirectory` and build the model in Visual Studio.

After installation, perform a full build and database synchronization.

## Usage

1. Navigate to **Procurement and Sourcing > Cleanup > Open Purchase Line Maintenance** (or via Accounts Payable).
2. Adjust the **From date** and **To date** filters to narrow results by expected delivery date. The default range is from the earliest date up to 3 months before today.
3. Use the quick filter (defaults to Vendor Account) or column filters to find specific lines.
4. Select one or more lines in the grid (multi-select is supported).
5. Click **Cancel Remainder** on the action pane.
6. The remaining physical quantity on each selected line is set to zero, intercompany quantities are synchronized, and the purchase order status advances accordingly.
7. Lines that encounter errors during processing will show a warning with the PO number, line number, item ID, and lot ID; other lines in the selection continue to process.
