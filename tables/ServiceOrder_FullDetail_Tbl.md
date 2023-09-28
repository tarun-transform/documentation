# ServiceOrder_FullDetail_Tbl

## Overview

This documentation outlines the structure and usage of the `ServiceOrder_FullDetail_Tbl` table in the `IH_DataScience` database within the `HS_PRICING` schema. 

This table is used for managing and tracking service orders in a home parts service system.

## Table Update

- The view updates approximately daily.

## Geography

- The United States is grouped into 29 districts including the mainland (27 districts), Puerto Rico, and Hawaii.

## Table Primary Key

- The primary key of the `ServiceOrder_FullDetail_Tbl` table is composed of `SVC_UN_NO` (for district) and `SO_NO` (service order).
  - The service number is unique only within each district.

## Columns Description

### General Information

- **`Servicer`**: Indicates who actually worked on the service order.
  - `W2`: Internal technician.
  - `ISP`: Contractor.
- **`Pay_Met2`**: Describes the demand channel.
  - `W2`: Sears / A&E (This is a B2B payment model).
- **`ServiceType`**:
  - `RPR`: Repair.
  - `PM`: Preventive Maintenance.

### Service Details

- **`SVC_LOC`**: Specifies the service location.
  - `site`: House.
  - `shop`: Sears service store or business customer like Best Buy.
- **`SVC_CUS_ID_NO`**: Customer to ID relationship (one-to-many), perhaps due to multiple signups through different channels.
- **`ORI_THD_PTY_ID`** and **`THD_PTY_ID`**: Original and ending process IDs to determine the type of call.
  - D2C call: Customer pays.
  - B2B call: Business files a claim.

### Warranty and Recall Information

- Warranty period ranges from 30 to 90 days.
- **`RECALL_PARENT`**: Initial service.
- **`RECALL_CHILD`**: Follow-up service.
- **`PM_CHK_CD`**:
  - `P`: Preventive Maintenance.
  - `R`: Repair.

### Product Information

- **`SO_PRI_CD`**: Emergency flag.
- **`MFG_BND_NM`**: Manufacturer brand name.
- **`PSV_ITM_MDL_NO`**: Model number.
- **`SRL_NO`**: Serial number for the item.
- **`SL_DT`**: Sale date.

### Scheduling and Routing Information

- **`PLANNING_AREA`**: Used for the routing of technicians.
- **`CHANNEL`**: How the service was created.
- **`CRT_DT`**: Service create date.
- **`FIRST_AVL_DT`**: First available date displayed.
- **`SCH_DT`**: Shows the most recent schedule date (useful if there are multiple attempts made).

### Status Codes

- **`SO_STS_CD`**: Status code of the service order.
  - `CO`: Complete.
  - `ED`: Estimate Decline.
  - `CA`: Cancellation.
  - Other codes represent open status.
- **`COMPLETE CODE`**: Any code ending with `0`, for example, `20` for B2B.

### Timing Information

- **`CYCLE_TIME`**: Time between request creation and completion.
- **`N_ATP`**: Number of attempts.
- **`N_NH`**: Not home.
- **`N_OP`**: Order parts.
- **`TST`**: Transit time.
- **`SVC`**: Service time.

## Conclusion

This documentation provides a comprehensive view of the `ServiceOrder_FullDetail_Tbl`, which plays a crucial role in managing service orders within the home parts service system. Proper understanding and usage of this table will ensure efficient and effective service order management.
