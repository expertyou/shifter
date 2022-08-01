# Tenant-Migration-Agent (TMA): description and responsibilities


## Description

The `tma` is in contrast to the other code-bases no service running in the cloud. Moreover, the `tma` should handle and deal with database changes which must be rolled out to all tenants in one
environment (`development` or `production`). Therefor, the `tma` is a `CLI tool` which is able to connect to ***N*** tenants coming from the specified environment and performs `DDL` statements.

One of the most important aspects of the `tma` is that the operations across all tenant databases must be `atomic` and show a high fault tolerance towards errors. Since the tool is dealing which multiple database in a distributed fashion using `2-Phase Commit Protocol` where the `CLI-Tool` is the coordinator can be from use here. Links and sources for 2-PCP can be found here:

- [What is a 2-Phase-Commit?](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)
- [how to use 2-Phase Commit on Postgres](https://stackoverflow.com/questions/59057621/using-two-phase-commits-on-postgres)


## CLI-Tool *shifter*:

The tool should be able to accept `DDL` statements through a file input or via StdIn. Furthermore, the tool should make a `dry-run` showcasing the changes performed on each tenant's database.
While the ***2-Phase-Commit-Protocol` is running a live update should indicate the state of the migration for each tenant which in turn should help the user understanding what is going on.
Lastly, the tool must provide a summery of all changes made on each tenant. 

Next to these main features it would be helpful if the tool could show all available connections (concrete database connection). Also preferable would be if the tool would clearly display the environment in which it is about to perform the migration (`development` || `production`).

Since the `tcm` will be hidden within a cluster and as such cannot be reached form the outside the tool must maintain database connection details itself. Therefore, another ***nice to have*** feature would be to perform a quick `ping` call to each database in order to validated the credentials and endpoints.

### Commands:

<br>

### shifter create migration \<name\>
<hr>

*this command creates a migration file with the naming convention of `<unix-timestamp>_<name>_<version>.sql`

**Flags**

| **flag** | **short** | **description**                                                 |
|----------|-----------|-----------------------------------------------------------------|
| syntax   | s         | denotes the file ending of the migration file (default is .sql) |

<br>

#### shifter prepare
<hr>

*this command starts the database migration and rolls out the changes to all database.*

**Flags**

| **flag**           | **short** | **description**                                                                                                                              |
|--------------------|-----------|----------------------------------------------------------------------------------------------------------------------------------------------|
| env                | e         | indicates which environment (development \|\| production) to use                                                                             |
| file               | f         | path to a local file which includes SQL DDL statements which will  be executed on each database (if not provided input via stdin is enabled) |
| auto-commit        | a         | if the auto-commit is turned on directly after all databases are done with preparing the transaction the prepared transactions are commit. Default is false. However, each time a timer is started and after 5 minutes (300 seconds) the transactions are rolled back automatically. This is due to the fact that prepared transactions should not be kept unhandled. 

<br>

### shifter commit

*this commands executes the `COMMIT PREPARED` statement on all databases which persists the transaction. However, one should be aware after the command `shifter prepare` has finished it will auto-commit after 2 minutes (120 seconds) as it is not advised to keep prepared transactions unhandled for too long (see [Postgres Docs - Prepare Transaction](https://www.postgresql.org/docs/current/sql-prepare-transaction.html))*

### shifter rollback 

*this command rolls back each prepared transaction as long as the auto-commit has not yet reached 0 seconds.*

#### shifter ping
<hr>

*this command performs a `ping` to all known database connections within the provided environment*

**Flags**

| **flag** | **short** | **description**                                                                                                                              |
|----------|-----------|----------------------------------------------------------------------------------------------------------------------------------------------|
| env      | e         | indicates which environment (development \|\| production) to use                                                                             |



### Notes

When using a migration file a certain syntax must be maintained. An example of can be found in the following SQL listing

```sql

--- shifter stmt:start
ALTER TABLE users ADD COLUMN last_active timestamp without time zone;
--- shifter stmt:end 

--- shifter stmt:start
CREATE TABLE test(
    id serial, 
    col1 varchar(36)
);
--- shifter stmt:end 

--- shifter stmt:start
ALTER TABLE test ADD CONSTRAINT fk_users FOREIGN KEY col1 REFERENCE ON users(id);
--- shifter stmt:end 
```

As shown in the script above each statement is wrapped in a `--- shifter stmt:start` and `--- shifter stmt:end` comment. These comments are required in order to determine when a statement is has stopped and a new one starts. Since the file can be in `pgsql` format where `;` are allowed and do not clearly indicate the stopping of a command these wrappers are required.


