# MySQL InnoDB Cluster on AWS EC2 --- Deployment Runbook

**Project:** `testingCluster`\
**Scope:** Build and validate a MySQL 8.4 InnoDB Cluster on Ubuntu EC2
instances, from server preparation through cluster membership and
replication testing.\
**Out of scope:** MySQL Router deployment, SSH tunnelling,
PHP/application connectivity, production hardening, and automated
backups. These will be documented separately.

> **Important:** This runbook records the environment and issues
> encountered during the deployment. Before reusing it, verify the
> current IP addresses, MySQL versions, cluster membership, and
> security-group rules. Do not assume that an old IP address is still
> assigned to an EC2 instance.

------------------------------------------------------------------------

## 1. Final architecture

The cluster uses **InnoDB Cluster with Group Replication in
single-primary mode**.

-   One member is the `PRIMARY` and accepts writes.
-   The other members are `SECONDARY` and receive transactions through
    Group Replication.
-   MySQL Shell AdminAPI manages cluster configuration and membership.
-   MySQL Router is not part of this setup yet.

### Known EC2 members

  ---------------------------------------------------------------------------------
  Member       Hostname observed           Private IP       MySQL port MySQL
                                                                       version
  ------------ -------------------- ----------------- ---------------- ------------
  Initial      `mysql-master` /       `172.31.22.175`             3306 8.4.11
  primary      `master`                                                

  Existing     `mysql-slave1` /       `172.31.17.122`             3306 8.4.11
  replica      `replica`                                               

  Third member Confirm from         Verify current IP             3306 Must be
               `cluster.status()`                                      compatible
                                                                       with cluster
  ---------------------------------------------------------------------------------

The first two members were confirmed `ONLINE` in the cluster status
output. A third EC2 was subsequently added and reported as working, but
its final IP and membership details were not captured in the notes used
to prepare this document. Confirm its exact identity with
`cluster.status()` and update this table.

### Cluster details

  Setting                                Value
  -------------------------------------- -----------------------------
  Cluster name                           `testingCluster`
  Topology mode                          `Single-Primary`
  MySQL version on original members      `8.4.11`
  MySQL Shell version observed           `8.4.8`
  MySQL port                             `3306`
  Group Replication communication port   `33061` (default XCom port)
  Cluster administrator                  `icadmin`
  Initial primary private IP             `172.31.22.175`

------------------------------------------------------------------------

## 2. Prerequisites

Before configuring an EC2 member, ensure:

1.  The EC2 instance is in a network that can communicate privately with
    the other cluster members.
2.  MySQL Server is installed and running.
3.  MySQL versions are compatible. The original members both ran MySQL
    `8.4.11`; using the same patch release is the simplest approach for
    this environment.
4.  Every MySQL instance has a unique `server_id` and `server_uuid`.
5.  The cluster members can communicate over the required security-group
    ports.
6.  You have an administrative MySQL account suitable for MySQL Shell
    AdminAPI.
7.  You have a backup of any data that must be retained.

### AWS Security Group

Use a dedicated database-cluster security group. Allow inbound traffic
from that same security group:

  ------------------------------------------------------------------------
  Protocol                          Port Source           Purpose
  ---------------- --------------------- ---------------- ----------------
  TCP                               3306 Cluster security MySQL client
                                         group            connections and
                                                          distributed
                                                          recovery

  TCP                              33061 Cluster security Group
                                         group            Replication XCom
                                                          communication
  ------------------------------------------------------------------------

Do not expose MySQL ports `3306` or `33061` to `0.0.0.0/0`.

If administering the EC2 instances over SSH, allow TCP `22` only from
trusted administrator IP addresses or through your approved access
method.

If a later application server or MySQL Router host needs MySQL access,
add a narrowly scoped rule for that source. Do not open ports publicly.

------------------------------------------------------------------------

## 3. Confirm instance identity and networking

Run on **each EC2 instance**:

``` bash
hostname
hostname -I
mysql --version
```

Connect to MySQL and run:

