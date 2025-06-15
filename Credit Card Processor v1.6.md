# Product Requirements Document: Chase and Amex Credit Card Transaction Import Script

**Version:** 1.6  
**Last Updated:** June 2025  
**Author:** System Administrator

## 1. Introduction
This document outlines the requirements for a Google Apps Script designed to automate the process of importing Chase and American Express credit card transactions from CSV files stored in Google Drive into an Airtable database. The script supports multiple credit card accounts by processing files with different prefixes. It processes files one at a time from a designated folder, focusing on importing only new transactions, preventing duplicates, handling different transaction types (Sales vs Returns), conditionally cleaning up processed CSV files, providing detailed logging, and allowing a configurable limit on new records processed for troubleshooting. It also provides a configurable transaction date range for importing.

It leverages several pre-existing global constants declared in the user's `main.gs` project file and helper functions.

## 2. Goals
* Automate the transfer of Chase and Amex credit card transaction data from CSV files to Airtable
* Process CSV files one at a time from a designated folder (oldest first)
* Ensure only new, non-existing transactions are imported into Airtable
* Properly handle different transaction types (Sales, Returns, etc.)
* Use a payee mapping file to intelligently categorize transactions
* Set specific default values based on card type:
  * If card is Chase2270: `CC Card` = "Chase Sapphire"
  * If card is Chase5991: `CC Card` = "Chase Business"
  * If card is Amex: `CC Card` = "Amex"
  * All cards: `Flag` = True
* Flag newly imported records in Airtable for easy identification
* Maintain a clean Google Drive by conditionally deleting processed CSV files (defaulting to retention)
* Provide a mechanism to limit the number of new records processed per run (defaulting to 2) for testing
* Provide an option to evaluate and import only records within a certain transaction date range
* Provide detailed logging for script execution status and troubleshooting

## 3. Target User
The primary user is an individual who manages their credit card expenses in Airtable, receives transaction history as CSV files from Chase and American Express, needs intelligent categorization of transactions based on merchant descriptions, requires proper handling of returns vs sales, and requires robust error handling and logging for troubleshooting.

## 4. Functional Requirements

### 4.1 CSV Processing
* Look for CSV files in the Google Drive folder specified by `CC_FOLDER_NAME`
* Process CSV files that start with any of the configured prefixes:
  * `CHASE2270_FILE_PREFIX` ("Chase2270") 
  * `CHASE5991_FILE_PREFIX` ("Chase5991")
  * `AMEX_FILE_PREFIX` ("activity")
* Select the oldest file (by creation date) from all matching files across all prefixes
* Process only one file per script execution
* Parse CSV with the following field mappings:
  * **Chase CSVs:**
    * Transaction Date → paymentDate (with robust date parsing)
    * Amount → paymentAmount (with sign reversal logic based on Type)
    * Description → merchantName (modified based on Type)
    * Type → Used to determine transaction handling
    * Card → Present in Chase5991 files but not currently used
  * **Amex CSVs:**
    * Date → paymentDate (with robust date parsing)
    * Amount → paymentAmount (no sign reversal needed)
    * Appears On Your Statement As → merchantName (first line only)
    * Category → Used for expense type mapping
* Skip malformed CSV rows and log them
* Process only transactions within the configured date range

### 4.2 Transaction Filtering
* **Payment Transactions**: Skip specific payment transactions that should not be imported:
  * Amex: Skip transactions where merchant name (from Extended Details) equals "AUTOPAY PAYMENT - THANK YOU"
  * Chase2270: Skip transactions where Description equals "Payment Thank You - Web"
  * Chase5991: Skip transactions where Description equals "AUTOMATIC PAYMENT - THANK"
* These transactions are logged as skipped but not processed or imported to Airtable

### 4.3 Transaction Type Handling
* **Chase Cards:**
  * **Sales** (Type = "Sale"):
    * Use the description as-is for merchantName
    * Convert negative amounts to positive (charges)
  * **Returns** (Type = "Return"):
    * Append " (return)" to the description for merchantName
    * Convert positive amounts to negative (credits)
  * **Other types** (Payment, Adjustment, etc.):
    * Append the type in parentheses to the description
    * Process amount with standard sign reversal
* **Amex Cards:**
  * **Sales** (positive amounts):
    * Use first line of "Appears On Your Statement As" for merchantName
    * Keep amount as positive
  * **Returns** (negative amounts):
    * Use first line of "Appears On Your Statement As" for merchantName
    * Append " (return)" to merchantName
    * Keep amount as negative

