# Product Requirements Document: Chase Credit Card Transaction Import Script

## 1. Introduction
This document outlines the requirements for a Google Apps Script designed to automate the process of importing Chase credit card transactions from CSV files stored in Google Drive into an Airtable database. The script processes files one at a time from a designated folder, focusing on importing only new transactions, preventing duplicates, handling different transaction types (Sales vs Returns), conditionally cleaning up processed CSV files, providing detailed logging, and allowing a configurable limit on new records processed for troubleshooting. It also provides a configurable transaction date range for importing.

It leverages several pre-existing global constants declared in the user's `main.gs` project file and helper functions.

## 2. Goals
* Automate the transfer of Chase credit card transaction data from CSV files to Airtable
* Process CSV files one at a time from a designated folder (oldest first)
* Ensure only new, non-existing transactions are imported into Airtable
* Properly handle different transaction types (Sales, Returns, etc.)
* Use a payee mapping file to intelligently categorize transactions
* Set specific default values:
  * `CC Card` = "Chase Sapphire"
  * `Flag` = True
* Flag newly imported records in Airtable for easy identification
* Maintain a clean Google Drive by conditionally deleting processed CSV files (defaulting to retention)
* Provide a mechanism to limit the number of new records processed per run (defaulting to 2) for testing
* Provide an option to evaluate and import only records within a certain transaction date range
* Provide detailed logging for script execution status and troubleshooting

## 3. Target User
The primary user is an individual who manages their credit card expenses in Airtable, receives transaction history as CSV files from Chase, needs intelligent categorization of transactions based on merchant descriptions, requires proper handling of returns vs sales, and requires robust error handling and logging for troubleshooting.

## 4. Functional Requirements

### 4.1 CSV Processing
* Look for CSV files in the Google Drive folder specified by `CC_FOLDER_NAME`
* Process only CSV files that start with the prefix defined in `CHASE2270_FILE_PREFIX` ("Chase2270")
* Select the oldest file (by creation date) from matching files
* Process only one file per script execution
* Parse CSV with the following field mappings:
  * Transaction Date → paymentDate (with robust date parsing)
  * Amount → paymentAmount (with sign reversal logic based on Type)
  * Description → merchantName (modified based on Type)
  * Type → Used to determine transaction handling
* Skip malformed CSV rows and log them
* Process only transactions within the configured date range

### 4.2 Transaction Type Handling
* **Sales** (Type = "Sale"):
  * Use the description as-is for merchantName
  * Convert negative amounts to positive (charges)
* **Returns** (Type = "Return"):
  * Append " (return)" to the description for merchantName
  * Convert positive amounts to negative (credits)
* **Other types** (Payment, Adjustment, etc.):
  * Append the type in parentheses to the description
  * Process amount with standard sign reversal

### 4.3 Payee Mapping
* Read payee mapping rules from a CSV file (`CHASE_CSV_PAYEEMAPPER_FILENAME`) in the Receipts folder
* Mapping file contains: TransactDesc, ExpPayee, ExpType, Location, BusinessExpense, BusType
* Support case-insensitive exact matching
* Use the original description (before type modification) for mapping lookup
* If no mapping match is found, use the transaction description and set mapped fields to null

### 4.4 Duplicate Detection
* Check for existing transactions in Airtable using:
  * Transaction Date within ±3 days of the new transaction
  * Amount matches exactly
* Skip duplicate transactions and log them

### 4.5 Record Creation
* Create new Airtable records for non-duplicate transactions
* Set fixed values:
  * CC Card = "Chase Sapphire"
  * Flag = True
* Use mapped values from payee mapping file for: ExpPayee, ExpType, Location, BusinessExpense, BusType
* Set OrderID and OrderDate to null

### 4.6 Error Handling
* **No matching CSV files found**: Log message and exit gracefully
* **Malformed CSV rows**: Skip and log, continue processing
* **Failed Airtable API calls**: Log error and stop processing
* **Missing payee mapping file**: Log error and stop processing
* **Date outside configured range**: Log and skip transaction
* **Invalid date range configuration**: Log error and stop processing
* **Undefined constants from main.gs**: Log error and stop processing