``` sql
SELECT
    @@hostname AS hostname,
    @@server_id AS server_id,
    @@server_uuid AS server_uuid,
    @@report_host AS report_host,
    @@port AS port,
    @@version AS mysql_version;
```

Check that:

-   Each instance has a unique `server_id`.
-   Each instance has a unique `server_uuid`.
-   `report_host` is set to that instance's own private IP address.
-   The MySQL version is compatible with the other cluster members.

From each relevant EC2 host, test connectivity to the other members:

``` bash
nc -vz PEER_PRIVATE_IP 3306
nc -vz PEER_PRIVATE_IP 33061
```

Replace `PEER_PRIVATE_IP` with the other EC2 instance's private IP. A
successful TCP test confirms network reachability; it does not by itself
confirm MySQL authentication or Group Replication health.

------------------------------------------------------------------------

## 4. Configure MySQL for Group Replication

On Ubuntu, the main configuration file used in this environment was:

``` bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Place the settings under the existing `[mysqld]` section. Do not create
duplicate `[mysqld]` sections unnecessarily.

Example for the **initial primary**:

``` ini
[mysqld]
server-id = 1
report_host = 172.31.22.175

gtid_mode = ON
enforce_gtid_consistency = ON

log_bin = mysql-bin
binlog_format = ROW
log_replica_updates = ON

binlog_checksum = CRC32
```

Example for the **second member**:

``` ini
[mysqld]
server-id = 2
report_host = 172.31.17.122

gtid_mode = ON
enforce_gtid_consistency = ON

log_bin = mysql-bin
binlog_format = ROW
log_replica_updates = ON

binlog_checksum = CRC32
```

For a third member, use a unique `server-id` (for example, `3`) and its
own private IP in `report_host`.

### Important configuration notes

-   `server-id` must be unique across the MySQL topology.
-   `server_uuid` is generated by MySQL and must also be unique. Do not
    copy a data directory containing another member's `auto.cnf` to
    create a new independent MySQL instance.
-   `report_host` must contain only the hostname or IP address, not the
    configuration key or an equals sign.
-   Keep other Group Replication-related settings consistent across
    members.
-   The original instances reported `lower_case_table_names = 0`. Keep
    this setting consistent. It is not a setting to casually change on
    an initialized data directory.
-   The original instances reported GTID mode enabled, row-based binary
    logging, `log_replica_updates` enabled, and CRC32 binlog checksums.

### Restart MySQL after file changes

``` bash
sudo systemctl restart mysql
sudo systemctl status mysql --no-pager
```

Then verify:

``` sql
SHOW VARIABLES LIKE 'report_host';
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'enforce_gtid_consistency';
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'log_replica_updates';
SHOW VARIABLES LIKE 'binlog_checksum';
```

------------------------------------------------------------------------

## 5. Important issue encountered: incorrect `report_host`

During the initial setup, the master advertised an invalid address.
MySQL Shell reported an error similar to:

``` text
MySQL server at 'master:3306' can't connect to 'master:3306'.
Verify configured report_host and that the address is valid at that host.
```

The `report_host` variable was then found to contain a malformed literal
value:

``` text
report_host = 172.31.22.175
```

That is **not** the correct value. The variable must contain only:

``` text
172.31.22.175
```

The correct configuration-file line is:

``` ini
report_host = 172.31.22.175
```

For the second member:

``` ini
report_host = 172.31.17.122
```

### Why `SET PERSIST` did not work

An attempt was made to run:

``` sql
SET PERSIST report_host = '172.31.22.175';
```

MySQL returned:

``` text
ERROR 1238 (HY000): Variable 'report_host' is a read only variable
```

`report_host` is read-only at runtime in this environment. Correct it in
the MySQL configuration file and restart the MySQL service. Do not try
to fix it with `SET PERSIST`.

### Verify the result

Run on each instance:

``` sql
SHOW VARIABLES LIKE 'report_host';
```

Expected values:

  Instance         Expected value
  ---------------- ------------------------------
  `mysql-master`   `172.31.22.175`
  `mysql-slave1`   `172.31.17.122`
  Third member     That member's own private IP

The value must not include `report_host =` or a port number.

------------------------------------------------------------------------

## 6. MySQL accounts and authentication

### Root account

On the original master, this query showed:

``` sql
SELECT user, host, plugin
FROM mysql.user
WHERE user = 'root';
```

Result:

``` text
root | localhost | auth_socket
```

`sudo mysql` worked, while password-based root authentication through
MySQL Shell failed with error `1698`.

**Do not change root authentication just to make MySQL Shell work.** Use
`sudo mysql` for local administrative SQL and a dedicated account for
MySQL Shell.

### Cluster administration account

A dedicated account named `icadmin` was created for AdminAPI operations.
The example used was:

``` sql
CREATE USER 'icadmin'@'%'
IDENTIFIED WITH caching_sha2_password
BY 'REPLACE_WITH_A_STRONG_PASSWORD';