### 4.4 Payee Mapping
* Read payee mapping rules from a CSV file (`CHASE_CSV_PAYEEMAPPER_FILENAME`) in the CC_FOLDER_NAME folder
* Mapping file contains: TransactDesc, ExpPayee, ExpType, Location, BusinessExpense, BusType
* Support case-insensitive matching with two modes:
  * **Exact match**: Full description matches TransactDesc exactly
  * **Prefix match**: Description starts with TransactDesc (e.g., "WWW.KOHLS.COM" matches "WWW.KOHLS.COM #0873")
* Matching priority: Exact matches are checked first, then prefix matches
* Use the original description (before type modification) for mapping lookup
* If no mapping match is found:
  * Use Category field to determine ExpType (see Category Mapping below)
  * Set other mapped fields (ExpPayee, Location, BusinessExpense, BusType) to null

### 4.5 Category Mapping (Fallback)
* When no payee mapping rule matches, use the Category field to set ExpType:
  * **Chase Categories:**
    * "Groceries" → ExpType = recON2gR100Ua4wbh
    * "Food & Drink" → ExpType = recSsjrsZqPFTo2mc
    * "Gas" → ExpType = recPToP7D1qsWoWnR
    * "Health & Wellness" → ExpType = recyZDc5GznqyQYPf
  * **Amex Categories:**
    * "Restaurant-Restaurant" → ExpType = recSsjrsZqPFTo2mc
    * "Transportation-Tolls & Fees" → ExpType = recPToP7D1qsWoWnR
    * "Transportation-Fuel" → ExpType = recPToP7D1qsWoWnR
* If Category doesn't match any of the above, ExpType remains null
* All other fields (ExpPayee, Location, BusinessExpense, BusType) remain null when using category mapping

### 4.6 Duplicate Detection
* Check for existing transactions in Airtable using:
  * Transaction Date within ±3 days of the new transaction
  * Amount matches exactly
* Skip duplicate transactions and log them

### 4.7 Record Creation
* Create new Airtable records for non-duplicate transactions
* Set fixed values based on card type:
  * For Chase2270 files: CC Card = "Chase Sapphire"
  * For Chase5991 files: CC Card = "Chase Business"
  * For Amex files: CC Card = "Amex"
  * Flag = True (for all card types)
* Use mapped values from payee mapping file for: ExpPayee, ExpType, Location, BusinessExpense, BusType
* Set OrderID and OrderDate to null

### 4.8 Error Handling
* **No matching CSV files found**: Log message and exit gracefully
* **Malformed CSV rows**: Skip and log, continue processing
* **Failed Airtable API calls**: Log error and stop processing
* **Missing payee mapping file**: Log error and stop processing
* **Date outside configured range**: Log and skip transaction
* **Invalid date range configuration**: Log error and stop processing
* **Undefined constants from main.gs**: Log error and stop processing

### 4.9 Processing Limits
* Respect `MAX_NEW_RECORDS_TO_PROCESS` limit (default: 2, defined in main.gs)
* A value of -1 or 0 indicates no limit

### 4.10 File Management
* Conditionally delete the CSV file after successful processing based on `DELETE_CSV_AFTER_PROCESSING` (default: false, defined in main.gs)
* File deletion occurs only if there were no errors during processing

## 5. Technical Requirements

### 5.1 CSV File Format
* **Chase CSV Fields (both card types)**: Transaction Date, Post Date, Description, Category, Type, Amount, Memo
* **Additional Chase5991 Field**: Card (integer indicating card number, not currently used)
* **Amex CSV Fields**: Date, Description, Card Member, Account #, Amount, Extended Details, Appears On Your Statement As, Address, City/State, Zip Code, Country, Reference, Category
* **Required fields**: 
  * Chase: Transaction Date, Amount, Description, Type, Category (for fallback mapping)
  * Amex: Date, Amount, Appears On Your Statement As, Category (for fallback mapping)
* Amount field: 
  * **Chase:**
    * Sales: Negative values (convert to positive for Airtable)
    * Returns: Positive values (convert to negative for Airtable)
  * **Amex:**
    * Sales: Positive values (keep positive for Airtable)
    * Returns: Negative values (keep negative for Airtable)
