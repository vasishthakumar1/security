# Oracle Privilege Analysis on Autonomous Database

## Introduction

This workshop introduces Oracle Privilege Analysis on Oracle Autonomous Database. It gives you an opportunity to capture privilege usage during a workload, generate Privilege Analysis results, and review used and unused privileges so you can identify grants that may no longer be needed.

*Estimated Lab Time:* 20 minutes

*Version tested in this lab:* Oracle Autonomous Database 26ai

### Objectives

In this lab, you will:

- Create sample users, tables, and grants for a privilege analysis scenario
- Capture workload activity in Autonomous Database
- Generate Privilege Analysis results
- Review used and unused privileges

### Prerequisites

This lab assumes you have:

- A Free Tier, Paid, or LiveLabs Oracle Cloud account
- An available Oracle Autonomous Database instance
- Access to Database Actions SQL Worksheet or SQL Developer
- ADMIN access, or access to a privileged user that can create users and grant `CAPTURE_ADMIN`

### Lab Timing (estimated)

| Task No. | Feature | Approx. Time |
|--|--|--|
| 1 | Prepare sample users and data | 5 minutes |
| 2 | Capture the workload to analyze | 5 minutes |
| 3 | Generate workload | 5 minutes |
| 4 | Analyze the workload captured | 5 minutes |
| 5 | Drop the capture | <5 minutes |

## Task 1: Prepare sample users and data

1. Open your Autonomous Database details page.

2. Click **Database Actions**, and then click **SQL**.

3. Connect as `ADMIN` or another privileged user.

4. Create the data owner, application users, and privilege analysis administrator.

    ```
    <copy>
    -- Create the data owner.
    CREATE USER hr_owner IDENTIFIED BY WElcome_123#;
    GRANT CREATE SESSION, CREATE TABLE TO hr_owner;
    GRANT UNLIMITED TABLESPACE TO hr_owner;

    -- Create application users.
    CREATE USER app_read IDENTIFIED BY WElcome_123#;
    CREATE USER app_write IDENTIFIED BY WElcome_123#;
    CREATE USER app_admin IDENTIFIED BY WElcome_123#;

    GRANT CREATE SESSION TO app_read, app_write, app_admin;

    -- Create the Privilege Analysis administrator.
    CREATE USER pa_admin IDENTIFIED BY WElcome_123#;
    GRANT CREATE SESSION TO pa_admin;
    GRANT CAPTURE_ADMIN TO pa_admin;

    -- Enable Database Actions access for the lab users.
    BEGIN
       ORDS_ADMIN.ENABLE_SCHEMA(
          p_enabled => TRUE,
          p_schema => UPPER('hr_owner'),
          p_url_mapping_type => 'BASE_PATH',
          p_url_mapping_pattern => LOWER('hr_owner'),
          p_auto_rest_auth => TRUE);

       ORDS_ADMIN.ENABLE_SCHEMA(
          p_enabled => TRUE,
          p_schema => UPPER('app_read'),
          p_url_mapping_type => 'BASE_PATH',
          p_url_mapping_pattern => LOWER('app_read'),
          p_auto_rest_auth => TRUE);

       ORDS_ADMIN.ENABLE_SCHEMA(
          p_enabled => TRUE,
          p_schema => UPPER('app_write'),
          p_url_mapping_type => 'BASE_PATH',
          p_url_mapping_pattern => LOWER('app_write'),
          p_auto_rest_auth => TRUE);

       ORDS_ADMIN.ENABLE_SCHEMA(
          p_enabled => TRUE,
          p_schema => UPPER('app_admin'),
          p_url_mapping_type => 'BASE_PATH',
          p_url_mapping_pattern => LOWER('app_admin'),
          p_auto_rest_auth => TRUE);

       ORDS_ADMIN.ENABLE_SCHEMA(
          p_enabled => TRUE,
          p_schema => UPPER('pa_admin'),
          p_url_mapping_type => 'BASE_PATH',
          p_url_mapping_pattern => LOWER('pa_admin'),
          p_auto_rest_auth => TRUE);
    END;
    /
    </copy>
    ```