GRANT ALL PRIVILEGES ON *.*
TO 'icadmin'@'%'
WITH GRANT OPTION;
```

Use a strong, unique password and store it securely. Do not commit real
passwords to a Git repository or this documentation.

The exact privileges required by MySQL Shell AdminAPI can vary with the
operation and MySQL Shell version. For a reusable deployment, prefer the
supported AdminAPI account-configuration workflow and verify account
privileges against the documentation for the installed MySQL Shell
version. Do not assume that an application account should have these
administrative privileges.

### Account availability on new members

When adding a member, ensure the account used to connect to that
instance exists and can authenticate. Depending on the joining and
recovery workflow, account provisioning and recovery can differ. Follow
the AdminAPI prompts and documentation for the selected recovery method.

------------------------------------------------------------------------

## 7. MySQL Shell installation and Python mode

MySQL Shell was installed on the initial primary. The observed version
was:

``` text
mysqlsh  Ver 8.4.8
```

The `mysql` server instances were version `8.4.11`.

Check the installed version:

``` bash
mysqlsh --version
```

Start MySQL Shell in Python mode:

``` bash
mysqlsh --py
```

In this environment, JavaScript mode was unavailable:

``` text
mysqlsh --js
JavaScript is not supported.
```

The command `\js` also returned `Unknown command`. Therefore, the
successful workflow used **Python mode** and Python-style AdminAPI
method names.

Connect to the local primary:

``` text
\connect icadmin@localhost:3306
```

Or connect to the primary using its private IP:

``` text
\connect icadmin@172.31.22.175:3306
```

Use Python API names such as:

``` python
dba.check_instance_configuration(...)
dba.create_cluster(...)
dba.get_cluster(...)
cluster.status()
cluster.add_instance(...)
```

Do not mix JavaScript API spelling (for example,
`checkInstanceConfiguration`) with Python mode.

------------------------------------------------------------------------

## 8. Validate each instance before cluster operations

Run from MySQL Shell Python mode, connected to the instance being
checked:

``` python
dba.check_instance_configuration('icadmin@localhost:3306')
```

For a remote instance:

``` python
dba.check_instance_configuration('icadmin@PEER_PRIVATE_IP:3306')
```

The original primary validation completed successfully and returned:

``` json
{
    "status": "ok"
}
```

The validation output confirmed:

-   No incompatible tables were detected.
-   The instance configuration was compatible with InnoDB Cluster.
-   The instance was valid for InnoDB Cluster usage.

Both original instances were reported as valid.

If validation fails, read the full diagnostic output and fix the
reported issue before proceeding. Do not ignore warnings about
addresses, GTIDs, storage engines, or Group Replication configuration.

------------------------------------------------------------------------

## 9. Create the initial InnoDB Cluster

The initial cluster was created on the master (`172.31.22.175`).

Start MySQL Shell:

``` bash
mysqlsh --py
```

Connect:

``` text
\connect icadmin@localhost:3306
```

Create the cluster:

``` python
cluster = dba.create_cluster('testingCluster')
```

Successful creation reported:

``` text
Cluster successfully created. Use Cluster.add_instance() to add MySQL instances.
```

The initial primary was:

``` text
172.31.22.175:3306
```

### Important

-   Run `dba.create_cluster()` only when creating a new cluster.
-   Do not run it again when adding another EC2 to an existing cluster.
-   To work with an existing cluster, use
    `dba.get_cluster('testingCluster')`.

------------------------------------------------------------------------

## 10. Add an existing MySQL instance to the cluster

The second original instance had previously been configured as an
asynchronous replica of the master. Its recorded state before the
migration included:

-   `Replica_IO_Running: Yes`
-   `Replica_SQL_Running: Yes`
-   `Seconds_Behind_Source: 0`
-   GTID auto-position enabled
-   Executed GTID set matching the master's recorded GTID history at
    that time

Both original instances had the same recorded GTID set:

``` text
64d2cd5f-d8fb-11f1-9ee8-0afffe486245:1-9
```

The second member was subsequently shown as `ONLINE` in the InnoDB
Cluster status output.

### Migration caution

An asynchronous replication channel and Group Replication are different
replication mechanisms. Before joining an existing asynchronous replica
to a cluster:

1.  Back up any data that must be preserved.
2.  Confirm the replica is healthy and caught up.
3.  Compare GTID execution history.
4.  Confirm that no independent writes were made on the replica.
5.  Follow the MySQL Shell AdminAPI migration procedure for the
    installed version.
6.  Stop and remove the old asynchronous replication channel only at the
    correct point in the migration.

Commands such as the following affect the old asynchronous channel and
must not be run casually:

``` sql
STOP REPLICA;
RESET REPLICA ALL;
```

`RESET REPLICA ALL` removes replication connection configuration. It is
not a substitute for validating GTID compatibility or preparing a member
for Group Replication.

### Add a new, prepared EC2 member

For a fresh or disposable joining instance, the general AdminAPI flow
is:

``` python
cluster = dba.get_cluster('testingCluster')

