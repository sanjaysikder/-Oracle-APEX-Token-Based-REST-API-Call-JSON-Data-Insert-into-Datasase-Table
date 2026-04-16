# 🔐 Oracle APEX: Token-Based REST API Call & JSON Data Insert into Datasase Table

## 📌 Overview

This document explains how to:

* Authenticate using a **token-based REST API**
* Fetch data from an external API
* Parse JSON response using `APEX_JSON` and `JSON_TABLE`
* Insert the retrieved data into an Oracle database table

---

## ⚙️ Procedure: `GET_ACHALAN_INFO`

```sql
CREATE OR REPLACE PROCEDURE GET_ACHALAN_INFO (p_chalan_no VARCHAR)
IS
    l_json_values    apex_json.t_values;
    l_response_clob  CLOB;
    l_request_body   CLOB;
    l_token          CLOB;
    l_result_clob    CLOB;

    PRAGMA AUTONOMOUS_TRANSACTION;

BEGIN
    -- Step 1: Prepare Request Body for Token
    l_request_body := 'UserName=police_test&Password=dbtest&grant_type=password';

    -- Step 2: Set Headers for Token Request
    apex_web_service.g_request_headers(1).name  := 'Content-Type';
    apex_web_service.g_request_headers(1).value := 'application/x-www-form-urlencoded';

    apex_web_service.g_request_headers(2).name  := 'Authorization';
    apex_web_service.g_request_headers(2).value := 'Basic UE9MSUNFOlBPTElD1QS04MkM1LUEwMzU5RTcxRTFGMg==';

    -- Step 3: Call Token API
    l_response_clob := apex_web_service.make_rest_request(
        p_url         => 'https://acpi.financebd.gov.bd/accsapi/api/token',
        p_http_method => 'POST',
        p_body        => l_request_body
    );

    -- Step 4: Parse Token from JSON Response
    apex_json.parse(l_response_clob);
    l_token := apex_json.get_varchar2(p_path => 'access_token');

    -- Step 5: Set Headers for Data API
    apex_web_service.g_request_headers.delete;

    apex_web_service.g_request_headers(1).name  := 'Content-Type';
    apex_web_service.g_request_headers(1).value := 'application/json';

    apex_web_service.g_request_headers(2).name  := 'Authorization';
    apex_web_service.g_request_headers(2).value := 'Bearer ' || l_token;

    -- Step 6: Call Challan তথ্য API
    l_result_clob := apex_web_service.make_rest_request(
        p_url         => 'https://acpi.financebd.gov.bd/accsapi/api/clients/payment/challanInfo?challan_no=' || p_chalan_no,
        p_http_method => 'GET'
    );

    -- Step 7: Parse JSON and Insert into Table
    FOR t IN (
        SELECT j.paymentid,
               j.challan_no,
               j.depositorname,
               j.paymentdate,
               j.paidamount
        FROM json_table(l_result_clob, '$[*]'
            COLUMNS
                paymentid      VARCHAR2(20)  PATH '$.paymentid',
                challan_no     VARCHAR2(20)  PATH '$.challan_no',
                depositorname  VARCHAR2(200) PATH '$.depositorname',
                paymentdate    VARCHAR2(100) PATH '$.paymentdate',
                paidamount     VARCHAR2(20)  PATH '$.paidamount'
        ) j
    )
    LOOP
        BEGIN
            INSERT INTO CHALAN_INFO (
                TRANID,
                SCRLNO,
                TRANTYPE,
                PAYEE,
                TRANDT,
                AMOUNT
            )
            VALUES (
                t.paymentid,
                t.challan_no,
                NULL,
                t.depositorname,
                TO_DATE(t.paymentdate, 'DD/MM/YYYY'),
                t.paidamount
            );

            COMMIT;

        END;
    END LOOP;

    -- Debug Output
    DBMS_OUTPUT.PUT_LINE('API Response: ' || l_result_clob);

END;
/
```

---

## 🔄 Step-by-Step Flow

### 1. 🔑 Token Generation

* Send `POST` request with credentials
* دریافت access_token from response

### 2. 📡 API Call with Token

* Use `Bearer Token` in header
* Call challan তথ্য API using `GET`

### 3. 📄 JSON Parsing

* Use `JSON_TABLE` to convert JSON into relational rows

### 4. 💾 Data Insert

* Loop through JSON data
* Insert into `CHALAN_INFO` table

---

## 🧠 Key Concepts

| Concept                  | Description               |
| ------------------------ | ------------------------- |
| `APEX_WEB_SERVICE`       | Used to call REST APIs    |
| `APEX_JSON`              | Parse JSON response       |
| `JSON_TABLE`             | Convert JSON to SQL rows  |
| `AUTONOMOUS_TRANSACTION` | Allows independent commit |

---

## ⚠️ Important Notes

* Always clear headers using:

  ```sql
  apex_web_service.g_request_headers.delete;
  ```
* Ensure date format matches API response
* Avoid committing inside loop for large data (use bulk processing if needed)
* Handle exceptions properly in production

---

## 🚀 Improvements (Recommended)

* Add exception handling:

```sql
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
```

* Use bulk insert (`FORALL`) for performance
* Validate JSON before parsing

---

## 📂 Use Case

This procedure is useful for:

* Government payment verification systems
* External financial API integration
* Automated data synchronization

---

 ## Thank you
 ## Sanjay Sikder

You can connect with me on <a href="https://www.linkedin.com/in/sanjay-sikder/" target="_blank">LinkedIn</a>!

---




