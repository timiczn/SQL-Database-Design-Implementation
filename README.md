# GrandStay Hotels & Resorts — SQL Database Design & Implementation

### Overview
This project is a full relational database design and implementation for **GrandStay Hotels & Resorts**, a hospitality group operating across six international cities: Lagos, Abuja, London, Dubai, Accra, and Nairobi.

The business had been running its entire operations on a single shared Excel workbook for four years. That workbook had grown to over 50 columns, contained duplicate guest records, broken formulas, comma-separated multi-value fields, and no reliable way to answer basic business questions. This project replaces it permanently with a properly normalised relational database built in SQL Server.

<img width="4018" height="2911" alt="Grandstay ER black" src="https://github.com/user-attachments/assets/7a185f92-a5fe-43bb-920d-22b14cba6526" />

### The Problem

The source Excel file had the following structural and data quality issues:

- Mixed date formats across booking and payment columns
- Duplicate booking references and duplicate guest records
- Services, service dates, and service costs stored as comma-separated values in three separate columns — impossible to aggregate
- Calculated columns (nights stayed, balance due, total room charge) with errors when source dates conflicted
- Currency not stored anywhere — all monetary values were raw numbers with no unit
- Inconsistent hotel name spelling across rows
- Discount stored as percentage text (`10%`) rather than a usable numeric value
- Only one payment date per booking — partial payment tracking was impossible
- Free-text fields with no standards or validation

### Solution

A fully normalised relational database (3NF) implemented in SQL Server, replacing the flat Excel structure with 8 related tables, proper constraints, indexes, and audit columns throughout.


### Database Schema

#### Entity Relationship Summary

```
RoomType ──< Room >── Hotel
                 \
                  >── Booking ──< ServiceCharge >── Service
                 /
         Guest ──
                  \
                   >── Payment
```

#### Tables

| Table | Description 
|---|---|
| `Hotel` | Master record for each GrandStay property 
| `Room_Type` | Room type definitions with bed type and max occupancy 
| `Room` | Individual rooms per hotel, linked to room type 
| `Guest` | Guest profiles with loyalty tier and contact details 
| `Booking` | Core transaction table linking hotel, room, and guest 
| `Service` | Master catalogue of available hotel services by category 
| `Service_Charge` | Individual service orders per booking 
| `Payment` | Individual payment records per booking 


### Key Design Decisions

**Grain**
One row in the source Excel file represents one booking. All design decisions flow from this. Booking is the central table; all other tables either exist independently as parent entities or as child tables dependent on Booking.

**Surrogate Primary Keys**
Every table uses an `IDENTITY(1,1)` integer as its primary key. Natural candidates such as `booking_ref` and `email` carry real-world instability risks. They are protected by `UNIQUE` constraints for business-level uniqueness but do not serve as PKs.

**Currency Standardisation**
All monetary values are stored in USD. GrandStay operates across six countries with different currencies. A mixed-currency model would make cross-property revenue aggregation meaningless. USD is added as a `currency_code` column on the Hotel table with a default of `'USD'`.

**Comma-Separated Fields Eliminated**
The three source columns `services_used`, `service_dates`, and `service_costs` are replaced entirely by the `Service` and `Service_Charge` tables. Each service instance is now its own row, making aggregation by category straightforward and reliable.

**Calculated Columns Dropped**
`nights_stayed`, `total_room_charge`, `balance_due`, and `service_charges_total` are all dropped from storage. They are derived at query time using `DATEDIFF`, arithmetic expressions, and `SUM` aggregations. Storing derived values introduces inconsistency risk when source values are updated.

**Partial Payment Support**
The source file stored only one payment date per booking. The `Payment` table gives each payment its own row with its own date, amount, method, and status — enabling full installment tracking per booking.

**Soft Delete**
Every table includes an `is_active BIT` column defaulting to `1`. Records are deactivated by setting `is_active = 0` rather than being physically removed. This preserves historical data integrity across all foreign key relationships.

**Audit Columns**
Every table includes `created_at` and `updated_at` `DATETIME` columns with `DEFAULT GETDATE()`. `updated_at` is manually refreshed on every `UPDATE` statement.


