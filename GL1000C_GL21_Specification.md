# GL1000C/GL21 - Transaction Entry Application
## Comprehensive Current State Analysis Document

---

# PART I: FUNCTIONAL SPECIFICATION

---

## 1. Executive Summary

### 1.1 System Purpose

GL21 (GL1000C) is the **General Ledger Transaction Entry** application, serving as the primary interface for entering, editing, voiding, and posting financial transactions within the accounting system. It functions as a **shared service** called by multiple upstream modules including Accounts Receivable, Accounts Payable, Payroll, and Service Dispatch.

### 1.2 Key Business Outcomes

- **Transaction Creation**: Entry of debit/credit journal entries with full audit trail
- **Document Management**: Tracking of invoices, checks, deposits, and other financial documents
- **Period Control**: Enforcement of fiscal period boundaries for transaction posting
- **Inter-Company Accounting**: Support for multi-company transactions with proper security controls
- **Void/Reversal Processing**: Capability to void transactions with automatic reversal journal entries
- **Balance Validation**: Ensures all transactions balance before posting

### 1.3 Module Hierarchy Position

Based on the knowledge analysis:

```
┌─────────────────────────────────────────────────────────┐
│                    UPSTREAM CALLERS                      │
├─────────────────────────────────────────────────────────┤
│  GL2010R (A/P Open Items)    │  GL2100R (A/R Open Items) │
│  GL2300R (Journal Entry)     │  GL2440R (Check Writing)  │
│  SE8000R (Service Dispatch)  │  SE8001R (Service Post)   │
│  GL2020R (Vendor History)    │  GL2150R (A/R History)    │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              GL1000R - TRANSACTION ENTRY                 │
│                  (Primary Entry Point)                   │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                 DOWNSTREAM PROGRAMS                      │
├─────────────────────────────────────────────────────────┤
│  GL1001R (Void Document)     │  GL8100R (Posting Engine) │
│  GL1100R (Posted Invoices)   │  GL1020R (Account Search) │
│  GL1006R (Document Inquiry)  │  GL1110R (Control Desc)   │
│  GL1140R (Unposted Docs)     │  GL8510R (Validation)     │
└─────────────────────────────────────────────────────────┘
```

### 1.4 Entry Call Modes

| Mode | Entry Point | Configuration Behavior |
|------|-------------|------------------------|
| **Menu Entry** | Direct menu selection | Full feature access based on user authorization |
| **Cross-Module Call** | Called from GL2010R, GL2100R, GL2300R, etc. | Context-sensitive with document type pre-set |
| **Inquiry Mode** | $ECALL = 'I' parameter | Read-only access, no modification allowed |
| **Void Mode** | $ECALL = 'V' parameter | Enables void document processing |
| **Batch/Import** | SE8000R/SE8001R integration | Automated posting without user interaction |

### 1.5 Security Prerequisites

**Minimum Authorization Requirements:**
- Valid user profile in SEPUAUTH security database
- Application access code 'GL' with function '2' authorization
- Company-level access to the selected company number

### 1.6 User Personas

| User Type | Entry Point | Authorization Required | Available Features |
|-----------|-------------|------------------------|-------------------|
| **Standard User** | Menu/Module | Basic GL access | View, Enter transactions |
| **Supervisor** | Menu/Module | Administrative flag | Void, Override closed periods |
| **Inter-Company User** | Menu/Module | IC authorization | Cross-company transactions |
| **Batch Processor** | SE8000R/SE8001R | System-level | Automated posting |
| **Inquiry User** | Cross-module | Read-only | View only, no modifications |

---

## 2. User Type Process Flows

### 2.1 Standard Transaction Entry Flow

**Entry Point:** Menu selection or cross-module call with $CFUNC = '0'

**Authorization Required:**
- SE0101R authorization check with $APP = 'GL', $AFUNC = '2'
- Company-level access validation

**Available Features:**
- Enter new transactions
- Edit unposted transactions
- View transaction details
- Account number search (F4)
- Control number search

**Process Steps:**

1. **System Initialization**
   - User selects GL21 from menu or is called from upstream module
   - SE0001R retrieves company header information ($HNAM60)
   - SE0101R validates user authorization for GL module

