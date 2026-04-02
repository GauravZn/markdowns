# 1. vw_executive_dashboard
**Displays:** Project names, Sub-Project names, and the overall status of DPR, Work, and Monitoring.

```SQL
CREATE OR REPLACE VIEW vw_executive_dashboard AS
SELECT 
    p.name AS project_name,
    p.serial_no AS project_serial,
    sp.name AS sub_project_name,
    sp.dpr_status,
    sp.work_status,
    sp.monitoring_status
FROM projects p
JOIN sub_projects sp ON p.id = sp.project_id;
```
Query

```SQL

SELECT * FROM vw_executive_dashboard;
```



# 2. vw_dpr_tracker
**Displays:** Only the steps related to the DPR phase, linked to their specific projects.

```SQL
CREATE OR REPLACE VIEW vw_dpr_tracker AS
SELECT 
    p.name AS project_name,
    sp.name AS sub_project_name,
    sp.dpr_status AS overall_dpr_status,
    d.heading AS dpr_step,
    -- REMARKS: Cleaned status/notes
    CASE 
        WHEN d.value LIKE '%{%' THEN COALESCE((d.value::json)->>'status', '')
        WHEN d.value ~ '^[0-9]{4}-[0-9]{2}-[0-9]{2}$' THEN 'Recorded'
        WHEN d.value LIKE '%|%' THEN 
            TRIM(BOTH ' , ' FROM 
                REGEXP_REPLACE(
                    REGEXP_REPLACE(
                        REPLACE(
                            REGEXP_REPLACE(
                                REGEXP_REPLACE(d.value, '(?i)(?:Rs\.?[ \t]*)?[0-9]+(?:\.[0-9]+)?[ \t]*(?:Crore|Lakh|Cr)', '', 'g'), 
                            '[0-9]{4}-[0-9]{2}-[0-9]{2}', '', 'g'),
                        '|', ','),
                    ',[ \t]*,', ',', 'g'), 
                '[ \t]+', ' ', 'g')
            )
        ELSE COALESCE(TRIM(d.value), '') 
    END AS remarks,
    -- AMOUNT: Financial extraction
    CASE 
        WHEN d.value LIKE '%{%' THEN COALESCE((d.value::json)->'data'->>'amount', '')
        WHEN d.value LIKE '%|%' THEN COALESCE(SUBSTRING(d.value FROM '(?i)(?:Rs\.?[ \t]*)?[0-9]+(?:\.[0-9]+)?[ \t]*(?:Crore|Lakh|Cr)'), '')
        ELSE ''
    END AS amount,
    CASE WHEN d.value LIKE '%{%' THEN COALESCE((d.value::json)->'data'->>'amountUnit', '') ELSE '' END AS amount_unit,
    -- STEP_DATE: Localized Indian Format (DD-MM-YYYY)
    CASE 
        WHEN d.value LIKE '%{%' AND NULLIF(TRIM((d.value::json)->'data'->>'date'), '') IS NOT NULL 
            THEN TO_CHAR(((d.value::json)->'data'->>'date')::DATE, 'DD-MM-YYYY')
        WHEN SUBSTRING(d.value FROM '[0-9]{4}-[0-9]{2}-[0-9]{2}') IS NOT NULL 
            THEN TO_CHAR((SUBSTRING(d.value FROM '[0-9]{4}-[0-9]{2}-[0-9]{2}'))::DATE, 'DD-MM-YYYY')
        ELSE ''
    END AS step_date,
    d.is_immediate
FROM projects p
JOIN sub_projects sp ON p.id = sp.project_id
JOIN dpr_data d ON sp.id = d.sub_project_id;
```

Query (example)

```SQL
SELECT * FROM vw_dpr_tracker;
```



# 3. vw_work_execution
**Displays:** Only the physical execution steps (Work Data) linked to the correct sub-project.