### 4.7 Processing Limits
* Respect `MAX_NEW_RECORDS_TO_PROCESS` limit (default: 2, defined in main.gs)
* A value of -1 or 0 indicates no limit

### 4.8 File Management
* Conditionally delete the CSV file after successful processing based on `DELETE_CSV_AFTER_PROCESSING` (default: false, defined in main.gs)
* File deletion occurs only if there were no errors during processing

## 5. Technical Requirements

### 5.1 CSV File Format
* **Chase CSV Fields**: Transaction Date, Post Date, Description, Category, Type, Amount, Memo
* **Required fields**: Transaction Date, Amount, Description, Type
* **Payee Mapping CSV Fields**: TransactDesc, ExpPayee, ExpType, Location, BusinessExpense, BusType
* Amount field: 
  * Sales: Negative values (convert to positive for Airtable)
  * Returns: Positive values (convert to negative for Airtable)
* Date parsing must handle multiple formats (MM/DD/YYYY, MM/DD/YY, YYYY-MM-DD, etc.)

### 5.2 Date Range Configuration
* `CHASE_CSV_STARTDATE`: Start date for transaction import (required, defined in script)
* `CHASE_CSV_ENDDATE`: End date for transaction import (required, defined in script)
* Both dates must be set or script stops with error

### 5.3 File Selection Logic
* Only process files in the root of `CC_FOLDER_NAME` (no subfolders)
* Files must have .csv extension (case-insensitive check)
* Files must start with `CHASE2270_FILE_PREFIX`
* Select oldest file by creation date
* Process one file per execution

## 6. Technical Details & Assumptions

### 6.1 Global Constants
The following are assumed **pre-declared in `main.gs`:**

**Airtable Configuration:**
* `AIRTABLE_BASE_ID`
* `AIRTABLE_EXPDB_TABLE_ID`
* `AIRTABLE_API_KEY`

**Airtable Field Name Mappings:**
* `AIRTABLE_EXP_FIELD_NAME` (for Description/merchantName)
* `AIRTABLE_EXP_FIELD_COST` (for Amount/paymentAmount)
* `AIRTABLE_EXP_FIELD_DATE` (for Transaction Date/paymentDate)
* `AIRTABLE_EXP_FIELD_ORDERDATE` (for OrderDate)
* `AIRTABLE_EXP_FIELD_ORDERID` (for OrderID)
* `AIRTABLE_EXP_FIELD_FLAG` (for flag)
* `AIRTABLE_EXP_FIELD_EXP_PAYEE` (for Exp Payee)
* `AIRTABLE_EXP_FIELD_CC_CARD` (for CC Card)
* `AIRTABLE_EXP_FIELD_EXP_TYPE` (for Exp Type)
* `AIRTABLE_EXP_FIELD_LOCATION` (for Location)
* `AIRTABLE_EXP_FIELD_BUSINESS_EXPENSE` (for Business Expense)
* `AIRTABLE_EXP_FIELD_BUS_TYPE` (for Bus Type)

**File/CSV Configuration:**
* `CC_FOLDER_NAME` (folder containing credit card CSV files)
* `CHASE2270_FILE_PREFIX` (prefix for Chase account 2270 files: "Chase2270")
* `CHASE_CSV_PAYEEMAPPER_FILENAME`
* `RECEIPTS_FOLDER_NAME`
* `CHASE_CSV_TRANSACTION_DATE`
* `CHASE_CSV_POST_DATE`
* `CHASE_CSV_DESCRIPTION`
* `CHASE_CSV_CATEGORY_TYPE`
* `CHASE_CSV_AMOUNT`
* `CHASE_CSV_MEMO`

**Script Behavior Constants (defined in main.gs):**
* `DELETE_CSV_AFTER_PROCESSING`: Boolean, default `false`
* `MAX_NEW_RECORDS_TO_PROCESS`: Integer, default `2`

**Script-Specific Constants (defined in this script):**
* `CHASE_CSV_STARTDATE`: String, required (e.g., "2025-06-01")
* `CHASE_CSV_ENDDATE`: String, required (e.g., "2025-06-09")

