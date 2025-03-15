<style>
  pre {
    background-color: #1e1e1e !important;
    color: #d4d4d4 !important;
    padding: 16px !important;
    border-radius: 6px !important;
    overflow: auto !important;
  }
  
  code {
    font-family: 'Consolas', 'Monaco', 'Courier New', monospace !important;
    font-size: 14px !important;
  }
  
  body {
    font-family: Arial, Helvetica, sans-serif !important;
    font-size: 12pt !important;
    line-height: 1.5 !important;
    max-width: 900px !important;
    margin: 0 auto !important;
    padding: 20px !important;
  }
  
  h1, h2, h3, h4 {
    font-family: Arial, Helvetica, sans-serif !important;
    color: #333 !important;
  }
  
  h1 {
    font-size: 20pt !important;
    margin-top: 20px !important;
  }
  
  h2 {
    font-size: 16pt !important;
    margin-top: 16px !important;
  }
  
  h3 {
    font-size: 14pt !important;
    margin-top: 14px !important;
  }
  
  table {
    border-collapse: collapse !important;
    width: 100% !important;
    margin: 16px 0 !important;
  }
  
  th, td {
    border: 1px solid #ddd !important;
    padding: 8px !important;
    text-align: left !important;
  }
  
  th {
    background-color: #f2f2f2 !important;
    font-weight: bold !important;
  }
  
  img {
    max-width: 100% !important;
    height: auto !important;
  }
</style>

# Tour Details System - Low-Level Design Document

## 1. Requirement Summary

The Tour Details System is designed to maintain tour-related records in the TOUR_DETAILS database, offering essential record management capabilities. The system will handle tour information including places, guides, languages, dates, group sizes, and pricing.

Core functionalities include:

- Database creation of two tables: TOUR_DETAILS and SEASON_DISCOUNT
- Special handling for null values in the TOUR_GUIDE field
- Multiple-tier discount calculation based on group size and language
- Final price calculation with additional reductions based on discount ranges
- Generation of formatted output files for processed records
- Database operations including selects with joins and case conversion features

## 2. Component List

### 2.1 Database Components

| Component Type | Component Name | Description | Input/Output |
|----------------|----------------|-------------|--------------|
| Table | TOUR_DETAILS | Stores tour information. Fields include: TOUR_ID, TOUR_PLACE, TOUR_GUIDE, LANGUAGE, TOUR_DATE, GROUP_SIZE, PRICE_PER_HEAD. | Input/Output |
| Table | SEASON_DISCOUNT | Stores calculated discount information based on tour details. Fields include: TOUR_PLACE, TOUR_GUIDE, TOUR_DATE, DISCOUNT, FINAL_PRICE, GROUP_DISCOUNT. | Output |

### 2.2 SPUFI Components

| Component Type | Component Name | Description | Input/Output |
|----------------|----------------|-------------|--------------|
| SPUFI Member | SB12L\<yyy\> | Creates the TOUR_DETAILS table. | Input |
| SPUFI Member | SB22L\<yyy\> | Creates the SEASON_DISCOUNT table. | Input |
| SPUFI Member | SB32L\<yyy\> | Inserts data into the TOUR_DETAILS table. | Input |
| SPUFI Member | SB42L\<yyy\> | Performs a LEFT OUTER JOIN between TOUR_DETAILS and SEASON_DISCOUNT tables, filtering for LANGUAGE = 'ENG' and GROUP_SIZE > 10. | Input |
| SPUFI Member | SB52L\<yyy\> | Selects records from SEASON_DISCOUNT where DISCOUNT <= 20, converting relevant fields to lowercase. | Input |

### 2.3 COBOL Program Components

| Component Type | Component Name | Description | Input/Output |
|----------------|----------------|-------------|--------------|
| COBOL Program | CB12L\<yyy\> | Main COBOL program responsible for processing tour records, calculating discounts, and generating output files. | Input |
| DCLGEN | DB12L\<yyy\> | Declaration generator for the TOUR_DETAILS table, providing COBOL data structures. | Input |
| DCLGEN | DB22L\<yyy\> | Declaration generator for the SEASON_DISCOUNT table, providing COBOL data structures. | Input |

### 2.4 JCL Components

| Component Type | Component Name | Description | Input/Output |
|----------------|----------------|-------------|--------------|
| JCL | JB12L\<yyy\> | JCL script that compiles the CB12L\<yyy\> COBOL program and executes it. | Input |

### 2.5 Files