```SQL
CREATE OR REPLACE VIEW vw_work_execution AS
SELECT 
    p.name AS project_name,
    sp.name AS sub_project_name,
    sp.work_status AS overall_work_status,
    w.heading AS work_step,
    CASE 
        WHEN w.value LIKE '%{%' THEN COALESCE((w.value::json)->>'status', '')
        WHEN w.value ~ '^[0-9]{4}-[0-9]{2}-[0-9]{2}$' THEN 'Recorded'
        WHEN w.value LIKE '%|%' THEN 
            TRIM(BOTH ' , ' FROM REGEXP_REPLACE(REGEXP_REPLACE(REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(w.value, '(?i)(?:Rs\.?[ \t]*)?[0-9]+(?:\.[0-9]+)?[ \t]*(?:Crore|Lakh|Cr)', '', 'g'), '[0-9]{4}-[0-9]{2}-[0-9]{2}', '', 'g'), '|', ','), ',[ \t]*,', ',', 'g'), '[ \t]+', ' ', 'g'))
        ELSE COALESCE(TRIM(w.value), '') 
    END AS remarks,
    CASE WHEN w.value LIKE '%{%' THEN COALESCE((w.value::json)->'data'->>'amount', '') WHEN w.value LIKE '%|%' THEN COALESCE(SUBSTRING(w.value FROM '(?i)(?:Rs\.?[ \t]*)?[0-9]+(?:\.[0-9]+)?[ \t]*(?:Crore|Lakh|Cr)'), '') ELSE '' END AS amount,
    CASE WHEN w.value LIKE '%{%' THEN COALESCE((w.value::json)->'data'->>'amountUnit', '') ELSE '' END AS amount_unit,
    CASE WHEN w.value LIKE '%{%' AND NULLIF(TRIM((w.value::json)->'data'->>'date'), '') IS NOT NULL THEN TO_CHAR(((w.value::json)->'data'->>'date')::DATE, 'DD-MM-YYYY') WHEN SUBSTRING(w.value FROM '[0-9]{4}-[0-9]{2}-[0-9]{2}') IS NOT NULL THEN TO_CHAR((SUBSTRING(w.value FROM '[0-9]{4}-[0-9]{2}-[0-9]{2}'))::DATE, 'DD-MM-YYYY') ELSE '' END AS step_date,
    w.is_immediate
FROM projects p
JOIN sub_projects sp ON p.id = sp.project_id
JOIN work_data w ON sp.id = w.sub_project_id;
```
Query
```SQL
SELECT * FROM vw_work_execution;
```


# 4. vw_urgent_tasks
**Displays:** Merges DPR, Work, Monitoring together, and shows  ONLY the tasks flagged with is_immediate = 1.