### Business Questions Answered
All six validation tests pass. Each query JOINs across multiple tables.



 01  Which hotels generate the most revenue and what is the average revenue per booking? 
 
 <img width="601" height="146" alt="01 Hotel Revenue Summary" src="https://github.com/user-attachments/assets/3a6a1fef-c831-49e7-985f-71677a96b9ea" />

 02  Which room types have the highest occupancy rates across all properties?
 
 <img width="409" height="129" alt="2  Room type occupancy" src="https://github.com/user-attachments/assets/c561249b-d755-4174-8c25-66ed92d08e57" />

 03  Which guest loyalty tier spends the most on average per stay? 
 
 <img width="397" height="102" alt="03  Guest Loyalty Spend" src="https://github.com/user-attachments/assets/be7040cf-2d9c-4178-83ee-02666043523d" />

 04  What is the most popular ancillary service category by total revenue? 
 
 <img width="280" height="164" alt="4  Service Category Revenue" src="https://github.com/user-attachments/assets/8779e3b1-63ba-4888-a956-ce2c44d77ac3" />

 05  Which bookings currently have an outstanding balance due? 
 
 <img width="739" height="260" alt="5 Outstanding Balances" src="https://github.com/user-attachments/assets/f1a98057-873d-489f-b525-3bbfbdcfddae" />

 06  How does booking volume and revenue trend month by month?

 <img width="451" height="194" alt="6 Monthly Booking Trend" src="https://github.com/user-attachments/assets/9ba1902a-e91d-4369-91c5-a4bffa0ae84d" />


### Constraints Applied

| Constraint Type | Example |
|---|---|
| `PRIMARY KEY` | Every table — auto-generated via `IDENTITY(1,1)` |
| `FOREIGN KEY` | `FK_Booking_hotel_id`, `FK_Room_room_type_id`, etc. |
| `UNIQUE` | `booking_ref`, `email`, `(hotel_id, room_number)` composite |
| `CHECK` | `booking_status IN (...)`, `star_rating IN (3,4,5)`, `check_out_date > check_in_date` |
| `DEFAULT` | `is_active = 1`, `currency_code = 'USD'`, `loyalty_tier = 'None'`, `created_at = GETDATE()` |
| `NOT NULL` | All core business columns |

### Indexes

Non-clustered indexes were created on all foreign key columns and columns that appear frequently in `WHERE`, `GROUP BY`, or `JOIN` clauses across the six validation queries.

| Index | Table | Column | Purpose |
|---|---|---|---|
| `IX_Hotel_city` | Hotel | city | Filter by city |
| `IX_Hotel_country` | Hotel | country | Filter by country |
| `IX_Room_hotel_id` | Room | hotel_id | JOIN to Hotel |
| `IX_Room_room_type_id` | Room | room_type_id | JOIN to Room_Type |
| `IX_Guest_email` | Guest | email | Guest lookup |
| `IX_Guest_loyalty_tier` | Guest | loyalty_tier | GROUP BY in Test 03 |
| `IX_Booking_hotel_id` | Booking | hotel_id | JOIN to Hotel |
| `IX_Booking_room_id` | Booking | room_id | JOIN to Room |
| `IX_Booking_guest_id` | Booking | guest_id | JOIN to Guest |
| `IX_Booking_booking_status` | Booking | booking_status | Filter in Test 02 |
| `IX_Booking_booking_date` | Booking | booking_date | GROUP BY in Test 06 |
| `IX_Service_Charge_booking_id` | Service_Charge | booking_id | JOIN to Booking |
| `IX_Service_Charge_service_id` | Service_Charge | service_id | JOIN to Service |
| `IX_Payment_booking_id` | Payment | booking_id | JOIN to Booking |
| `IX_Payment_payment_status` | Payment | payment_status | Filter in Test 05 |


### Data Quality Issues Resolved

| Source Issue | Resolution |
|---|---|
| Mixed date formats | `DATE` / `DATETIME` columns enforce a single format at storage level |
| Duplicate booking references | `UNIQUE` constraint on `booking_ref` |
| Duplicate guest records | `UNIQUE` constraint on `email` |
| Comma-separated services | Replaced by `Service` and `Service_Charge` tables |
| Calculated columns with errors | Dropped; derived at query time |
| Currency not stored | `currency_code CHAR(3)` on `Hotel`, all values standardised to USD |
| Inconsistent hotel name spelling | Single canonical `Hotel` record per property; bookings reference by ID |
| Free-text loyalty tier | `CHECK` constraint on `loyalty_tier`; defaults to `'None'` |
| Discount as percentage text | Dropped; `discount_amount DECIMAL(10,2)` used instead |
| Single payment date per booking | `Payment` table gives each installment its own row |

### Tools Used

- **SSMS (SQL Server Management Studio)** — query execution and validation
- **Excalidraw** — ERD design