| File | DD Name | Record Length | Description | Input/Output |
|------|---------|--------------|-------------|--------------|
| \<TLABID\>.L2L.TOUR.PS | OUTTOUR | 80 | Output file for processed records with discount information. | Output |
| \<TLABID\>.L2L.ODEL.PS | OUTODEL | 80 | Output file for records with null TOUR_GUIDE values. | Output |

## 3. Flowchart

![System Flowchart](media/image1.png)

## 4. Pseudo Code

```
IDENTIFICATION DIVISION.
PROGRAM-ID. CB12L<yyy>.

DATA DIVISION.

PROCEDURE DIVISION.

0000-MAIN-PARA.
    PERFORM 1000-INITIALIZATION
    PERFORM 2000-PROCESS-TOUR-RECORDS
    PERFORM 9000-TERMINATION.

1000-INITIALIZATION.
    OPEN OUTPUT OUTTOUR
    OPEN OUTPUT OUTODEL
    INITIALIZE WS-VARIABLES
    EXEC SQL
        CONNECT TO DBLAB01
    END-EXEC.

2000-PROCESS-TOUR-RECORDS.
    EXEC SQL
        DECLARE TOUR-CURSOR CURSOR FOR
        SELECT TOUR_PLACE, TOUR_GUIDE, LANGUAGE, TOUR_DATE, GROUP_SIZE, PRICE_PER_HEAD
        FROM TOUR_DETAILS
        WHERE GROUP_SIZE > 1 AND PRICE_PER_HEAD > 1000
    END-EXEC
    EXEC SQL
        OPEN TOUR-CURSOR
    END-EXEC
    EXEC SQL
        FETCH TOUR-CURSOR INTO
        :WS-TOUR-PLACE, :WS-TOUR-GUIDE :WS-TOUR-GUIDE-NULL-IND,
        :WS-LANGUAGE, :WS-TOUR-DATE, :WS-GROUP-SIZE, :WS-PRICE-PER-HEAD
    END-EXEC
    PERFORM 3000-PROCESS-CURSOR
    UNTIL SQLCODE NOT = 0
    EXEC SQL
        CLOSE TOUR-CURSOR
    END-EXEC.

3000-PROCESS-CURSOR.
    IF WS-TOUR-GUIDE-NULL-IND < 0
        MOVE 'YTD' TO WS-TOUR-GUIDE
        PERFORM 4000-WRITE-NULL-RECORD
        PERFORM 5000-DELETE-TOUR-DETAIL
    ELSE
        PERFORM 6000-CALC-DISCOUNT
        COMPUTE WS-GROUP-DISCOUNT = WS-GROUP-SIZE * WS-DISCOUNT
        PERFORM 7000-CALC-FINAL-PRICE
        PERFORM 8000-INSERT-SEASON-DISCOUNT
        PERFORM 8100-WRITE-PROCESSED-RECORD
    END-IF
    EXEC SQL
        FETCH TOUR-CURSOR INTO
        :WS-TOUR-PLACE, :WS-TOUR-GUIDE :WS-TOUR-GUIDE-NULL-IND,
        :WS-LANGUAGE, :WS-TOUR-DATE, :WS-GROUP-SIZE, :WS-PRICE-PER-HEAD
    END-EXEC.

4000-WRITE-NULL-RECORD.
    MOVE WS-TOUR-PLACE TO OUT-ODEL-TOUR-PLACE
    MOVE WS-TOUR-GUIDE TO OUT-ODEL-TOUR-GUIDE
    MOVE WS-LANGUAGE TO OUT-ODEL-LANGUAGE
    MOVE WS-TOUR-DATE TO OUT-ODEL-TOUR-DATE
    MOVE WS-GROUP-SIZE TO OUT-ODEL-GROUP-SIZE
    MOVE WS-PRICE-PER-HEAD TO OUT-ODEL-PRICE-PER-HEAD
    WRITE OUTODEL-REC.

5000-DELETE-TOUR-DETAIL.
    EXEC SQL
        DELETE FROM TOUR_DETAILS
        WHERE CURRENT OF TOUR-CURSOR
    END-EXEC.

6000-CALC-DISCOUNT.
    EVALUATE TRUE
    WHEN WS-GROUP-SIZE = 5
        COMPUTE WS-DISCOUNT = WS-PRICE-PER-HEAD * 0.01
    WHEN WS-GROUP-SIZE > 5 AND WS-GROUP-SIZE <= 10
    AND WS-LANGUAGE = 'ENG'
        COMPUTE WS-DISCOUNT = WS-PRICE-PER-HEAD * 0.02
    WHEN WS-GROUP-SIZE > 10 AND WS-GROUP-SIZE <= 20
        COMPUTE WS-DISCOUNT = WS-PRICE-PER-HEAD * 0.03
    WHEN WS-GROUP-SIZE > 20 AND WS-GROUP-SIZE <= 30
    AND WS-LANGUAGE = 'ENG'
        COMPUTE WS-DISCOUNT = WS-PRICE-PER-HEAD * 0.018
    WHEN WS-GROUP-SIZE > 20 AND WS-GROUP-SIZE <= 30
    AND WS-LANGUAGE = 'TAM'
        COMPUTE WS-DISCOUNT = WS-PRICE-PER-HEAD * 0.015
    END-EVALUATE.

7000-CALC-FINAL-PRICE.
    EVALUATE TRUE
    WHEN WS-DISCOUNT >= 10 AND WS-DISCOUNT <= 20
        COMPUTE WS-FINAL-PRICE = WS-PRICE-PER-HEAD -
        (WS-GROUP-SIZE * WS-DISCOUNT) - 10
    WHEN WS-DISCOUNT > 20 AND WS-DISCOUNT <= 40
        COMPUTE WS-FINAL-PRICE = WS-PRICE-PER-HEAD -
        (WS-GROUP-SIZE * WS-DISCOUNT) - 12
    WHEN WS-DISCOUNT > 40 AND WS-DISCOUNT <= 50
        COMPUTE WS-FINAL-PRICE = WS-PRICE-PER-HEAD -
        (WS-GROUP-SIZE * WS-DISCOUNT) - 13
    WHEN WS-DISCOUNT > 50 AND WS-DISCOUNT < 100
        COMPUTE WS-FINAL-PRICE = WS-PRICE-PER-HEAD -
        (WS-GROUP-SIZE * WS-DISCOUNT) - 9
    END-EVALUATE.

8000-INSERT-SEASON-DISCOUNT.
    EXEC SQL
        INSERT INTO SEASON_DISCOUNT
        (TOUR_PLACE, TOUR_GUIDE, TOUR_DATE, DISCOUNT, FINAL_PRICE, GROUP_DISCOUNT)
        VALUES
        (:WS-TOUR-PLACE, :WS-TOUR-GUIDE, :WS-TOUR-DATE,
        :WS-DISCOUNT, :WS-FINAL-PRICE, :WS-GROUP-DISCOUNT)
    END-EXEC.

8100-WRITE-PROCESSED-RECORD.
    MOVE WS-TOUR-PLACE TO OUT-TOUR-TOUR-PLACE
    MOVE WS-TOUR-GUIDE TO OUT-TOUR-TOUR-GUIDE
    MOVE WS-LANGUAGE TO OUT-TOUR-LANGUAGE
    MOVE WS-TOUR-DATE TO OUT-TOUR-TOUR-DATE
    MOVE WS-GROUP-SIZE TO OUT-TOUR-GROUP-SIZE
    MOVE WS-DISCOUNT TO OUT-TOUR-DISCOUNT
    MOVE WS-GROUP-DISCOUNT TO OUT-TOUR-GROUP-DISCOUNT
    MOVE WS-FINAL-PRICE TO OUT-TOUR-FINAL-PRICE
    WRITE OUTTOUR-REC.

9000-TERMINATION.
    CLOSE OUTTOUR
    CLOSE OUTODEL
    EXEC SQL
        DISCONNECT
    END-EXEC
    STOP RUN.
```

## 5. Test Cases

| Test Case Description | Status (Pass/Fail) |
|------------------------|-------------------|
| Create TOUR_DETAILS table with correct column definitions | |
| Create SEASON_DISCOUNT table with correct column definitions | |
| Insert sample records into TOUR_DETAILS table | |
| Process records with NULL values in TOUR_GUIDE field | |
| Calculate discount for GROUP_SIZE = 5 | |
| Calculate discount for GROUP_SIZE between 6 and 10 with LANGUAGE = "ENG" | |
| Calculate discount for GROUP_SIZE between 11 and 20 | |
| Calculate discount for GROUP_SIZE between 21 and 30 with LANGUAGE = "ENG" | |
| Calculate discount for GROUP_SIZE between 21 and 30 with LANGUAGE = "TAM" | |
| Calculate FINAL_PRICE for DISCOUNT between 10 and 20 | |
| Calculate FINAL_PRICE for DISCOUNT between 21 and 40 | |
| Calculate FINAL_PRICE for DISCOUNT between 41 and 50 | |
| Calculate FINAL_PRICE for DISCOUNT between 51 and 99 | |
