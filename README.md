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
    d.value AS step_details,
    d.is_immediate
FROM projects p
JOIN sub_projects sp ON p.id = sp.project_id
JOIN dpr_data d ON sp.id = d.sub_project_id;
```

Query (example)

```SQL
SELECT * FROM vw_dpr_tracker WHERE project_name = 'Nashik Kumbhmela';
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
    w.value AS step_details,
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
SELECT 
    p.name AS project_name,
    sp.name AS sub_project_name,
    'DPR Phase' AS phase,
    d.heading AS task_name,
    d.value AS task_details
FROM projects p
JOIN sub_projects sp ON p.id = sp.project_id
JOIN dpr_data d ON sp.id = d.sub_project_id
WHERE d.is_immediate = 1

UNION ALL

SELECT 
    p.name AS project_name,
    sp.name AS sub_project_name,
    'Work Phase' AS phase,
    w.heading AS task_name,
    w.value AS task_details
FROM projects p
JOIN sub_projects sp ON p.id = sp.project_id
JOIN work_data w ON sp.id = w.sub_project_id
WHERE w.is_immediate = 1

UNION ALL

SELECT 
    p.name AS project_name,
    sp.name AS sub_project_name,
    'Monitoring Phase' AS phase,
    m.heading AS task_name,
    m.value AS task_details
FROM projects p
JOIN sub_projects sp ON p.id = sp.project_id
JOIN monitoring_data m ON sp.id = m.sub_project_id
WHERE m.is_immediate = 1;
```
Query 
```SQL
SELECT * FROM vw_urgent_tasks;
```
