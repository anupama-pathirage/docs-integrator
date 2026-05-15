---
title: Setup Guide
---

# Setup Guide

This guide walks you through setting up an Oracle Database instance and obtaining the connection details required to use the Oracle DB connector.

## Prerequisites

- An Oracle Database instance (on-premise or cloud). If you do not have one, you can use [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/) or run a local instance using [Oracle Database Free container image](https://container-registry.oracle.com/ords/ocr/ba/database/free).

## Step 1: Set up the Oracle database instance

1. Install or provision an Oracle Database instance (version 12c or later is recommended).
2. Ensure the database listener is running and accessible on the desired host and port (default port is **1521**).
3. Note the following connection details:
    - **Host**: The hostname or IP address of the database server (e.g., `localhost`).
    - **Port**: The listener port (default: `1521`).
    - **Service name or SID**: The database service name or SID (e.g., `ORCL`, `FREEPDB1`, or `XEPDB1`).

For Oracle Cloud Autonomous Database, download the wallet file from the OCI console and use it for secure connections.

## Step 2: Create a database user

1. Connect to the database as a privileged user (e.g., `SYS` or `SYSTEM`):

    ```
    sqlplus sys/<password>@<host>:<port>/<service_name> as sysdba
    ```

2. Create a new user for your application:

    ```sql
    CREATE USER app_user IDENTIFIED BY YourSecurePassword;
    ```

3. Grant the necessary privileges:

    ```sql
    GRANT CONNECT, RESOURCE TO app_user;
    GRANT UNLIMITED TABLESPACE TO app_user;
    ```

4. Optionally, grant additional privileges based on your needs (e.g., `CREATE VIEW`, `CREATE PROCEDURE`).

Do not use the SYS or SYSTEM account for application connections. Always create a dedicated user with the minimum required privileges.

## Step 3: Create your application schema

1. Connect as the application user:

    ```
    sqlplus app_user/YourSecurePassword@<host>:<port>/<service_name>
    ```

2. Create the tables and other database objects required by your application. For example:

    ```sql
    CREATE TABLE customers (
        id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
        name VARCHAR2(100) NOT NULL,
        email VARCHAR2(255),
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    ```

## Step 4: Configure network access (if applicable)

1. If the database is behind a firewall, ensure that port **1521** (or your configured listener port) is open for inbound connections from your Ballerina application host.
2. For Oracle Cloud databases, configure the **Access Control List (ACL)** or **Network Security Group** to allow your application's IP address.
3. If using SSL/TLS, obtain the server certificate or wallet file and configure it in your connection settings.

For local development with a containerized Oracle Database, map the container's port 1521 to your host (e.g., `docker run -p 1521:1521 ...`).
