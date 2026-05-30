# MCP Server - Oracle DB Context

[English](README.md)

A powerful Model Context Protocol (MCP) server that provides contextual database schema information for large Oracle databases, enabling AI assistants to understand and work with databases containing thousands of tables.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Usage](#usage)
  - [Integration with GitHub Copilot in VSCode Insiders](#integration-with-github-copilot-in-vscode-insiders)
    - [Option 1: Using Docker (Recommended)](#option-1-using-docker-recommended)
    - [Option 2: Using UV (Local Installation)](#option-2-using-uv-local-installation)
  - [Starting the Server locally](#starting-the-server-locally)
  - [Streamable HTTP Server Mode](#streamable-http-server-mode)
    - [Quick Start with Docker Compose](#quick-start-with-docker-compose)
    - [Manual Docker Run](#manual-docker-run)
    - [Connecting LLM Clients](#connecting-llm-clients)
    - [Authentication](#authentication)
    - [Health Check](#health-check)
  - [Available Tools](#available-tools)
- [Architecture](#architecture)
  - [Transport Layer](#transport-layer)
  - [Application Layers](#application-layers)
  - [Schema Caching Strategy](#schema-caching-strategy)
  - [Request Flow](#request-flow)
- [Connection Modes](#connection-modes)
  - [Thin Mode (Default)](#thin-mode-default)
  - [Thick Mode](#thick-mode)
- [Test Environment](#test-environment)
  - [Oracle Test Database](#oracle-test-database)
  - [Test Schema Structure](#test-schema-structure)
  - [Running the Test Database](#running-the-test-database)
  - [Test Queries](#test-queries)
- [System Requirements](#system-requirements)
- [Performance Considerations](#performance-considerations)
- [Contributing](#contributing)
- [License](#license)
- [Support](#support)

## Overview

The MCP Oracle DB Context server solves a critical challenge when working with very large Oracle databases: how to provide AI models with accurate, relevant database schema information without overwhelming them with tens of thousands of tables and relationships.

By intelligently caching and serving database schema information, this server allows AI assistants to:

- Look up specific table schemas on demand
- Search for tables that match specific patterns
- Understand table relationships and foreign keys
- Get database vendor information

## Features

- **Smart Schema Caching**: Builds and maintains a local cache of your database schema to minimize database queries
- **Targeted Schema Lookup**: Retrieve schema for specific tables without loading the entire database structure
- **Table Search**: Find tables by name pattern matching
- **Relationship Mapping**: Understand foreign key relationships between tables
- **Oracle Database Support**: Built specifically for Oracle databases
- **Dual Transport Modes**: Run as a local subprocess over stdio (for desktop AI clients) or as a persistent HTTP server (streamable-http) consumable by any MCP-compatible LLM client over the network
- **API Key Authentication**: Optional Bearer token protection for the HTTP server mode
- **MCP Integration**: Works seamlessly with GitHub Copilot in VSCode, Claude, ChatGPT, and other AI assistants that support MCP

## Usage

### Integration with GitHub Copilot in VSCode Insiders

To use this MCP server with GitHub Copilot in VSCode Insiders, follow these steps:

1. **Install VSCode Insiders**
   - Download and install the latest version of [VSCode Insiders](https://code.visualstudio.com/insiders/)

2. **Install GitHub Copilot Extension**
   - Open VSCode Insiders
   - Go to the Extensions marketplace
   - Search for and install "GitHub Copilot"

3. **Configure MCP Server**
   - **Recommended: [Using Docker](#option-1-using-docker-recommended)**
   - Alternative: [Using UV](#option-2-using-uv-local-installation)

4. **Enable Agent Mode**
   - Open Copilot chat in VSCode Insiders
   - Click on "Copilot Edits"
   - Choose "Agent mode"
   - Click the refresh button in the chat input to load the available tools

After completing these steps, you'll have access to all database context tools through GitHub Copilot's chat interface.

#### Option 1: Using Docker (Recommended)

In VSCode Insiders, go to your user or workspace `settings.json` file and add the following:

```json
"mcp": {
    "inputs": [
     {
       "id": "db-password",
       "type": "promptString",
       "description": "Oracle DB Password",
       "password": true,
     }
   ],
    "servers": {
        "oracle": {
            "command": "docker",
            "args": [
                "run",
                "-i",
                "--rm",
                "-e",
                "ORACLE_CONNECTION_STRING",
                "-e",
                "TARGET_SCHEMA",
                "-e",
                "CACHE_DIR",
                "-e",
                "THICK_MODE",
                "deivisutp/oracle-mcp-server"
            ],
            "env": {
               "ORACLE_CONNECTION_STRING":"<db-username>/${input:db-password}@<host>:1521/<service-name>",
               "TARGET_SCHEMA":"",
               "CACHE_DIR":".cache",
               "THICK_MODE":"",  // Optional: set to "1" to enable thick mode
               "ORACLE_CLIENT_LIB_DIR":"" // Optional: in case you use thick mode and you want to set a non-default directory for client libraries
            }
        }
    }
}
```

When using Docker (recommended approach):

- All dependencies are included in the container
- Set `THICK_MODE=1` in the environment variables to enable thick mode if needed
- If you use `THICK_MODE`, you can optionally set the path where Oracle Client libraries are installed with `ORACLE_CLIENT_LIB_DIR` if it differs from the default location.

#### Option 2: Using UV (Local Installation)

This option requires installing and setting up the project locally:

1.  **Prerequisites**
    - Python 3.12 or higher
    - Oracle database access
    - Oracle instant client (required for the `oracledb` Python package)

2.  **Install UV**

    ```bash
    # Install uv using curl (macOS/Linux)
    curl -LsSf https://astral.sh/uv/install.sh | sh

    # Or using PowerShell (Windows)
    irm https://astral.sh/uv/install.ps1 | iex
    ```

    Make sure to restart your terminal after installing uv.

3.  **Project Setup**

    ```bash
    # Clone repository
    git clone https://github.com/yourusername/oracle-mcp-server.git
    cd oracle-mcp-server

    # Create and activate virtual environment
    uv venv

    # Activate (On Unix/macOS)
    source .venv/bin/activate

    # Activate (On Windows)
    .venv\Scripts\activate

    # Install dependencies
    uv pip install -e .
    ```

4.  **Configure VSCode Settings**
    ```json
    "mcp": {
       "inputs": [
          {
             "id": "db-password",
             "type": "promptString",
             "description": "Oracle DB Password",
             "password": true,
          }
       ],
       "servers": {
          "oracle": {
                "command": "/path/to/your/.local/bin/uv",
                "args": [
                   "--directory",
                   "/path/to/your/oracle-mcp-server",
                   "run",
                   "main.py"
                ],
                "env": {
                   "ORACLE_CONNECTION_STRING":"<db-username>/${input:db-password}@<host>:1521/<service-name>",
                   "TARGET_SCHEMA":"",
                   "CACHE_DIR":".cache",
                   "THICK_MODE":"",  // Optional: set to "1" to enable thick mode
                   "ORACLE_CLIENT_LIB_DIR":"" // Optional: in case you use thick mode and if you want to set a non-default directory for client libraries
                }
          }
       }
    }
    ```

- Replace the paths with your actual uv binary path and oracle-mcp-server directory path

For both options:

- Replace the `ORACLE_CONNECTION_STRING` with your actual database connection string
- The `TARGET_SCHEMA` is optional, it will default to the user's schema
- The `CACHE_DIR` is optional, defaulting to `.cache` within the MCP server root folder

### Starting the Server locally

To run the MCP server directly:

```bash
uv run main.py
```

For development and testing:

```bash
# Install the MCP Inspector
uv pip install mcp-cli

# Test with MCP Inspector
mcp dev main.py

# Or install in Claude Desktop
mcp install main.py
```

### Streamable HTTP Server Mode

In addition to the standard stdio subprocess mode, the server supports the **MCP Streamable HTTP transport** (spec `2025-03-26`). This lets you run the MCP server as a long-lived HTTP service that any MCP-compatible client can reach over the network — no subprocess required.

The same Docker image handles both modes. The `MCP_TRANSPORT` environment variable selects the transport at startup.

| Variable        | Default   | Description                                   |
| --------------- | --------- | --------------------------------------------- |
| `MCP_TRANSPORT` | `stdio`   | Set to `streamable-http` to enable HTTP mode  |
| `MCP_PORT`      | `8000`    | Port the HTTP server listens on               |
| `MCP_HOST`      | `0.0.0.0` | Bind address (keep as-is inside containers)   |
| `MCP_API_KEY`   | _(unset)_ | When set, enables Bearer token authentication |

#### Quick Start with Docker Compose

Copy the provided `docker-compose.yml` to your server, set the required environment variables, and start:

```bash
export ORACLE_CONNECTION_STRING="user/pass@host:1521/service"
export MCP_API_KEY="your-secret-key"   # optional but recommended
docker compose up -d
```

The service will be healthy (curl `/health` returns 200) once the Oracle schema cache has been built. This takes 30–90 seconds on first launch, depending on database size. The cache is persisted in a named Docker volume so subsequent restarts are fast.

#### Manual Docker Run

```bash
docker run -d \
  -e MCP_TRANSPORT=streamable-http \
  -e MCP_PORT=8000 \
  -e MCP_API_KEY=your-secret-key \
  -e ORACLE_CONNECTION_STRING="user/pass@host:1521/service" \
  -e TARGET_SCHEMA=MY_SCHEMA \
  -p 8000:8000 \
  --restart unless-stopped \
  deivisutp/oracle-mcp-server
```

#### Connecting LLM Clients

The MCP endpoint is available at:

```
http://<server>:8000/mcp
```

Example configuration for clients that support MCP Streamable HTTP:

```json
{
  "mcpServers": {
    "oracle": {
      "url": "http://your-server:8000/mcp",
      "headers": {
        "Authorization": "Bearer your-secret-key"
      }
    }
  }
}
```

#### Authentication

When `MCP_API_KEY` is set, every request to `/mcp` must include the header:

```
Authorization: Bearer <MCP_API_KEY>
```

Requests without a valid key receive `401 Unauthorized`. The `/health` endpoint is always unauthenticated.

If `MCP_API_KEY` is not set the server runs without authentication — suitable for private/trusted networks only.

#### Health Check

```bash
curl http://your-server:8000/health
# → {"status":"ok"}
```

The health endpoint is unauthenticated and is used by Docker / load-balancers to verify the service is ready.

### Available Tools

When connected to an AI assistant like GitHub Copilot in VSCode Insiders or Claude, the following tools will be available:

#### `get_table_schema`

Get detailed schema information for a specific table including columns, data types, nullability, and relationships.
Example:

```
Can you show me the schema for the EMPLOYEES table?
```

#### `get_tables_schema`

Get schema information for multiple tables at once. More efficient than calling get_table_schema multiple times.
Example:

```
Please provide the schemas for both EMPLOYEES and DEPARTMENTS tables.
```

#### `search_tables_schema`

Search for tables by name pattern and retrieve their schemas.
Example:

```
Find all tables that might be related to customers and show their schemas.
```

#### `rebuild_schema_cache`

Force a rebuild of the schema cache. Use sparingly as this is resource-intensive.
Example:

```
The database structure has changed. Could you rebuild the schema cache?
```

#### `get_database_vendor_info`

Get information about the connected Oracle database version and schema.
Example:

```
What Oracle database version are we running?
```

#### `search_columns`

Search for tables containing columns that match a specific term. Useful when you know what data you need but aren't sure which tables contain it.
Example:

```
Which tables have columns related to customer_id?
```

#### `get_pl_sql_objects`

Get information about PL/SQL objects like procedures, functions, packages, triggers, etc.
Example:

```
Show me all stored procedures that start with 'CUSTOMER_'
```

#### `get_object_source`

Retrieve the source code for a PL/SQL object. Useful for debugging and understanding database logic.
Example:

```
Can you show me the source code for the CUSTOMER_UPDATE_PROC procedure?
```

#### `get_table_constraints`

Get all constraints (primary keys, foreign keys, unique constraints, check constraints) for a table.
Example:

```
What constraints are defined on the ORDERS table?
```

#### `get_table_indexes`

Get all indexes defined on a table, helpful for query optimization.
Example:

```
Show me all indexes on the CUSTOMERS table.
```

#### `get_dependent_objects`

Find all objects that depend on a specified database object.
Example:

```
What objects depend on the CUSTOMER_VIEW view?
```

#### `get_user_defined_types`

Get information about user-defined types in the database.
Example:

```
Show me all custom types defined in the schema.
```

#### `get_related_tables`

Get all tables that are related to a specified table through foreign keys, showing both incoming and outgoing relationships.
Example:

```
What tables are related to the ORDERS table?
```

#### `execute_select_query`

Execute a SELECT SQL query and return the results as a formatted table. Only queries starting with SELECT are allowed.
Example:

```
Run: SELECT * FROM EMPLOYEES WHERE DEPARTMENT_ID = 10;
```

#### `execute_update_query`

Execute an UPDATE SQL query and return the number of affected rows. Only queries starting with UPDATE are allowed.
Example:

```
Update the salary for all employees in department 10: UPDATE EMPLOYEES SET SALARY = SALARY * 1.1 WHERE DEPARTMENT_ID = 10;
```

#### `execute_delete_query`

Execute a DELETE SQL query and return the number of affected rows. Only queries starting with DELETE are allowed.
Example:

```
Delete all employees in department 10: DELETE FROM EMPLOYEES WHERE DEPARTMENT_ID = 10;
```

#### `execute_plsql_ddl`

Execute PL/SQL DDL code to create or replace database objects like procedures, functions, packages, triggers, etc. This tool is essential for creating stored procedures and other PL/SQL objects in the Oracle database.

**Supported PL/SQL Object Types:**

- Procedures (CREATE OR REPLACE PROCEDURE)
- Functions (CREATE OR REPLACE FUNCTION)
- Packages (CREATE OR REPLACE PACKAGE)
- Package Bodies (CREATE OR REPLACE PACKAGE BODY)
- Triggers (CREATE OR REPLACE TRIGGER)
- Types (CREATE OR REPLACE TYPE)
- Type Bodies (CREATE OR REPLACE TYPE BODY)

The tool automatically checks for compilation errors after execution and reports them if any occur. All DDL operations are automatically committed to the database.

Example:

```
Create a new procedure:
CREATE OR REPLACE PROCEDURE update_employee_salary(
    p_employee_id NUMBER,
    p_new_salary NUMBER
) IS
BEGIN
    UPDATE employees
    SET salary = p_new_salary
    WHERE employee_id = p_employee_id;
    COMMIT;
END;
```

#### `execute_plsql_call`

Execute a PL/SQL procedure or function call. This tool allows you to run existing stored procedures, functions, or other executable PL/SQL code blocks in the Oracle database.

**Supported Call Formats:**

- **EXEC format**: `EXEC procedure_name('parameter')`
- **Anonymous PL/SQL blocks**: `BEGIN procedure_name('parameter'); END;`
- **DECLARE blocks**: `DECLARE var VARCHAR2(10); BEGIN procedure_name(var); END;`
- **Direct procedure calls**: `procedure_name('parameter')`

All formats are automatically converted to proper PL/SQL anonymous blocks for execution. The tool handles parameter passing and automatically commits the transaction after execution. Trailing '/' characters (common in SQL\*Plus scripts) are automatically removed.

Examples:

```
Execute a procedure using EXEC format:
EXEC update_employee_salary(123, 75000);

Execute using a DECLARE block:
DECLARE
    v_emp_id NUMBER := 123;
    v_new_salary NUMBER := 80000;
BEGIN
    update_employee_salary(v_emp_id, v_new_salary);
    DBMS_OUTPUT.PUT_LINE('Salary updated successfully');
END;

Execute using BEGIN...END format:
BEGIN
    update_employee_salary(456, 85000);
END;
```

## Architecture

### Transport Layer

The server supports two MCP transports, selected at runtime via the `MCP_TRANSPORT` environment variable:

```
┌─────────────────────────────────────────────────────────────┐
│                        MCP Clients                          │
│  GitHub Copilot · Claude Desktop · Custom LLM applications  │
└────────────────────┬──────────────────┬─────────────────────┘
                     │                  │
           stdio subprocess        HTTP network
           (local, default)    (streamable-http)
                     │                  │
         ┌───────────▼──────────────────▼────────────┐
         │              FastMCP Framework              │
         │  ┌─────────────────────────────────────┐  │
         │  │  Starlette app  (HTTP mode only)    │  │
         │  │  ├─ GET/POST /mcp  (MCP protocol)  │  │
         │  │  ├─ GET /health   (unauthenticated) │  │
         │  │  └─ _APIKeyMiddleware (Bearer auth) │  │
         │  └─────────────────────────────────────┘  │
         │                                            │
         │           MCP Tool Handlers (13 tools)     │
         └────────────────────┬───────────────────────┘
                              │
         ┌────────────────────▼───────────────────────┐
         │              DatabaseContext               │
         └──────────────┬─────────────┬───────────────┘
                        │             │
          ┌─────────────▼──┐   ┌──────▼──────────────┐
          │ DatabaseConnector│   │   SchemaManager     │
          │ (oracledb)       │   │ (cache + indexing)  │
          └─────────────┬───┘   └──────┬──────────────┘
                        │              │
         ┌──────────────▼──────────────▼──────────────┐
         │              Oracle Database                │
         └─────────────────────────────────────────────┘
```

**stdio mode** is the standard way to integrate with desktop AI clients (Claude Desktop, GitHub Copilot in VSCode). The client launches the server as a child process and communicates over stdin/stdout. No network port is opened.

**Streamable HTTP mode** (`MCP_TRANSPORT=streamable-http`) runs the server as a persistent HTTP service using uvicorn. The MCP protocol is tunnelled over HTTP POST/GET on the `/mcp` path. This mode is designed for:

- Shared team servers where multiple developers or services consume the same MCP instance
- Cloud/container deployments where you cannot manage a local subprocess
- LLM platforms and CI pipelines that call MCP via HTTP

### Application Layers

The server is built with three distinct layers:

**1. DatabaseConnector** (`db_context/database.py`)

Wraps the `oracledb` driver and handles all Oracle-specific concerns:

- Connection pooling (sync or async depending on thick/thin mode)
- Raw SQL execution and cursor management
- All metadata queries (`all_tables`, `all_columns`, `all_constraints`, `all_indexes`, `all_dependencies`, `all_source`, `dbms_metadata`)
- DML/DDL execution with automatic commit
- PL/SQL anonymous block normalization for `execute_plsql_call`

**2. SchemaManager** (`db_context/schema/manager.py`)

An intelligent caching layer that sits between the connector and the MCP tools:

- **Lazy loading**: only fetches full column and relationship details for a table when a tool actually requests it, keeping startup fast for databases with thousands of tables
- **Persistent JSON cache**: written to `CACHE_DIR/{schema}.json` so the schema survives process restarts without a full re-query
- **TTL-based refresh**: different cache lifetimes per object type (PL/SQL objects: 30 min; constraints, indexes, types: 1 hour; related tables: 30 min)
- **Schema index**: a lightweight first pass that loads only table names, used for search and enumeration without paying the full metadata cost
- **Formatted output** (`db_context/schema/formatter.py`): compact grouping for tables with > 20 columns or > 10 relationships to keep AI context windows manageable

**3. DatabaseContext** (`db_context/__init__.py`)

The public API consumed by the MCP tool handlers. It composes `DatabaseConnector` and `SchemaManager` and exposes a clean async interface for every operation. It also manages the application lifecycle (startup cache build, graceful connection close on shutdown) via FastMCP's lifespan hook.

### Schema Caching Strategy

```
First startup                         Subsequent startups
──────────────                        ───────────────────
1. Connect to Oracle                  1. Connect to Oracle
2. Query all_tables → name list       2. Load {schema}.json from disk
3. Write lightweight index to disk    3. Serve requests from cache
4. Lazy-load details on first use     4. Background TTL refresh as needed
5. Append details to cache file
```

The cache file layout (`{schema}.json`):

```json
{
  "tables": {
    "EMPLOYEES": { "columns": [...], "relationships": {...}, "fully_loaded": true }
  },
  "all_table_names": ["EMPLOYEES", "DEPARTMENTS", ...],
  "last_updated": 1748000000.0,
  "object_cache": {
    "plsql": {},
    "constraints": {},
    "indexes": {},
    "related_tables": {},
    "types": {}
  },
  "cache_stats": { "hits": 42, "misses": 7, "last_full_refresh": 1748000000.0 }
}
```

### Request Flow

A typical tool call from an AI assistant follows this path:

```
AI assistant
  → calls tool (e.g. get_table_schema("EMPLOYEES"))
  → MCP protocol message sent over stdio or HTTP POST /mcp
  → FastMCP dispatches to @mcp.tool() handler
  → handler reads DatabaseContext from lifespan context
  → DatabaseContext.get_schema_info("EMPLOYEES")
      ├─ cache hit → return cached TableInfo immediately
      └─ cache miss → DatabaseConnector queries all_columns + all_constraints
                    → result stored in SchemaManager cache
                    → TableInfo returned and formatted
  → handler formats result as string
  → MCP response sent back to AI assistant
```

## Connection Modes

The database connector supports two connection modes:

### Thin Mode (Default)

By default, the connector uses Oracle's thin mode, which is a pure Python implementation. This mode is:

- Easier to set up and deploy
- Sufficient for most basic database operations
- More portable across different environments

### Thick Mode

For scenarios requiring advanced Oracle features or better performance, you can enable thick mode:

- When using Docker (recommended): Set `THICK_MODE=1` in the Docker environment variables
- When using local installation: Export `THICK_MODE=1` environment variable and ensure Oracle Client libraries, compatible with your system architecture and database version, are installed

You can specify a custom location for the Oracle Client libraries using the `ORACLE_CLIENT_LIB_DIR` environment variable. This is particularly useful when:

- You have Oracle Client libraries installed in non-standard locations
- You need to work with multiple Oracle Client versions on the same system
- You don't have administrative privileges to install Oracle Client in standard locations
- You need specific Oracle Client versions for compatibility with certain database features

Note: When using Docker, you don't need to worry about installing Oracle Client libraries as they are included in the container (Oracle Instant Client v23.7). The container supports Oracle databases versions 19c up to 23ai in both linux/arm64 and linux/amd64 architectures.

## Test Environment

The `test/db/` directory contains a fully self-contained Oracle database environment for developing and validating the MCP server locally.

### Oracle Test Database

The test database runs **Oracle XE 21c** (via the `gvenzl/oracle-xe` community image — no Oracle account required). Two schemas are pre-created and populated by the initialization scripts:

| Schema         | User           | Password     | Purpose                                                                                                                 |
| -------------- | -------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------- |
| Main (complex) | `testuser`     | `testpass`   | Multi-domain schema with ~1 200 tables across 10 business domains, designed to stress-test the caching and search logic |
| Simple         | `simpleschema` | `simplepass` | Minimal two-table schema (`categories`, `items`) for demonstrating cross-schema queries                                 |

Both schemas share the same PDB: `FREEPDB1` on port `1521`.

Connection strings:

```
testuser/testpass@//localhost:1521/FREEPDB1
simpleschema/simplepass@//localhost:1521/FREEPDB1
```

### Test Schema Structure

The main schema (`testuser`) spans 10 business domains, each with ~50-120 tables, and includes sequences, views, procedures, functions, triggers, and user-defined types:

| Domain       | Example tables                                      |
| ------------ | --------------------------------------------------- |
| HR           | `departments`, `job_titles`, `employees`, `payroll` |
| Sales        | `customers`, `orders`, `order_items`, `promotions`  |
| Inventory    | `products`, `stock`, `warehouses`, `suppliers`      |
| Finance      | `accounts`, `transactions`, `invoices`, `budgets`   |
| Operations   | `facilities`, `maintenance`, `logistics`            |
| Service      | `tickets`, `service_data_*`, `satisfaction_scores`  |
| Marketing    | `campaigns`, `leads`, `segments`                    |
| Compliance   | `audits`, `regulatory_reports`                      |
| Analytics    | `kpi_metrics`, `dashboards`                         |
| Cross-domain | `reporting_views`, `integrated_metrics`             |

The initialization is driven by four SQL scripts executed in order:

| Script                             | Purpose                                                       |
| ---------------------------------- | ------------------------------------------------------------- |
| `init/01_create_tables.sql`        | Creates all sequences and tables (both schemas)               |
| `init/02_create_simple_schema.sql` | Provisions the `simpleschema` user and its tables             |
| `init/03_populate_tables.sql`      | Inserts representative sample data                            |
| `init/04_create_views.sql`         | Creates analytical views, procedures, functions, and triggers |

### Running the Test Database

**Prerequisites**: Docker and Docker Compose.

```bash
# Start Oracle XE (first run downloads the image and initializes the DB — takes ~5 min)
cd test/db
docker compose -f docker-compose-simple.yml up -d

# Watch initialization progress
docker compose -f docker-compose-simple.yml logs -f

# Verify the DB is ready (healthcheck should be "healthy")
docker compose -f docker-compose-simple.yml ps
```

Once healthy, point the MCP server at the test database:

```bash
# Copy the example env file
cp test/db/.env.example .env

# Edit .env with:
# ORACLE_CONNECTION_STRING=testuser/testpass@//localhost:1521/FREEPDB1
# TARGET_SCHEMA=TESTUSER

# Start the MCP server against the test DB
uv run main.py
```

Stop and clean up:

```bash
# Stop (keeps data volume)
docker compose -f test/db/docker-compose-simple.yml down

# Full reset (removes data volume)
docker compose -f test/db/docker-compose-simple.yml down -v
```

### Test Queries

The `test/db/queries/` directory contains 10 SQL scripts that exercise the MCP server's query execution tools against the test database. Each script creates supporting views, procedures, and sequences — and then runs a realistic multi-domain analytical query:

| Script                                   | Analysis                                      |
| ---------------------------------------- | --------------------------------------------- |
| `01_service_satisfaction_analysis.sql`   | Satisfaction trends across service tables     |
| `02_inventory_stock_analysis.sql`        | Stock levels and reorder alerts               |
| `03_financial_transaction_analysis.sql`  | Transaction volume and anomaly detection      |
| `04_operations_facility_analysis.sql`    | Facility utilization and maintenance load     |
| `05_sales_trend_analysis.sql`            | Revenue trends and top-performing segments    |
| `06_hr_attribute_analysis.sql`           | Headcount, payroll distribution, and tenure   |
| `07_cross_department_analysis.sql`       | Cross-domain KPIs joined across domains       |
| `08_inventory_sales_correlation.sql`     | Stock-out risk correlated with sales velocity |
| `09_seasonal_trend_analysis.sql`         | Month-over-month seasonality patterns         |
| `10_integrated_performance_analysis.sql` | Executive dashboard combining all domains     |

These queries can be run directly via the `execute_select_query` and `execute_plsql_call` MCP tools, letting you verify end-to-end that the server can read, cache, and query the test database correctly.

- **Python**: Version 3.12 or higher (required for optimal performance)
- **Memory**: 4GB+ available RAM for large databases (10,000+ tables)
- **Disk**: Minimum 500MB free space for schema cache
- **Oracle**: Compatible with Oracle Database 11g and higher
- **Network**: Stable connection to Oracle database server

## Performance Considerations

- Initial cache building may take 5-10 minutes for very large databases
- Subsequent startups typically take less than 30 seconds
- Schema lookups are generally sub-second after caching
- Memory usage scales with active schema size

## Contributing

We welcome contributions! Please see our [Contributing Guidelines](CONTRIBUTING.md) for details.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

For issues and questions:

- Create an issue in this GitHub repository