```SQL

CREATE OR REPLACE VIEW vw_urgent_tasks AS

-- SECTION 1: DPR DATA
SELECT 
    p.name AS project_name, 
    sp.name AS sub_project_name, 
    'DPR' AS phase, 
    d.heading AS task,
    -- REMARKS COLUMN
    CASE 
        WHEN d.value LIKE '%{%' THEN COALESCE((d.value::json)->>'status', '') 
        WHEN d.value ~ '^[0-9]{4}-[0-9]{2}-[0-9]{2}$' THEN 'Recorded' 
        ELSE TRIM(BOTH ' , ' FROM REGEXP_REPLACE(REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(d.value, '(?i)(?:Rs\.?[ \t]*)?[0-9]+(?:\.[0-9]+)?[ \t]*(?:Crore|Lakh|Cr)', '', 'g'), '[0-9]{4}-[0-9]{2}-[0-9]{2}', '', 'g'), '|', ','), '([ \t]*,[ \t]*)+', ', ', 'g')) 
    END AS remarks,
    -- AMOUNT COLUMN
    CASE 
        WHEN d.value LIKE '%{%' THEN COALESCE((d.value::json)->'data'->>'amount', '') 
        WHEN d.value LIKE '%|%' THEN COALESCE(SUBSTRING(d.value FROM '(?i)(?:Rs\.?[ \t]*)?[0-9]+(?:\.[0-9]+)?[ \t]*(?:Crore|Lakh|Cr)'), '') 
        ELSE '' 
    END AS amount,
    -- DATE COLUMN (Indian Format)
    CASE 
        WHEN d.value LIKE '%{%' AND NULLIF(TRIM((d.value::json)->'data'->>'date'), '') IS NOT NULL THEN TO_CHAR(((d.value::json)->'data'->>'date')::DATE, 'DD-MM-YYYY') 
        WHEN SUBSTRING(d.value FROM '[0-9]{4}-[0-9]{2}-[0-9]{2}') IS NOT NULL THEN TO_CHAR((SUBSTRING(d.value FROM '[0-9]{4}-[0-9]{2}-[0-9]{2}'))::DATE, 'DD-MM-YYYY') 
        ELSE '' 
    END AS step_date
FROM projects p 
JOIN sub_projects sp ON p.id = sp.project_id 
JOIN dpr_data d ON sp.id = d.sub_project_id 
WHERE d.is_immediate = 1

UNION ALL

-- SECTION 2: WORK DATA
SELECT 
    p.name AS project_name, 
    sp.name AS sub_project_name, 
    'Work' AS phase, 
    w.heading AS task,
    -- REMARKS COLUMN (Must match Section 1)
    CASE 
        WHEN w.value LIKE '%{%' THEN COALESCE((w.value::json)->>'status', '') 
        WHEN w.value ~ '^[0-9]{4}-[0-9]{2}-[0-9]{2}$' THEN 'Recorded' 
        ELSE TRIM(BOTH ' , ' FROM REGEXP_REPLACE(REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(w.value, '(?i)(?:Rs\.?[ \t]*)?[0-9]+(?:\.[0-9]+)?[ \t]*(?:Crore|Lakh|Cr)', '', 'g'), '[0-9]{4}-[0-9]{2}-[0-9]{2}', '', 'g'), '|', ','), '([ \t]*,[ \t]*)+', ', ', 'g')) 
    END AS remarks,
    -- AMOUNT COLUMN (Must match Section 1)
    CASE 
        WHEN w.value LIKE '%{%' THEN COALESCE((w.value::json)->'data'->>'amount', '') 
        WHEN w.value LIKE '%|%' THEN COALESCE(SUBSTRING(w.value FROM '(?i)(?:Rs\.?[ \t]*)?[0-9]+(?:\.[0-9]+)?[ \t]*(?:Crore|Lakh|Cr)'), '') 
        ELSE '' 
    END AS amount,
    -- DATE COLUMN (Must match Section 1)
    CASE 
        WHEN w.value LIKE '%{%' AND NULLIF(TRIM((w.value::json)->'data'->>'date'), '') IS NOT NULL THEN TO_CHAR(((w.value::json)->'data'->>'date')::DATE, 'DD-MM-YYYY') 
        WHEN SUBSTRING(w.value FROM '[0-9]{4}-[0-9]{2}-[0-9]{2}') IS NOT NULL THEN TO_CHAR((SUBSTRING(w.value FROM '[0-9]{4}-[0-9]{2}-[0-9]{2}'))::DATE, 'DD-MM-YYYY') 
        ELSE '' 
    END AS step_date
FROM projects p 
JOIN sub_projects sp ON p.id = sp.project_id 
JOIN work_data w ON sp.id = w.sub_project_id 
WHERE w.is_immediate = 1

UNION ALL

-- SECTION 3: MONITORING DATA
SELECT 
    p.name AS project_name, 
    sp.name AS sub_project_name, 
    'Monitoring' AS phase, 
    m.heading AS task,
    -- REMARKS COLUMN (Must match Section 1)
    CASE 
        WHEN m.value LIKE '%{%' THEN COALESCE((m.value::json)->>'status', '') 
        WHEN m.value ~ '^[0-9]{4}-[0-9]{2}-[0-9]{2}$' THEN 'Recorded' 
        ELSE TRIM(BOTH ' , ' FROM REGEXP_REPLACE(REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(m.value, '(?i)(?:Rs\.?[ \t]*)?[0-9]+(?:\.[0-9]+)?[ \t]*(?:Crore|Lakh|Cr)', '', 'g'), '[0-9]{4}-[0-9]{2}-[0-9]{2}', '', 'g'), '|', ','), '([ \t]*,[ \t]*)+', ', ', 'g')) 
    END AS remarks,
    -- AMOUNT COLUMN (Must match Section 1)
    CASE 
        WHEN m.value LIKE '%{%' THEN COALESCE((m.value::json)->'data'->>'amount', '') 
        WHEN m.value LIKE '%|%' THEN COALESCE(SUBSTRING(m.value FROM '(?i)(?:Rs\.?[ \t]*)?[0-9]+(?:\.[0-9]+)?[ \t]*(?:Crore|Lakh|Cr)'), '') 
        ELSE '' 
    END AS amount,
    -- DATE COLUMN (Must match Section 1)
    CASE 
        WHEN m.value LIKE '%{%' AND NULLIF(TRIM((m.value::json)->'data'->>'date'), '') IS NOT NULL THEN TO_CHAR(((m.value::json)->'data'->>'date')::DATE, 'DD-MM-YYYY') 
        WHEN SUBSTRING(m.value FROM '[0-9]{4}-[0-9]{2}-[0-9]{2}') IS NOT NULL THEN TO_CHAR((SUBSTRING(m.value FROM '[0-9]{4}-[0-9]{2}-[0-9]{2}'))::DATE, 'DD-MM-YYYY') 
        ELSE '' 
    END AS step_date
FROM projects p 
JOIN sub_projects sp ON p.id = sp.project_id 
JOIN monitoring_data m ON sp.id = m.sub_project_id 
WHERE m.is_immediate = 1;

```
Query 
```SQL
SELECT * FROM vw_urgent_tasks;
```