cluster.status()
```

Validate the joining instance:

``` python
dba.check_instance_configuration(
    'icadmin@NEW_MEMBER_PRIVATE_IP:3306'
)
```

Then add it using the appropriate recovery method:

``` python
cluster.add_instance(
    'icadmin@NEW_MEMBER_PRIVATE_IP:3306',
    {'recoveryMethod': 'clone'}
)
```

**Clone warning:** Clone replaces the joining instance's existing MySQL
data with a copy from a donor member. Use it only when the joining
instance's existing data can be discarded. For an instance whose data
must be retained, evaluate incremental recovery and GTID compatibility
instead.

The AdminAPI may prompt for confirmation or report that instance
configuration changes are needed. Read and understand each prompt before
accepting it.

After adding a member, wait for the operation to finish and verify that
the new member is `ONLINE`.

------------------------------------------------------------------------

## 11. Check cluster health

From MySQL Shell Python mode:

``` python
cluster = dba.get_cluster('testingCluster')
cluster.status()
```

The initial two-member status showed:

-   Cluster name: `testingCluster`
-   Topology mode: `Single-Primary`
-   Primary: `172.31.22.175:3306`
-   `172.31.22.175:3306`: `PRIMARY`, `R/W`, `ONLINE`
-   `172.31.17.122:3306`: `SECONDARY`, `R/O`, `ONLINE`
-   Cluster status: `OK_NO_TOLERANCE`

Example structure:

``` json
{
  "clusterName": "testingCluster",
  "defaultReplicaSet": {
    "primary": "172.31.22.175:3306",
    "status": "OK_NO_TOLERANCE",
    "topologyMode": "Single-Primary"
  }
}
```

The actual output includes each member's address, role, mode, and
status.

### Status meanings

  -----------------------------------------------------------------------
  Status or field                     Meaning
  ----------------------------------- -----------------------------------
  `ONLINE`                            Member is online and participating

  `PRIMARY`                           Current write-primary member

  `SECONDARY`                         Non-primary member

  `R/W`                               Read/write mode

  `R/O`                               Read-only mode

  `Single-Primary`                    One primary at a time

  `OK_NO_TOLERANCE`                   Cluster is operating but cannot
                                      tolerate a member failure

  `OK`                                Cluster is healthy and has the
                                      expected fault tolerance
  -----------------------------------------------------------------------

### Quorum and number of members

A two-member cluster cannot maintain a majority if either member is
lost. With three voting members, the cluster can maintain a majority
when one member fails, assuming the other two remain healthy and can
communicate.

Do not treat a two-member `OK_NO_TOLERANCE` cluster as highly available.

After adding the third member, verify that all three are `ONLINE` and
that the cluster status reports the expected fault tolerance. Do not
infer the third member's address or role; read the current
`cluster.status()` output.

------------------------------------------------------------------------

## 12. Test Group Replication with application data

A simple test was performed after the cluster members were online.

On the current primary, create a test database and table:

``` sql
CREATE DATABASE cluster_test;