### 6.2 Helper Functions
**`createHAirtableRecord`:**
* **Signature**: `function createHAirtableRecord(paymentAmount, paymentDate, merchantName, DBExpType, DBExpPayee, ccard, flag, location, businessExpense, BusType, OrderID, OrderDate)`
* **Return Value**: `newRecordId` (string) or `null`
* **Argument Mapping**:
  * paymentAmount: Processed amount (sign based on transaction type)
  * paymentDate: Parsed and formatted transaction date
  * merchantName: Transaction description (modified based on type)
  * DBExpType: From mapping file or null
  * DBExpPayee: From mapping file or null
  * ccard: "Chase Sapphire"
  * flag: true
  * location: From mapping file or null
  * businessExpense: From mapping file or false
  * BusType: From mapping file or null
  * OrderID: null
  * OrderDate: null

### 6.3 Function Structure
The script is organized into the following functions:

1. **`validateRequiredConstants()`** - Validates that required constants from main.gs are defined
2. **`parseRobustDate(dateString)`** - Robust date parsing function supporting multiple formats
3. **`isWithinDateRange(date, startDate, endDate)`** - Check if date is within range
4. **`reverseAmountSign(amount)`** - Convert charge/credit amounts (always reverses sign)
5. **`logError(message, details)`** - Helper for consistent error logging
6. **`processChaseTransactions()`** - Main orchestration function
7. **`loadPayeeMapping()`** - Load and parse the payee mapping CSV
8. **`findOldestChaseCSVFile()`** - Find the oldest Chase CSV file in the CC folder matching the prefix
9. **`parseChaseCSV(csvContent)`** - Parse the Chase CSV content, returns object with headers, headerIndexMap, and data
10. **`findPayeeMapping(description, mappingData)`** - Find matching payee mapping rule (uses original description)
11. **`checkDuplicateTransaction(amount, date)`** - Check for existing transaction in Airtable
12. **`processTransaction(transaction, mappingData, headerIndexMap)`** - Process a single transaction with type handling

### 6.4 Logging Requirements
Detailed logging includes:
* Script start/end timestamps
* Configuration values (date range, limits, etc.)
* File locations and status
* Selected file name and creation date
* For each transaction:
  * Original CSV data including Type
  * Parsed values
  * Transaction type handling (Sale/Return/Other)
  * Payee mapping results
  * Duplicate check results
  * Airtable creation status
* Summary statistics:
  * Total transactions in CSV
  * Transactions within date range
  * Breakdown by type (Sales, Returns, Other)
  * New transactions created
  * Duplicates skipped
  * Errors encountered
* All errors with context

### 6.5 Permissions
* Google Drive: Read access to CC_FOLDER_NAME and RECEIPTS_FOLDER_NAME folders, optional delete for CSV file
* External HTTP: Airtable API access via helper function

## 7. Success Criteria
* Successfully imports new Chase transactions into Airtable
* Correctly identifies and skips duplicate transactions
* Properly handles different transaction types (Sales show as positive charges, Returns show as negative credits)
* Properly categorizes transactions using payee mapping
* Processes files one at a time in chronological order
* Handles all error cases gracefully with appropriate logging
* Respects configured limits and date ranges
* Provides detailed logs for troubleshooting

## 8. Implementation Notes
* The script processes CSV files from the `CC_FOLDER_NAME` directory
* Only files starting with `CHASE2270_FILE_PREFIX` are processed
* Files are processed one at a time, oldest first (by creation date)
* The script uses a headerIndexMap to properly access CSV columns by name
* Transaction type handling occurs before payee mapping to ensure mapping uses original descriptions
* The Type field is a required column in the Chase CSV
* Amount sign reversal logic works in conjunction with transaction type to ensure proper positive/negative values
* All helper functions that are called within the main function must be defined before the main function to avoid hoisting issues

## 9. Version History
* **Version 1.0**: Initial implementation with single file processing from RECEIPTS_FOLDER_NAME
* **Version 1.1**: Updated to process files from CC_FOLDER_NAME with prefix-based filtering and oldest-first selection

## 10. Future Enhancements
* Email notification upon completion
* Support for multiple credit card types (different prefixes)
* Automatic detection of CSV format
* Bulk duplicate checking for better performance
* Configuration UI for non-technical users
* Support for transaction categories from Chase
* Configurable transaction type handling rules
* Batch processing of multiple files in a single run
* Archive folder for processed files instead of deletion
