# Getting Started with Db2

This document contains basic Db2 commands to help you create tables, insert mock data, query data, and explore your database. It also includes some Db2 environment configuration commands using `db2set`.

---

## 1. Connect to the database
```bash
db2 connect to BLUDB
```

## 2. Create a schema (optional)
```bash
db2 "CREATE SCHEMA TESTSCHEMA"
db2 "SET SCHEMA TESTSCHEMA"
```

## 3. Create a table
```bash
db2 "CREATE TABLE employees (
        emp_id INT NOT NULL PRIMARY KEY,
        first_name VARCHAR(50),
        last_name VARCHAR(50),
        hire_date DATE,
        salary DECIMAL(10,2)
     )"
```

## 4. Insert mock data
```bash
db2 "INSERT INTO employees VALUES
     (1, 'Alice', 'Green', '2021-05-10', 85000.00),
     (2, 'Bob', 'Khan', '2020-03-22', 96000.00),
     (3, 'Charlie', 'Lopez', '2019-11-05', 110000.00)"
```

## 5. Query the table
```bash
db2 "SELECT * FROM employees"
```

## 6. Describe table structure
```bash
db2 "DESCRIBE TABLE employees"
```

## 7. Add a new column
```bash
db2 "ALTER TABLE employees ADD COLUMN department VARCHAR(50)"
```

## 8. Update data
```bash
db2 "UPDATE employees SET department = 'Engineering' WHERE emp_id = 1"
```

## 9. Delete a row
```bash
db2 "DELETE FROM employees WHERE emp_id = 3"
```

## 10. Insert more mock data
```bash
db2 "INSERT INTO employees (emp_id, first_name, last_name, hire_date, salary, department) VALUES
     (4, 'Dana', 'White', '2022-01-15', 78000.00, 'Finance'),
     (5, 'Evan', 'Stone', '2023-02-03', 69000.00, 'HR')"
```

## 11. Query with conditions
```bash
db2 "SELECT emp_id, first_name, salary FROM employees WHERE salary > 90000"
```

## 12. Count rows
```bash
db2 "SELECT COUNT(*) FROM employees"
```

## 13. Order results
```bash
db2 "SELECT * FROM employees ORDER BY salary DESC"
```

## 14. Create an index
```bash
db2 "CREATE INDEX idx_emp_lastname ON employees(last_name)"
```

## 15. Drop table (cleanup)
```bash
db2 "DROP TABLE employees"
```

---

## 16. Db2 Environment Configuration with `db2set`
`db2set` is used to configure Db2 registry variables, which control the behavior of the Db2 instance or client. These settings can affect performance, logging, or SQL compatibility.

### Common `db2set` commands

**View all Db2 registry variables:**
```bash
db2set -all
```

**Set a registry variable for the current instance:**
```bash
db2set DB2_COMPATIBILITY_VECTOR=ORA
```
This enables Oracle compatibility mode, useful when migrating from Oracle to Db2.

**Unset a registry variable:**
```bash
db2set DB2_COMPATIBILITY_VECTOR=
```

**Save registry changes:**
```bash
db2stop
```
```bash
db2start
```
Some registry variable changes require stopping and restarting the instance.

### Use-cases for `db2set`
- **Enabling Oracle SQL compatibility**: Set `DB2_COMPATIBILITY_VECTOR=ORA` to allow certain Oracle SQL syntax.
- **Controlling logging behavior**: Variables like `DB2LOGPATH` can redirect logs to a specific directory.
- **Performance tuning**: Variables such as `DB2_NUM_IOCLEANERS` or `DB2_MAX_AGENT_CFG` help tune instance behavior.
- **Debugging or troubleshooting**: Enable trace or verbose output using registry variables.

### Example scenario
```bash
# Enable Oracle compatibility mode
db2set DB2_COMPATIBILITY_VECTOR=ORA

# Stop and start the instance to apply
db2stop
/db2start
```
This allows SQL scripts originally written for Oracle to run with minimal changes on Db2.