USE cluster_test;

CREATE TABLE replication_demo (
    id INT PRIMARY KEY AUTO_INCREMENT,
    message VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO replication_demo (message)
VALUES
('First InnoDB Cluster test'),
('Testing Group Replication'),
('Master to secondary replication');
```

Connect directly to a secondary member and run:

``` sql
SELECT *
FROM cluster_test.replication_demo;
```

The expected result is that all inserted rows are visible on the
secondary.

This verifies that transactions written to the primary are replicated to
the other cluster members.

Clean up the test database when it is no longer needed:

``` sql
DROP DATABASE cluster_test;
```

Run cleanup on the primary; Group Replication will replicate the DDL to
the other members.

------------------------------------------------------------------------

## 13. Troubleshooting notes from this deployment

### Problem A: MySQL Shell could not connect to `master:3306`

Observed error:

``` text
MySQL server at 'master:3306' can't connect to 'master:3306'.
Verify configured report_host and that the address is valid at that host.
```

**Cause:** The advertised instance hostname was not reachable/resolvable
from the server.

**Fix used:** Set `report_host` to the instance's own EC2 private IP in
`mysqld.cnf`, then restart MySQL.

Example:

``` ini
report_host = 172.31.22.175
```

### Problem B: Invalid address format

Observed error:

``` text
ValueError: Dba.create_cluster: Invalid address format:
'report_host = 172.31.22.175:3306'
```

**Cause:** The value of `report_host` included the literal text
`report_host =`.

**Fix:** Correct the configuration-file value so that the variable
contains only the IP address:

``` ini
report_host = 172.31.22.175
```

Verify the runtime value:

``` sql
SHOW VARIABLES LIKE 'report_host';
```

Expected value:

``` text
172.31.22.175
```

### Problem C: `SET PERSIST report_host` failed

Observed error:

``` text
ERROR 1238 (HY000): Variable 'report_host' is a read only variable
```

**Cause:** `report_host` is read-only at runtime.

**Fix:** Change the MySQL configuration file and restart MySQL. Do not
use `SET PERSIST` for this variable.

### Problem D: MySQL Shell root authentication failed

Observed error:

``` text
ERROR 1698
```

**Cause:** Root used `auth_socket` authentication.

**Fix:** Keep root authentication unchanged. Use `sudo mysql` for local
administration and a dedicated AdminAPI account for MySQL Shell.

### Problem E: JavaScript mode was unavailable

Observed:

``` text
JavaScript is not supported.
```

**Fix:** Use:

``` bash
mysqlsh --py
```

Use Python-style AdminAPI methods and commands.

------------------------------------------------------------------------

## 14. Operational commands cheat sheet

### Linux service checks

``` bash
sudo systemctl status mysql --no-pager
sudo systemctl restart mysql
sudo journalctl -u mysql -n 100 --no-pager
```

### MySQL identity and configuration

``` sql
SELECT
    @@hostname,
    @@server_id,
    @@server_uuid,
    @@report_host,
    @@port,
    @@version;

SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'enforce_gtid_consistency';
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'log_replica_updates';
SHOW VARIABLES LIKE 'binlog_checksum';
SHOW VARIABLES LIKE 'lower_case_table_names';

SELECT @@GLOBAL.gtid_executed;
```

### MySQL Shell cluster management

``` bash
mysqlsh --py
```

``` text
\connect icadmin@172.31.22.175:3306
```

``` python
cluster = dba.get_cluster('testingCluster')

cluster.status()
```

### Network checks

``` bash
nc -vz PEER_PRIVATE_IP 3306
nc -vz PEER_PRIVATE_IP 33061
```

------------------------------------------------------------------------

## 15. Backup and safety recommendations

Before changing cluster membership or recovering an instance:

-   Take a logical or physical backup appropriate for the database size
    and recovery requirements.
-   Confirm which instance is primary.
-   Check that all members are online and synchronized.
-   Review GTID sets before joining an existing data-bearing instance.
-   Never use clone recovery on an instance whose data must be
    preserved.
-   Keep database ports private to the cluster network.
-   Store passwords and private keys outside source control.
-   Test recovery and failover procedures before using the cluster for
    production workloads.

A small test database is not a reason to skip backups when practicing a
migration. The purpose is to develop a safe repeatable procedure.

------------------------------------------------------------------------

## 16. Current completion boundary and next phase

### Completed in this deployment

-   [x] Two original MySQL EC2 instances prepared for InnoDB Cluster.
-   [x] GTID-based configuration verified on the original members.
-   [x] `report_host` issue diagnosed and corrected using private IP
    addresses.
-   [x] MySQL Shell Python mode used for AdminAPI.
-   [x] Both original instances passed
    `dba.check_instance_configuration()`.
-   [x] `testingCluster` created successfully on the initial primary.
-   [x] Original primary and secondary were shown `ONLINE`.
-   [x] Test data replication was confirmed.
-   [x] A third EC2 was subsequently reported as successfully added;
    verify its final membership and IP using `cluster.status()`.

### Not covered yet

-   [ ] Record and verify the final third-member hostname, private IP,
    UUID, and role.
-   [ ] Deploy MySQL Router.
-   [ ] Configure Router read/write and read-only endpoints.
-   [ ] Configure local laptop access through a secure SSH tunnel or
    another private network path.
-   [ ] Connect the PHP application using PDO.
-   [ ] Implement application reconnect/retry behavior for failover.
-   [ ] Test primary failover and recovery.
-   [ ] Configure monitoring, alerting, and scheduled backups.
-   [ ] Document production security and disaster-recovery procedures.

**This runbook intentionally stops before application connectivity.**
MySQL Router, SSH tunnelling, and PHP configuration belong to the next
phase.

------------------------------------------------------------------------

## 17. References

Use the official MySQL documentation for the exact MySQL Server and
Shell release deployed:

-   MySQL InnoDB Cluster:
    https://dev.mysql.com/doc/mysql-shell/8.4/en/mysql-innodb-cluster.html
-   AdminAPI:
    https://dev.mysql.com/doc/mysql-shell/8.4/en/mysql-shell-apis.html
-   InnoDB Cluster setup:
    https://dev.mysql.com/doc/mysql-shell/8.4/en/mysql-innodb-cluster-deploy.html
-   Adding instances:
    https://dev.mysql.com/doc/mysql-shell/8.4/en/mysql-innodb-cluster-adding-instances.html
-   MySQL Router: https://dev.mysql.com/doc/mysql-router/8.4/en/
-   Group Replication:
    https://dev.mysql.com/doc/refman/8.4/en/group-replication.html

------------------------------------------------------------------------

## 18. Handover checklist for the next engineer

Before making changes, the next engineer should:

1.  Read this document completely.
2.  Obtain the current EC2 private IPs and hostnames from AWS.
3.  Run `cluster.status()` and record all current members.
4.  Confirm all members are `ONLINE`.
5.  Confirm which member is `PRIMARY`.
6.  Confirm MySQL Server and MySQL Shell versions.
7.  Confirm the security-group rules are restricted to trusted sources.
8.  Take a backup before any topology or recovery change.
9.  Continue with the separate MySQL Router and application-connectivity
    runbook.

**End of runbook --- InnoDB Cluster setup and validation phase.**