5. Create the sample tables and data.

    ```
    <copy>
    CREATE TABLE hr_owner.employees (
       id NUMBER PRIMARY KEY,
       name VARCHAR2(50),
       department VARCHAR2(50),
       salary NUMBER
    );

    CREATE TABLE hr_owner.departments (
       dept_id NUMBER PRIMARY KEY,
       dept_name VARCHAR2(50)
    );

    INSERT INTO hr_owner.employees VALUES (1, 'Alice', 'HR', 5000);
    INSERT INTO hr_owner.employees VALUES (2, 'Bob', 'IT', 6000);
    INSERT INTO hr_owner.employees VALUES (3, 'Charlie', 'Finance', 7000);

    INSERT INTO hr_owner.departments VALUES (1, 'HR');
    INSERT INTO hr_owner.departments VALUES (2, 'IT');
    INSERT INTO hr_owner.departments VALUES (3, 'Finance');

    COMMIT;
    </copy>
    ```

6. Grant privileges to the application users.

    These grants intentionally include privileges that will not be used during the workload. This helps Privilege Analysis show both used and unused privileges.

    ```
    <copy>
    -- app_read can query employees and departments.
    GRANT SELECT ON hr_owner.employees TO app_read;
    GRANT SELECT ON hr_owner.departments TO app_read;

    -- Extra unused privilege for app_read.
    GRANT DELETE ON hr_owner.departments TO app_read;

    -- app_write can query, insert, and update employees.
    GRANT SELECT, INSERT, UPDATE ON hr_owner.employees TO app_write;

    -- app_admin can query, insert, update, and delete employees,
    -- and can query departments.
    GRANT SELECT, INSERT, UPDATE, DELETE ON hr_owner.employees TO app_admin;
    GRANT SELECT ON hr_owner.departments TO app_admin;
    </copy>
    ```

## Task 2: Capture the workload to analyze

1. Connect as `pa_admin`.

2. Create the privilege capture.

    ```
    <copy>
    BEGIN
       DBMS_PRIVILEGE_CAPTURE.CREATE_CAPTURE(
          name        => 'All Database Capture',
          description => 'Capture all privilege usage',
          type        => DBMS_PRIVILEGE_CAPTURE.G_DATABASE
       );
    END;
    /
    </copy>
    ```

3. Enable the capture.

    ```
    <copy>
    BEGIN
       DBMS_PRIVILEGE_CAPTURE.ENABLE_CAPTURE(
          name => 'All Database Capture'
       );
    END;
    /
    </copy>
    ```

4. Verify that the capture is enabled.

    ```
    <copy>
    SELECT name, type, enabled
    FROM dba_priv_captures
    WHERE name = 'All Database Capture';
    </copy>
    ```

    The expected result should show:

    ```
    ENABLED: Y
    ```

## Task 3: Generate workload

1. Connect as `app_read`, and then query the application tables.

    ```
    <copy>
    SELECT * FROM hr_owner.employees;
    SELECT name FROM hr_owner.employees WHERE id = 1;
    </copy>
    ```

    **Note:** The `DELETE` privilege granted to `app_read` is not used in this workload, so it should appear as unused after results are generated.

2. Connect as `app_write`, and then query, insert, and update employee data.

    ```
    <copy>
    SELECT * FROM hr_owner.employees;

    INSERT INTO hr_owner.employees
    VALUES (4, 'David', 'IT', 8000);

    UPDATE hr_owner.employees
    SET salary = 6500
    WHERE id = 2;

    COMMIT;
    </copy>
    ```

3. Connect as `app_admin`, and then query employee data, delete one employee row, and query departments.

    ```
    <copy>
    SELECT * FROM hr_owner.employees;

    DELETE FROM hr_owner.employees
    WHERE id = 3;

    SELECT * FROM hr_owner.departments;

    COMMIT;
    </copy>
    ```

4. Connect as `pa_admin`, and then disable the capture.

    ```
    <copy>
    BEGIN
       DBMS_PRIVILEGE_CAPTURE.DISABLE_CAPTURE(
          name => 'All Database Capture'
       );
    END;
    /
    </copy>
    ```

5. Verify that the capture is disabled.

    ```
    <copy>
    SELECT name, type, enabled
    FROM dba_priv_captures
    WHERE name = 'All Database Capture';
    </copy>
    ```

    The expected result should show:

    ```
    ENABLED: N
    ```

## Task 4: Analyze the workload captured

1. Connect as `pa_admin`.

2. Generate the result.

    ```
    <copy>
    BEGIN
       DBMS_PRIVILEGE_CAPTURE.GENERATE_RESULT(
          name => 'All Database Capture'
       );
    END;
    /
    </copy>
    ```

    **Note:**

    - This step compares privileges used during the capture with privileges granted to the users.
    - It may take a few minutes to generate depending on the amount of captured activity.