2. **Document Selection/Creation**
   - User selects document type (Invoice, Check, Deposit, Journal Entry)
   - System validates document number format
   - For new documents: generates transaction number from GLPAPEN (GATRN#)
   - For existing documents: retrieves from GLPTRNS

3. **Transaction Line Entry**
   - User enters account number (validated via GL1020R)
   - User enters control number (validated via GL1110R for customer/vendor)
   - User enters debit or credit amount
   - System calculates running balance

4. **Balance Validation**
   - System verifies debits = credits
   - GL8510R performs comprehensive validation
   - Inter-company validation via GL0072R if applicable

5. **Posting/Saving**
   - User selects Post (F10) or Save
   - GL8100R posting engine processes transaction
   - GLPTRNS records updated with GTPOST = 'Y'

**Business Outcomes:**
- Transaction recorded in GLPTRNS
- Account balances updated in GLPMAST
- Document control record in GLPACTL
- Audit trail created in GLPDTIM

### 2.2 Void Document Flow

**Entry Point:** GL1001R called from GL1000R with void request

**Authorization Required:**
- Administrative authorization via SE0261R ($AUTHADMIN = 'Y')
- Period open validation ($VOIDDOC flag)

**Available Features:**
- Void posted transactions
- Automatic reversal journal entry creation
- Reconciliation status clearing

**Process Steps:**

1. **Void Initiation**
   - User selects void option on posted document
   - System validates administrative authorization
   - GL1001R receives document parameters

2. **Period Validation**
   - System checks if transaction month is open via GL0070R
   - If closed: routes to @REVERSE subroutine for reversal
   - If open: routes to @VOID subroutine for direct void

3. **Void Processing (@VOID)**
   - Retrieves all transaction lines from GLLTRNAA
   - Sets GTPOST = 'V' (voided status)
   - Clears reconciliation status (GTRSTS)
   - Updates document with voided indicator

4. **Reversal Processing (@REVERSE)**
   - Creates new reversing transactions
   - Generates new transaction number
   - Signs amounts reversed (Z-SUB GTTAMT)
   - Posts reversal via GL8100R

5. **Completion**
   - GL1130R logs void action
   - clearReconcile procedure clears related reconciliation records
   - Control records updated

**Business Outcomes:**
- Original transaction marked as voided (GTPOST = 'V')
- Reversal transaction created if month closed
- Reconciliation records cleared
- Audit trail updated

### 2.3 Inter-Company Transaction Flow

**Entry Point:** Standard entry with inter-company account detected

**Authorization Required:**
- Inter-company user authorization via GL0202R
- GLPICUSER table entry for user/account combination

**Available Features:**
- Cross-company debit/credit entry
- Automatic offsetting entries
- Inter-company code validation

**Process Steps:**

1. **Account Detection**
   - User enters account number
   - GL0555R retrieves inter-company configuration
   - $USEICA preference checked ('1' or '2' for IC enabled)

2. **Security Validation**
   - GL0202R validates user authorization for IC account
   - Checks GLPICUSER for GICSUSER = current user
   - Returns $ICSERR = '1' if unauthorized

3. **Inter-Company Code Entry**
   - User enters IC location code
   - GL0072R validates code against GLPIXCO
   - Returns $ICSERR4 if invalid

4. **Posting with IC Processing**
   - GL8100R handles inter-company posting
   - Creates offsetting entries in related company
   - Validates cross-company balancing

**Business Outcomes:**
- Transactions recorded in both companies
- Inter-company clearing accounts updated
- Full audit trail across companies

### 2.4 Inquiry Mode Flow

**Entry Point:** Cross-module call with $ECALL = 'I'

**Authorization Required:**
- Basic read access only

**Available Features:**
- View transaction details
- Drill-down to source documents
- No modification capability

**Process Steps:**

1. **Document Display**
   - System retrieves document from GLPTRNS
   - Displays in read-only format
   - Function keys limited to navigation only

2. **Source Document Access**
   - F6 displays source invoice/check details
   - Calls GL1100R for posted invoice display
   - Returns to calling program on exit

**Business Outcomes:**
- Information access without modification risk
- Audit trail not impacted

---

## 3. Business Rules & Validations

### 3.1 Field-Level Validations

| Field | Validation Rule | Source Program |
|-------|-----------------|----------------|
| **Company Number ($CO#)** | Must exist in SEPCOMP | SE0001R |
| **Account Number (GTACCT)** | Must exist in GLPMAST, must be active (GMACTIVE <> 'N') | GL1020R, GL1021R |
| **Control Number (GTCTL#)** | Must match control type for account | GL1110R, GL1080R |
| **Document Number (GTDOC#)** | Max 17 characters, unique per company/type | GLPTRNS |
| **Transaction Date (GTDATE)** | Must be within open fiscal period | GL0070R |
| **Amount (GTTAMT)** | 14,2 packed decimal, sign determines DR/CR | GL8510R |

### 3.2 Cross-Field Validations

| Validation | Rule Logic | Enforcement Point |
|------------|-----------|-------------------|
| **Account-Control Type Match** | Account's GMCTYP must match control record type | GL1110R |
| **Document-Journal Match** | Document type (GTDTYP) must correspond to journal (GTJRNL) | GL8100R |
| **Period-Date Match** | Transaction date must fall within selected fiscal period | GL0070R |
| **IC Account-User Match** | User must be authorized for inter-company account | GL0202R |

### 3.3 Balancing Logic

**Pre-Post Balance Validation (GL8510R):**

```
For each transaction number:
  SUM(Debit amounts) = SUM(Credit amounts)
  
  If $total <> 0:
    Set error_indicator = *ON
    Return message_id = validation error
```

**Split Account Processing:**
- Split transactions detected when multiple detail lines exist
- Each split must individually balance
- GL8100R calls gl8170r for split resolution

### 3.4 Period Control Rules

| Rule | Description | Source |
|------|-------------|--------|
| **Open Month Check** | GL0070R validates date against GLPOPYR | GL0070R |
| **Fiscal Year Boundary** | GACAYR (current year) and GAFYSM (fiscal start month) define boundaries | GLPAPEN |
| **Prior Period Override** | Requires $AUTHADMIN authorization | SE0261R |
| **Future Date Prevention** | Transaction date cannot exceed current date + tolerance | GL8510R |

### 3.5 Currency/Multi-Company Rules

| Rule | Implementation |
|------|----------------|
| **Company Isolation** | All transactions keyed by $CO# (company number) |
| **IC Account Detection** | $USEICA preference ('1' = enabled, '2' = advanced, 'N' = disabled) |
| **IC Code Validation** | GL0072R validates against GLPIXCO configuration |
| **Cross-Company Posting** | GL8100R creates offsetting entries automatically |

### 3.6 Reversal Logic

**Void vs. Reverse Decision Tree:**

```
IF transaction month is OPEN:
  Execute @VOID subroutine
  - Set GTPOST = 'V'
  - Clear GTRSTS (reconcile status)
  - Clear GTPSEL (payment selection)
  
ELSE (month is CLOSED):
  Execute @REVERSE subroutine
  - Create new transaction number
  - Reverse all amounts (Z-SUB GTTAMT)
  - Post reversal via GL8100R
  - Original transaction remains posted
```

**Reference:** GL1001R.SQLRPGLE lines 911-970

### 3.7 User Authorization Controls

#### 3.7.1 Application-Level Authorization

**Primary Authorization Program:** SE0101R

```rpgle
C                   MOVE      'GL'          $APP
C                   MOVE      '2'           $AFUNC
C                   CALL      'SE0101R'
C                   PARM                    $CO#
C                   PARM                    $APP
C                   PARM                    $AFUNC
C                   PARM                    $UAREA
C                   PARM                    $ECODE
```

**Authorization Flow:**
1. User profile retrieved from system
2. SEPUAUTH table queried for GL/company access
3. $UAREA returned with user's feature flags
4. $ECODE returned with environment code

#### 3.7.2 Feature-Level Authorization Matrix

| Feature | Authorization Flag | Source Location |
|---------|-------------------|-----------------|
| **Basic Entry** | $UAREA default | SE0101R |
| **Void Document** | $AUTHADMIN | SE0261R |
| **Override Closed Period** | $UA41 = 'Y' | User area byte 41 |
| **Payroll Journal Access** | $UA17 = 'Y' | User area byte 17 |
| **IC Account Access** | GLPICUSER entry | GL0202R |
| **Document Management** | dm_toggle_value | SE4300R |

#### 3.7.3 Administrative Authorization

**Program:** SE0261R

```rpgle
C                   MOVE      'GL'          $APPL
C                   CALL      'SE0261R'
C                   PARM                    $CO#
C                   PARM                    $APPL
C                   PARM                    $AUTHADMIN
```

**Usage in System:**
- Required for void processing
- Enables period override capabilities
- Controls administrative function access

#### 3.7.4 Inter-Company Transaction Security

**Validation Program:** GL0202R

**Enforcement Points:**
1. Account entry validation
2. Transaction posting
3. Split account processing

**Failure Handling:**
- $ICSERR = '1' returned on authorization failure
- Error message displayed to user
- Transaction blocked from posting

#### 3.7.5 Security Database Schema

| File | Purpose | Key Fields |
|------|---------|------------|
| **GLPICUSER** | Inter-company user authorization | GICSCO#, GICSACCT, GICSUSER |
| **SEPUAUTH** | Application user authorization | Company, Application, User |
| **GLPAPEN** | Period and year controls | GACO#, GACAYR, GAOPF3 |

### 3.8 Application Behavior Configuration

| Preference Key | Purpose | Values |
|----------------|---------|--------|
| **&USEICACCOUNT** | Inter-company account mode | '1'=enabled, '2'=advanced, 'N'=disabled |
| **&VOIDTRANNOTE** | Void transaction notes required | 'Y'/'N' |
| **&REVERSEALLVOIDS** | Force reversal for all voids | 'Y'/'N' |
| **&VOIDORREVERSE** | Void/reverse selection mode | User selectable |

**Reference:** GL1001R.SQLRPGLE lines 287-310

---

# PART II: TECHNICAL SPECIFICATION

---

## 4. RPGLE Artifact Mapping

### 4.1 Program Call Hierarchy

```
Level 0 (Entry Points):
├── GL2010R.SQLRPGLE (A/P Open Items)
├── GL2020R.SQLRPGLE (Vendor History)
├── GL2100R.RPGLE (A/R Open Items)
├── GL2300R.SQLRPGLE (Journal Entry)
├── GL2440R.SQLRPGLE (Check Writing)
├── SE8000R.RPGLE (Service Dispatch GL)
└── SE8001R.RPGLE (Service Post GL)

Level 1 (Primary Entry):
└── GL1000R (Transaction Entry - Primary)
    ├── Calls GL1001R (Void Document)
    ├── Calls GL1100R (Posted Invoice Display)
    ├── Calls GL1006R (Document Inquiry)
    └── Calls GL1020R (Account Search)

Level 2 (Secondary Processing):
├── GL1001R.SQLRPGLE (Void Document)
│   ├── Calls GL8100R (Posting Engine)
│   ├── Calls GL0070R (Period Validation)
│   ├── Calls GL0555R (IC Configuration)
│   ├── Calls GL0202R (IC Security)
│   └── Calls GL1130R (Document Logging)
│
├── GL1100R.SQLRPGLE (Posted Invoice Display)
│   └── Display-only processing
│
└── GL1020R.SQLRPGLE (Account Search)
    └── Calls GL1021R (Account Post Validation)

Level 3 (Core Processing):
├── GL8100R.SQLRPGLE (Posting Engine)
│   ├── Calls gl8170r (Split Transaction Processing)
│   ├── Updates GLPTRNS
│   ├── Updates GLPACTL
│   └── Updates GLPUDOC
│
├── GL8510R.SQLRPGLE (Transaction Validation)
│   └── Pre-post validation logic
│
└── GL0072R (Inter-Company Code Validation)
    └── Validates GLPIXCO

Level 4 (Utility Programs):
├── SE0001R (Company Header Retrieval)
├── SE0101R (Authorization Check)
├── SE0261R (Admin Authorization)
├── SE9010R (Preference Retrieval)
└── SE9150R (Print Setup)

Level 5 (Support Programs):
├── GL0080R (Account Masking)
├── GL1080R (Control Type Validation)
├── GL1110R (Control Description)
└── GL1130R (Document Logging)
```

### 4.2 Program Inventory

| Program | Type | Purpose | Activation Group |
|---------|------|---------|-----------------|
| GL1000R | SQLRPGLE | Transaction Entry Main | *CALLER |
| GL1001R | SQLRPGLE | Void Document Processing | *CALLER |
| GL1002R | RPGLE | Unapplied Transaction Check | *CALLER |
| GL1003R | RPGLE | Balance Calculation | *CALLER |
| GL1005R | RPGLE | Invoice Duplicate Check | *CALLER |
| GL1006R | RPGLE | Document Inquiry | *CALLER |
| GL1020R | SQLRPGLE | Account Number Search | *CALLER |
| GL1021R | RPGLE | Account Post Validation | *CALLER |
| GL1040R | SQLRPGLE | Customer Search | *CALLER |
| GL1080R | RPGLE | Control Type Validation | *CALLER |
| GL1090R | RPGLE | Journal Search | *CALLER |
| GL1091R | RPGLE | Unapplied Doc Journals | *CALLER |
| GL1100R | SQLRPGLE | Posted Invoice Display | *CALLER |
| GL1110R | RPGLE | Control Description/Validate | *CALLER |
| GL1130R | RPGLE | Document Logging | *CALLER |
| GL1140R | SQLRPGLE | Unposted Documents | *CALLER |
| GL8100R | SQLRPGLE | Posting Engine | *CALLER |
| GL8510R | SQLRPGLE | Transaction Validation | *CALLER |

### 4.3 Service Programs and Binding Directories

| Binding Directory | Purpose |
|-------------------|---------|
| DTCOMMONBN | Common utilities |
| GLBN | GL-specific functions |
| FEATUREBN | Feature flag processing |
| QC2LE | C runtime |
| @CHKSQLDIR | SQL utilities |
| STRINGBND | String manipulation |
| DATEBND | Date handling |

### 4.4 Display Files

| Display File | Purpose | Format Names |
|--------------|---------|--------------|
| GL1001D | Void Document Screen | GL1001DA, GL1001DB, GL1001DC, GL1001DD |
| GL1020D | Account Search | GL1020DA |
| GL1100D | Posted Invoice Display | Various formats |
| GL1110D | Control Number Entry | GL1110DA |
| GL1140D | Unposted Documents | GL1140DA |

### 4.5 Copybooks

| Copybook | Purpose | Location |
|----------|---------|----------|
| @MSGCDE | Message code retrieval function | EISFUNC |
| @MSGCDEC | Message code constants | EISFUNC |
| GLPR | GL procedure prototypes | EISMOD |
| DTCOMMONPR | Common prototypes | EISMOD |

### 4.6 External Objects

| Object | Type | Purpose |
|--------|------|---------|
| EISWORK library | Work files | Temporary processing files |
| WFPGLxx | Physical files | Work file templates |
| WFLGLxx | Logical files | Work file views |
| Data areas (AR + JOBNUM) | *DTAARA | Session tracking |

### 4.7 Call Graph Diagram

```
                    ┌─────────────────┐
                    │   Menu/Module   │
                    │   Entry Point   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    GL1000R      │◄──────────────────────────┐
                    │ Transaction     │                           │
                    │    Entry        │                           │
                    └────────┬────────┘                           │
                             │                                    │
         ┌───────────────────┼───────────────────┐               │
         │                   │                   │               │
         ▼                   ▼                   ▼               │
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐      │
│    GL1001R      │ │    GL1100R      │ │    GL1020R      │      │
│ Void Document   │ │ Posted Invoice  │ │ Account Search  │      │
└────────┬────────┘ └─────────────────┘ └────────┬────────┘      │
         │                                       │               │
         │                                       ▼               │
         │                              ┌─────────────────┐      │
         │                              │    GL1021R      │      │
         │                              │ Post Validation │      │
         │                              └─────────────────┘      │
         │                                                       │
         ▼                                                       │
┌─────────────────┐                                              │
│    GL8100R      │◄─────────────────────────────────────────────┘
│ Posting Engine  │
└────────┬────────┘
         │
         ├──────────────────────────┐
         │                          │
         ▼                          ▼
┌─────────────────┐        ┌─────────────────┐
│    gl8170r      │        │    GLPTRNS      │
│ Split Processing│        │   (Database)    │
└─────────────────┘        └─────────────────┘
```

---

## 5. Data Model

### 5.1 Database File Inventory

#### Physical Files (PF)

| File | System Name | Purpose | Key Fields |
|------|-------------|---------|------------|
| GLPTRNS | GL_TRNS | Transaction Master | GTCO#, GTTRN#, GTSEQ# |
| GLPMAST | - | Account Master | GMCO#, GMACCT, GMMODE, GMYEAR |
| GLPAPEN | - | Accounting Period | GACO# |
| GLPCUST | - | Customer Master | GCCO#, GCCUS# |
| GLPJRND | - | Journal Definitions | GJCO#, GJJRNL |
| GLPACTL | - | Active Control | GLCO#, GLDOC# |
| GLPUDOC | - | Unapplied Documents | GZCO#, GZDOC# |
| GLPOPYR | - | Open Periods by Year | Company, Year |
| GLPIXCO | - | Inter-Company Codes | Company, IC Code |
| GLPICUSER | - | IC User Authorization | GICSCO#, GICSACCT, GICSUSER |
| GLPCTYP | - | Customer Types | GYCO#, GYCTYP |
| GLPDTIM | - | Date/Time Stamp | Company, Document |
| GLPBNKA | - | Bank Account | GBCO#, GBBNK# |
| GLPVTYP | - | Vendor Types | Company, Type |
| GLPPTRM | - | Payment Terms | Company, Terms |

#### Logical Files (LF)

| File | Base File | Purpose | Key |
|------|-----------|---------|-----|
| GLLTRNA | GLPTRNS | By Company/Doc Type/Doc# | GTCO#, GTDTYP, GTDOC# |
| GLLTRNB | GLPTRNS | By Company/Journal | GTCO#, GTJRNL |
| GLLTRNI | GLPTRNS | Transaction Index | GTCO#, GTTRN#, GTSEQ# |
| GLLTRNR | GLPTRNS | By Reference | GTCO#, GTREF# |
| GLLTRNS | GLPTRNS | By Status | GTCO#, GTRSTS |
| GLLCUSV | GLPCUST | Customer by Vendor# | GCCO#, GCVND# |
| GLLCUSC | GLPCUST | Customer by Customer# | GCCO#, GCCUS# |

### 5.2 Field Dictionary - GLPTRNS

| Field | System Name | Type/Length | Purpose |
|-------|-------------|-------------|---------|
| GTCO# | Company Number | CHAR(3) | Company identifier |
| GTTRN# | Transaction Number | DEC(11,0) | Unique transaction ID |
| GTSEQ# | Sequence Number | DEC(5,0) | Line sequence within transaction |
| GTDTYP | Document Type | CHAR(1) | S=Sale, P=Purchase, J=Journal |
| GTTYPE | Record Type | CHAR(1) | R=Receivable, P=Payable, $=Cash |
| GTPOST | Post Status | CHAR(1) | Y=Posted, V=Voided, blank=Unposted |
| GTRSTS | Reconcile Status | CHAR(1) | Y=Reconciled, blank=Not |
| GTADJUST | Adjustment | CHAR(1) | Y=Adjustment transaction |
| GTPSEL | Payment Selection | CHAR(1) | Selection for payment |
| GTJRNL | Journal Code | CHAR(3) | Journal type (PAY, GJE, etc.) |
| GTDATE | Transaction Date | DATE | Date of transaction |
| GTRDATE | Reconcile Date | DATE | Date reconciled |
| GTSDATE | Statement Date | DATE | Statement date |
| GTACCT | Account Number | CHAR(10) | GL account number |
| GTCTL# | Control Number | CHAR(17) | Customer/Vendor number |
| GTDOC# | Document Number | CHAR(17) | Invoice/Check number |
| GTREF# | Reference Number | CHAR(17) | Reference document |
| GTODOC# | Original Document | CHAR(17) | Original document reference |
| GTRDOC# | Related Document | CHAR(17) | Related document |
| GTDESC | Description | CHAR(30) | Transaction description |
| GTTAMT | Transaction Amount | DEC(14,2) | Debit(+)/Credit(-) amount |
| GTCOST | Cost Amount | DEC(14,2) | Cost basis amount |
| GTOCO# | Original Company | CHAR(3) | Source company |
| GTVND# | Vendor Number | CHAR(9) | Vendor reference |
| GTCTLO | Control Original | CHAR(17) | Original control |
| GTRDTYP | Related Doc Type | CHAR(1) | Related document type |

### 5.3 Relationship Mapping

```
GLPTRNS (Transactions)
    │
    ├──► GLPMAST (via GTCO#, GTACCT)
    │    Account validation and balance updates
    │
    ├──► GLPCUST (via GTCO#, GTCTL#)
    │    Customer information for A/R transactions
    │
    ├──► GLPAPEN (via GTCO#)
    │    Period control and transaction numbering
    │
    ├──► GLPJRND (via GTCO#, GTJRNL)
    │    Journal type definitions
    │
    ├──► GLPACTL (via GLCO#, GLDOC#)
    │    Active transaction control/locking
    │
    ├──► GLPUDOC (via GZCO#, GZDOC#)
    │    Unapplied document tracking
    │
    └──► GLPIXCO (via GTCO#, IC Code)
         Inter-company code validation
```

### 5.4 Lookup/Reference Tables

| Table | Usage Location | Purpose |
|-------|----------------|---------|
| GLPCTYP | GL2131R, GL2132R, GL2137R | Customer type descriptions |
| GLPVTYP | GL2010R | Vendor type validation |
| GLPPTRM | GL2020R | Payment terms lookup |
| BOLNAME | GL2100R, GL1110R | Bill of lading names |
| SEPPRTD | GL2130R, GL2140R | Printer definitions |
| SEPCOMP | Various | Company information |

---

## 6. Error Handling & Message Flow

### 6.1 Error Message Inventory

| Message ID | Description | Trigger Condition |
|------------|-------------|-------------------|
| GL10001 | Invalid account number | Account not found in GLPMAST |
| GL10002 | Account inactive | GMACTIVE = 'N' |
| GL10003 | Adjustment error | Period adjustment validation failed |
| GL10004 | Document already exists | Duplicate document number |
| GL10005 | Transaction out of balance | Debits ≠ Credits |
| C001121 | Interest Charged | Interest transaction description |
| C003274 | Control type error | Invalid control type for account |
| D003336 | Reconciled deposit warning | Attempt to void reconciled item |
| D003622 | Report description | Print job description |

### 6.2 Error Mechanisms

**Indicator-Based Error Handling:**
```rpgle
C     *IN40         Error indicator (validation failure)
C     *IN96         End of file indicator
C     *IN98         Record not found indicator
C     *IN12         Exit/Cancel indicator
```

**Error Flow:**
1. Validation routine sets error indicator
2. Error message ID stored in $MSGID variable
3. @SFERROR subroutine displays error on subfile
4. User corrects and resubmits

**Audit Logging:**
- GL1130R logs all document actions
- GLPDTIM stores date/time stamps
- SYPLOG receives error messages for tracking

### 6.3 Batch vs. Online Error Handling

| Mode | Error Behavior |
|------|----------------|
| **Online** | Display error on screen, allow correction |
| **Batch** | Log error to SYPLOG, skip record, continue processing |
| **Import** | Return error indicator to caller, rollback transaction |

---

## 7. UI Behavior

### 7.1 Function Key Map

| Function Key | Action | Context |
|--------------|--------|---------|
| F3 | Exit | All screens |
| F4 | Prompt/Search | Account number, Control number fields |
| F5 | Refresh | Display screens |
| F6 | Drill-down/Detail | View source document |
| F10 | Post/Process | Transaction entry |
| F11 | Alternate view | Toggle display format |
| F12 | Cancel/Previous | Return to prior screen |
| F18 | Print | Generate report |
| F21 | Enter transactions | From summary screen |
| F24 | More keys | Display additional function keys |

### 7.2 Context-Sensitive Dispatch

**$ECALL Parameter Routing:**
```rpgle
IF $ECALL = 'I':    // Inquiry mode
  Display read-only
  Disable modification keys
  
ELSE IF $ECALL = 'V':    // Void mode
  Enable void processing
  Call GL1001R
  
ELSE:    // Standard entry
  Full feature access
```

### 7.3 Screen Formats

| Format | Purpose | Key Fields |
|--------|---------|------------|
| GL1001DA | Transaction entry line | Account, Control, Amount |
| GL1001DB | Subfile control | Function keys, totals |
| GL1001DC | Message/Error display | Error messages |
| GL1001DD | Confirmation | Void/Post confirmation |

---

## 8. Posting Logic

### 8.1 Pre-Post Validation Checklist

1. **Transaction Balance**: SUM(Debits) = SUM(Credits)
2. **Account Validity**: All accounts exist and are active
3. **Period Open**: Transaction date within open fiscal period
4. **Control Type Match**: Control numbers match account control types
5. **IC Authorization**: User authorized for inter-company accounts
6. **IC Code Valid**: Inter-company codes exist in GLPIXCO
7. **Document Not Duplicate**: Document number unique for type
8. **No Unapplied Conflicts**: GLPUDOC cleared or matched

### 8.2 Validation Stage Flow

```
┌─────────────────┐
│  Begin Posting  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│ GL8510R         │────►│ Error Return    │
│ Pre-Validation  │ Fail│                 │
└────────┬────────┘     └─────────────────┘
         │ Pass
         ▼
┌─────────────────┐
│ GL0070R         │
│ Period Check    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ GL8100R         │
│ Posting Engine  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Database Update │
│ COMMIT          │
└─────────────────┘
```

### 8.3 Posting Engine Database Operations (GL8100R)

**Files Read:**
- GLPTRNS (Transaction details)
- GLPMAST (Account validation)
- GLPAPEN (Period information)
- GLPJRND (Journal definitions)
- GLPCTYP (Customer types)

**Files Written:**
- GLPTRNS (Update GTPOST status)
- GLPACTL (Remove active control record)
- GLPUDOC (Add/remove unapplied documents)
- GLPDTIM (Date/time stamp)

**GL8100R Processing Logic:**
```rpgle
// Main posting sequence
EXSR @DATETIME        // Set date/time
EXSR @JRND           // Process journal definitions

IF $FUNC = 'V':       // Void processing
  GOTO @END
ENDIF

gl8170r($co#:$trn#:$useica:$func:$error:OT_GLpost:ErrorMessage)

IF $ERROR = 'Y':
  // Error handling
  CHAIN $key1 GLPACTLA
  IF %found:
    DELETE GLPACTLA
  ENDIF
ENDIF
```

**Reference:** GL8100R.SQLRPGLE lines 318-350

### 8.4 Commit Control

**Commit Unit:**
- All transaction lines for a single document number
- Control records (GLPACTL, GLPUDOC)
- Related date/time stamps

**Commit Configuration:**
```sql
exec sql
  set option commit = *none, usrprf = *owner, dynusrprf = *owner,
             datfmt = *iso, closqlcsr = *endmod;
```

**Rollback Triggers:**
- Validation failure in GL8510R
- Database error during update
- Inter-company posting failure

**Post-Rollback State:**
- Transaction remains in unposted status
- GLPACTL record retained
- Error logged to SYPLOG

### 8.5 Voiding Logic

**@VOID Subroutine (GL1001R):**
```rpgle
C     @VOID         BEGSR
C                   EVAL      $VOIDDOC= 'Y'
C                   MOVE      *ZERO         $R
      DO $RN:
        CHAIN GL1001DA
        IF found AND GTPOST <> 'V':
          Eval $USEQ# = 0
          CHAIN GLPJRNDA
          CHAIN GLPTRNSA
          IF found AND GTPOST = 'Y':
            Eval $USEQ# = GTSEQ#
            MOVE 'V' GTPOST         // Mark as voided
            eval gtrsts = *blank    // Clear reconcile
            UPDATE GLPTRNSA
          ENDIF
        ENDIF
      ENDDO
```

**@REVERSE Subroutine (GL1001R):**
```rpgle
C     @REVERSE      BEGSR
C                   EVAL      $VOIDDOC= 'Y'
C                   EVAL      $POST = 'N'
      // Get new transaction number
      CHAIN GLPAPENA
      ADD 1 GATRN#
      UPDATE GLPAPENA
      
      DO transaction lines:
        // Copy original
        // Reverse amount sign
        Z-SUB GTTAMT GTTAMT
        // Set new transaction number
        MOVE GATRN# $ATRN#
        // Post via GL8100R
        CALL 'GL8100R'
      ENDDO
```

### 8.6 Unapplied Document Mechanism

**GLPUDOC Table Purpose:**
- Tracks documents awaiting full application
- Links partial payments to invoices
- Maintains open item status

**Processing:**
```rpgle
// On posting incomplete application
IF document has remaining balance:
  CHAIN GLPUDOCA
  IF not %found:
    WRITE GLPUDOCA
  ELSE:
    UPDATE with new balance
  ENDIF
ENDIF

// On void/reversal
CHAIN GLPUDOCA
IF %found:
  DELETE GLPUDOCA
ENDIF
```

### 8.7 Split Account Processing

**Detection:**
- Multiple detail lines with different accounts
- Inter-company accounts requiring offset

**gl8170r Processing:**
- Validates split balances
- Creates inter-company entries
- Ensures each split segment balances

### 8.8 Inter-Company Posting

**Flow:**
1. Detect IC account (GMCTYP or IC flag)
2. Validate user authorization (GL0202R)
3. Validate IC code (GL0072R)
4. Create offset in target company
5. Post both sides atomically

**Reference:** GL1001R.SQLRPGLE lines 308-325

### 8.9 Journaling

**Journal Types (GTJRNL):**
| Code | Description |
|------|-------------|
| GJE | General Journal Entry |
| PAY | Payroll |
| CRE | Credit |
| CHK | Check |
| DEP | Deposit |

**Journal Definition Lookup:**
```rpgle
CHAIN($CO#:$JRNL) GLPJRNDA
// Returns journal configuration
```

### 8.10 Transaction Lifecycle

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Created │───►│ Entered │───►│ Posted  │───►│ Voided  │
│ (New)   │    │ (GTPOST │    │ (GTPOST │    │ (GTPOST │
│         │    │ = blank)│    │ = 'Y')  │    │ = 'V')  │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
                    │              │
                    │              ▼
                    │         ┌─────────┐
                    └────────►│Reconciled│
                              │(GTRSTS  │
                              │ = 'Y')  │
                              └─────────┘
```

---

## 9. Interfaces & Dependencies

### 9.1 Upstream Callers

| Caller | Purpose | Parameters Passed |
|--------|---------|-------------------|
| **GL2010R** | A/P Open Items | $CO#, $HNAM, $ECODE, $DTYP, $DOC#, $JRNL, $TRN#, $ECALL, $IN12 |
| **GL2020R** | Vendor History | Same as GL2010R |
| **GL2100R** | A/R Open Items | $CO#, $HNAM, $ECODE, $IN03, $TRTOT, $OPT |
| **GL2300R** | Journal Entry | $CO#, $HNAM, $ECODE, GTDTYP, GTDOC#, GTJRNL, GTTRN#, $ECALL |
| **GL2440R** | Check Writing | $CO#, document parameters |
| **SE8000R** | Service Dispatch | SCGLCO, $ATRN#, $JRNL, $GFUNC |
| **SE8001R** | Service Post | Same as SE8000R |

### 9.2 Downstream Dependencies

| Called Program | Purpose | From |
|----------------|---------|------|
| **GL1001R** | Void document | GL1000R |
| **GL8100R** | Posting engine | GL1001R, GL2130R, SE8000R, SE8001R |
| **GL1020R** | Account search | GL1000R, GL2100R |
| **GL1100R** | Posted invoice | GL1000R, GL2010R, GL2150R |
| **GL1110R** | Control validation | GL1000R |
| **GL8510R** | Pre-post validation | GL1000R |

### 9.3 Internal/System Program Calls

| Program | Purpose | Parameters |
|---------|---------|------------|
| **SE0001R** | Company header | $CO#, $HNAM60 |
| **SE0101R** | Authorization | $CO#, $APP, $AFUNC, $UAREA, $ECODE |
| **SE0261R** | Admin auth | $CO#, $APPL, $AUTHADMIN |
| **SE9010R** | Preference | $CO#, XZKEY, XZVALUE |
| **SE9150R** | Print setup | Multiple print parameters |
| **GL0070R** | Period check | $CO#, $DATE, $ADJUST, $ERR |
| **GL0072R** | IC validation | $CO#, $ACCT#, $XDATE, $ICCODE4, $ICSERR4 |
| **GL0080R** | Account mask | $CO#, $ACCT10, $ACCT12 |
| **GL0202R** | IC security | $CO#, $ACCT#, $USER, $ICSCODE, $ICSERR |
| **GL0555R** | IC config | $CO#, $ICC, $ICA, $ICR, $USEICA |

### 9.4 Cross-Module Interfaces

| Module | Interface Type | Integration Point |
|--------|---------------|-------------------|
| **Accounts Payable** | Bidirectional | GL2010R ↔ GL1000R |
| **Accounts Receivable** | Bidirectional | GL2100R ↔ GL1000R |
| **Service Dispatch** | Outbound | SE8000R → GL8100R |
| **Statement Printing** | Outbound | GL2130R → GL8100R |
| **Check Writing** | Outbound | GL2440R → GL1000R |

### 9.5 Batch/Import Integration

**SE8000R/SE8001R Integration:**
```rpgle
// From SE8000R
$STAT = 'A';
EXSR @ACTL;
Evalr $ATRN# = %char($TRN#);
$GFUNC = '1';

CALL 'GL8100R'
  PARM SCGLCO
  PARM $ATRN#
  PARM $JRNL
  PARM $GFUNC

CHAIN $KEY3 GLPACTLA
IF %FOUND:
  DELETE GLPACTLA
ENDIF
```

### 9.6 Application Preferences

| Preference | Location | Impact |
|------------|----------|--------|
| **&USEICACCOUNT** | SE9010R | Enables/disables IC processing |
| **&VOIDTRANNOTE** | SE9010R | Requires notes on void |
| **&REVERSEALLVOIDS** | SE9010R | Forces reversal vs void |
| **&VOIDORREVERSE** | SE9010R | User choice for void method |

---

## 10. Session Management

### 10.1 Document Locking

**Mechanism:** GLPACTL (Active Control Table)

**Lock Acquisition:**
```rpgle
// Create lock record
EVAL GLCO# = $CO#
EVAL GLDOC# = $DOC#
EVAL GLDTYP = $DTYP
EVAL GLCTL# = $CTL#
EVAL GLSTAT = 'A'        // Active status
WRITE GLPACTLA
```

**Lock Check:**
```rpgle
CHAIN($CO#:$DOC#) GLPACTLA
IF %FOUND:
  IF GLUSER <> $USER:
    // Document locked by another user
    Set error
  ENDIF
ENDIF
```

### 10.2 Document Takeover

**Takeover Conditions:**
- Original user session ended
- Administrative override
- Timeout exceeded

**Takeover Process:**
```rpgle
// Check if user can take over
IF $AUTHADMIN = 'Y':
  UPDATE GLPACTLA
  SET GLUSER = $USER
ELSE:
  Display locked message
ENDIF
```

### 10.3 Active Session Tracking

**Data Area Usage:**
```clle
// Create session data area
CHGVAR VAR(&DTA) VALUE('AR' *TCAT &JOBNUM)
CRTDTAARA DTAARA(EISWORK/&DTA) TYPE(*CHAR) LEN(14)
CHGDTAARA DTAARA(EISWORK/&DTA (1 14)) VALUE(&TRTOT)

// Retrieve session data
RTVDTAARA DTAARA(EISWORK/&DTA (1 14)) RTNVAR(&TRTOT)

// Clean up on exit
DLTDTAARA DTAARA(EISWORK/&DTA)
```

**Reference:** GL2100C.CLLE lines 81-95

**Session Information in UDS:**
```rpgle
D                UDS
D $DTFMT               1017   1017    // Date format
D Enterprise           1018   1021    // Enterprise code
D $LANG                1022   1024    // Language code
```

**SDS (System Data Structure):**
```rpgle
D                SDS
D $USER                 254    263    // Current user
D  #PARMS           *PARMS           // Parameter count
```

---

## Appendix A: Complete Program Cross-Reference

| Program | Calls | Called By |
|---------|-------|-----------|
| GL1000R | GL1001R, GL1020R, GL1100R, GL8510R | GL2010R, GL2100R, GL2300R, GL2440R |
| GL1001R | GL8100R, GL0070R, GL0555R, GL0202R, GL1130R | GL1000R |
| GL1020R | GL1021R | GL1000R, GL2100R |
| GL1100R | - | GL1000R, GL2010R, GL2130R |
| GL8100R | gl8170r | GL1001R, SE8000R, SE8001R, GL2130R |
| GL8510R | GL0070R, GL0072R | GL1000R |

## Appendix B: Key Data Structures

**Transaction Entry Data Structure:**
```rpgle
d $doc#_ds        ds
d  long_$doc#                   17a
d  extra_$doc#                   8a   overlay(long_$doc#)
d  $doc#                         9a   overlay(long_$doc#:9)

d $acct_ctl#                    27
d  gtacct                       10    overlay($acct_ctl#)
d  gtctl#                       17    overlay($acct_ctl#:11)
```

**Period Control Structure:**
```rpgle
D               E DS                  EXTNAME(GLPOPYR)
D  $om                    9     21    DIM(13)
```

---

*Document generated from knowledge graph analysis of GL1000C/GL21 codebase*
*Source files: EISSRC_Program Source Codes/, DB2 Schema/*
*Generated: March 16, 2026*