* Date parsing must handle multiple formats:
  * Chase: MM/DD/YYYY, MM/DD/YY, YYYY-MM-DD, etc.
  * Amex: MM/DD/YY format
* Payee mapping examples:
  * TransactDesc "WWW.KOHLS.COM" will match descriptions like "WWW.KOHLS.COM #0873"
  * TransactDesc "CVS/PHARMACY" will match descriptions like "CVS/PHARMACY #00531"
  * TransactDesc "INYO POOLS PRODUCTS" will match "INYO POOLS PRODUCTS" from Amex
* Category mapping (fallback when no payee match):
  * Chase: "Groceries" → Grocery expense type, etc.
  * Amex: "Restaurant-Restaurant" → Food expense type, etc.

### 5.2 Date Range Configuration
* `CC_CSV_STARTDATE`: Start date for transaction import (required, defined in main.gs)
* `CC_CSV_ENDDATE`: End date for transaction import (required, defined in main.gs)
* Both dates must be set or script stops with error

### 5.3 File Selection Logic
* Only process files in the root of `CC_FOLDER_NAME` (no subfolders)
* Files must have .csv extension (case-insensitive check)
* Files must start with one of the configured prefixes:
  * `CHASE2270_FILE_PREFIX`
  * `CHASE5991_FILE_PREFIX`
  * `AMEX_FILE_PREFIX`
* Select oldest file by creation date across all matching files
* Process one file per execution
* The specific card type is determined by the file prefix

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
* `CHASE5991_FILE_PREFIX` (prefix for Chase account 5991 files: "Chase5991")
* `AMEX_FILE_PREFIX` (prefix for Amex files: "activity")
* `CHASE_CSV_PAYEEMAPPER_FILENAME`
* `RECEIPTS_FOLDER_NAME`
* Chase CSV field constants (CHASE_CSV_TRANSACTION_DATE, etc.)
* Amex CSV field constants (AMEX_CSV_DATE, AMEX_CSV_APPEARSONCC, etc.)

**Script Behavior Constants (defined in main.gs):**
* `DELETE_CSV_AFTER_PROCESSING`: Boolean, default `false`
* `MAX_NEW_RECORDS_TO_PROCESS`: Integer, default `2`
* `CC_CSV_STARTDATE`: String, required (e.g., "2025-06-01")
* `CC_CSV_ENDDATE`: String, required (e.g., "2025-06-09")

### 6.2 Helper Functions
**`createHAirtableRecord`:**
* **Signature**: `function createHAirtableRecord(paymentAmount, paymentDate, merchantName, DBExpType, DBExpPayee, ccard, flag, location, businessExpense, BusType, OrderID, OrderDate)`
* **Return Value**: `newRecordId` (string) or `null`
* **Argument Mapping**:
  * paymentAmount: Processed amount (sign based on card type and transaction type)
  * paymentDate: Parsed and formatted transaction date
  * merchantName: Transaction description (modified based on type)
  * DBExpType: From mapping file or null
  * DBExpPayee: From mapping file or null
  * ccard: "Chase Sapphire" for Chase2270, "Chase Business" for Chase5991, "Amex" for Amex
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
4. **`reverseAmountSign(amount)`** - Convert charge/credit amounts (only used for Chase)
5. **`logError(message, details)`** - Helper for consistent error logging
6. **`processChaseAndAmexTransactions()`** - Main orchestration function (renamed from processChaseTransactions)
7. **`loadPayeeMapping()`** - Load and parse the payee mapping CSV
8. **`findOldestCSVFile()`** - Find the oldest CSV file in the CC folder matching any prefix (renamed from findOldestChaseCSVFile)
9. **`parseChaseCSV(csvContent)`** - Parse the Chase CSV content, returns object with headers, headerIndexMap, and data
10. **`parseAmexCSV(csvContent)`** - Parse the Amex CSV content, returns object with headers, headerIndexMap, and data
11. **`findPayeeMapping(description, mappingData)`** - Find matching payee mapping rule using exact or prefix matching
12. **`checkDuplicateTransaction(amount, date)`** - Check for existing transaction in Airtable
13. **`processTransaction(transaction, mappingData, headerIndexMap, cardType)`** - Process a single transaction with type and card handling
14. **`getCategoryExpenseType(category, cardType)`** - Map category to Airtable expense type (updated to handle both Chase and Amex)
15. **`extractFirstLineFromAmexDescription(description)`** - Extract first line from Amex "Appears On Your Statement As" field