3. Review used and unused object privileges for `app_read`.

    ```
    <copy>
    -- Object privileges used by app_read.
    SELECT DISTINCT username, obj_priv, object_owner, object_name
    FROM dba_used_objprivs
    WHERE username = 'APP_READ'
      AND object_owner = 'HR_OWNER'
      AND object_name IN ('EMPLOYEES', 'DEPARTMENTS')
    ORDER BY object_name, obj_priv;

    -- Object privileges granted to app_read but not used.
    SELECT DISTINCT username, obj_priv, object_owner, object_name
    FROM dba_unused_objprivs
    WHERE username = 'APP_READ'
      AND object_owner = 'HR_OWNER'
      AND object_name IN ('EMPLOYEES', 'DEPARTMENTS')
    ORDER BY object_name, obj_priv;
    </copy>
    ```

4. Review used and unused object privileges for `app_write`.

    ```
    <copy>
    -- Object privileges used by app_write.
    SELECT DISTINCT username, obj_priv, object_owner, object_name
    FROM dba_used_objprivs
    WHERE username = 'APP_WRITE'
      AND object_owner = 'HR_OWNER'
      AND object_name IN ('EMPLOYEES', 'DEPARTMENTS')
    ORDER BY object_name, obj_priv;

    -- Object privileges granted to app_write but not used.
    SELECT DISTINCT username, obj_priv, object_owner, object_name
    FROM dba_unused_objprivs
    WHERE username = 'APP_WRITE'
      AND object_owner = 'HR_OWNER'
      AND object_name IN ('EMPLOYEES', 'DEPARTMENTS')
    ORDER BY object_name, obj_priv;
    </copy>
    ```

5. Review used and unused object privileges for `app_admin`.

    ```
    <copy>
    -- Object privileges used by app_admin.
    SELECT DISTINCT username, obj_priv, object_owner, object_name
    FROM dba_used_objprivs
    WHERE username = 'APP_ADMIN'
      AND object_owner = 'HR_OWNER'
      AND object_name IN ('EMPLOYEES', 'DEPARTMENTS')
    ORDER BY object_name, obj_priv;

    -- Object privileges granted to app_admin but not used.
    SELECT DISTINCT username, obj_priv, object_owner, object_name
    FROM dba_unused_objprivs
    WHERE username = 'APP_ADMIN'
      AND object_owner = 'HR_OWNER'
      AND object_name IN ('EMPLOYEES', 'DEPARTMENTS')
    ORDER BY object_name, obj_priv;
    </copy>
    ```

    **Note:**

    - You can see which object privileges were used and unused by each application user during the capture.
    - This helps determine whether users have privileges they do not need.
    - Unused privileges are good candidates for further review before revocation.

## Task 5: Clean up the lab

Once you have reviewed the results, drop the privilege capture and optionally remove the sample users and data.

1. Connect as `pa_admin`.

2. Drop the privilege capture.

    ```
    <copy>
    BEGIN
       DBMS_PRIVILEGE_CAPTURE.DROP_CAPTURE(
          name => 'All Database Capture'
       );
    END;
    /
    </copy>
    ```

3. If you want to remove the sample users and data after the lab, connect as `ADMIN` or another privileged user and run:

    ```
    <copy>
    DROP USER app_read CASCADE;
    DROP USER app_write CASCADE;
    DROP USER app_admin CASCADE;
    DROP USER pa_admin CASCADE;
    DROP USER hr_owner CASCADE;
    </copy>
    ```

You may now proceed to the next lab.

## Appendix: About the Product

### Overview

Privilege Analysis increases the security of applications and database operations by helping you implement least privilege best practices for database roles and privileges.

Running inside the Oracle Database kernel, Privilege Analysis helps reduce the attack surface of user, tool, and application accounts by identifying used and unused privileges. You can use this information to implement a least privilege model where users are granted only the privileges needed to perform their work.

Privilege Analysis captures privileges used by database users and applications at runtime and writes its findings to data dictionary views that you can query. The results help you identify privileges that were exercised during a workload and privileges that were granted but not used.

### Benefits of Using Privilege Analysis

- Find unnecessarily granted privileges
- Implement least privilege best practices
- Reduce the risk from compromised or overprivileged accounts
- Support separation of duties
- Review privilege usage during application development and testing

## Want to Learn More?

Technical Documentation:

- [Oracle Privilege Analysis Release 23](https://docs.oracle.com/en/database/oracle/oracle-database/23/dbseg/performing-privilege-analysis-identify-privilege-use.html#GUID-44CB644B-7B59-4B3B-B375-9F9B96F60186)

## Acknowledgements

- **Author** - Vasishtha Kumar, Database Security PM