### 6.4 Logging Requirements
Detailed logging includes:
* Script start/end timestamps
* Configuration values (date range, limits, etc.)
* File locations and status
* Selected file name and creation date
* For each transaction:
  * Original CSV data including Type/Amount and Category
  * Parsed values
  * Transaction type handling (Sale/Return/Other)
  * Payee mapping results or category-based fallback
  * Duplicate check results
  * Airtable creation status
* Summary statistics:
  * Total transactions in CSV
  * Transactions within date range
  * Breakdown by type (Sales, Returns, Other)
  * New transactions created
  * Duplicates skipped
  * Payment transactions skipped
  * Errors encountered
* All errors with context

### 6.5 Permissions
* Google Drive: Read access to CC_FOLDER_NAME folder, optional delete for CSV files
* External HTTP: Airtable API access via helper function

## 7. Success Criteria
* Successfully imports new Chase and Amex transactions into Airtable
* Correctly identifies and skips duplicate transactions
* Properly skips payment transactions based on card-specific patterns
* Properly handles different transaction types:
  * Chase: Sales show as positive charges, Returns show as negative credits
  * Amex: Sales remain positive, Returns remain negative
* Properly categorizes transactions using payee mapping
* Processes files one at a time in chronological order
* Handles all error cases gracefully with appropriate logging
* Respects configured limits and date ranges
* Provides detailed logs for troubleshooting

## 8. Implementation Notes
* The script processes CSV files from the `CC_FOLDER_NAME` directory
* The payee mapping file is also located in the `CC_FOLDER_NAME` directory
* Files starting with `CHASE2270_FILE_PREFIX`, `CHASE5991_FILE_PREFIX`, or `AMEX_FILE_PREFIX` are processed
* Files are processed one at a time, oldest first (by creation date) across all supported prefixes
* The script uses a headerIndexMap to properly access CSV columns by name
* Transaction type handling differs between Chase (uses Type field) and Amex (uses amount sign)
* Payee mapping supports both exact and prefix matching (case-insensitive)
* Category-based expense type mapping is used as a fallback when no payee mapping matches
* The Type field is required in Chase CSVs but not present in Amex CSVs
* The Category field is used for fallback expense type mapping in both Chase and Amex
* Amex merchant names are extracted from the first line of "Appears On Your Statement As" field
* All helper functions that are called within the main function must be defined before the main function to avoid hoisting issues

## 9. Version History
* **Version 1.0**: Initial implementation with single file processing from RECEIPTS_FOLDER_NAME
* **Version 1.1**: Updated to process files from CC_FOLDER_NAME with prefix-based filtering and oldest-first selection
* **Version 1.2**: Added support for multiple card types (Chase2270 and Chase5991) with shared processing logic
* **Version 1.3**: Moved payee mapping file from RECEIPTS_FOLDER_NAME to CC_FOLDER_NAME for centralized file management
* **Version 1.4**: Added prefix matching support for payee mapping to handle transaction descriptions with suffixes
* **Version 1.5**: Added category-based expense type mapping as fallback when no payee mapping matches
* **Version 1.6**: Added support for American Express credit card processing with different CSV format and transaction handling, added automatic skipping of payment transactions

## 10. Future Enhancements
* Email notification upon completion
* Support for additional credit card types (different prefixes)
* Automatic detection of CSV format
* Bulk duplicate checking for better performance
* Configuration UI for non-technical users
* Configurable transaction type handling rules
* Batch processing of multiple files in a single run
* Archive folder for processed files instead of deletion
* Enhanced category mapping for American Express to include Card Member name mapping:
  * Use Card Member field to set ExpType based on cardholder
  * Example: "Natalie O'Connell" → ExpType = "Discretionary - Natalie"
  * Example: "Adrian C Panek" → ExpType = "Discretionary - Adrian"
  * This would provide automatic expense categorization based on who made the purchase
* Create global variables in main.gs for commonly used Airtable record IDs:
  * Define ExpType record IDs as constants (e.g., `EXPTYPE_DISCRETIONARY_NATALIE`, `EXPTYPE_DISCRETIONARY_ADRIAN`)
  * Define ExpPayee record IDs as constants for frequent merchants
  * This would improve code maintainability and reduce hardcoded record IDs throughout the project