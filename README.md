# How to setup Oracle ODBC for HCL Domino on Linux

Oracle is very selective about which versions of software to use. It is recommended to use the Instant Client version that corresponds to the Oracle database version. Regarding the unixODBC version, it is possible to deviate a little, without going overboard.  
By following this guide, it is possible to obtain full functionality of the Lotusscript `LCConnection` and `ODBCConnection` objects.

## Environment

- **OS**: Rocky Linux 9.5  
- **Database**: Oracle 11.2.0.3  
- **Domino**: HCL Domino 14 FP4  

## References

- [Oracle Instant Client ODBC release compatibility](https://www.oracle.com/it/database/technologies/releasenote-odbc-ic.html)
- [unixODBC 2.3.11 RPMs](https://rpmfind.net/linux/rpm2html/search.php?query=unixODBC)
- [Oracle Instant Client Downloads (Basic + ODBC)](https://www.oracle.com/database/technologies/instant-client/downloads.html)

---

## Setup Steps

### 1. Install Required Packages

Download and install the following RPMs:

- Oracle Instant Client (Basic + ODBC)
- unixODBC 2.3.11

Install them using:

```bash
dnf install ./<package.rpm>
```

### 2. Configure TNS

Create or copy `tnsnames.ora` to `/etc/oracle/tnsnames.ora`

### 3. Verify configuration paths

Check the current unixODBC configuration with:

```bash
odbcinst -j
```

Example output:

```unixODBC 2.3.11
DRIVERS............: /etc/odbcinst.ini
SYSTEM DATA SOURCES: /etc/odbc.ini
FILE DATA SOURCES..: /etc/ODBCDataSources
USER DATA SOURCES..: /root/.odbc.ini
SQLULEN Size.......: 8
SQLLEN Size........: 8
SQLSETPOSIROW Size.: 8
```

The first two paths will be configured.

### 4. Configure `/etc/odbcinst.ini`

```ini
[Oracle11g]
Description     = Oracle ODBC driver for Oracle 11g
Driver          = /usr/lib/oracle/11.2/client64/lib/libsqora.so.11.1
Setup           =
FileUsage       = 1
CPTimeout       = 5
CPReuse         = 5
UsageCount      = 10
```

Making sure the `Driver` value points to the actual Oracle library.  

### 5. Configure `/etc/odbc.ini`

```ini
[DSNNAME]
Driver      = Oracle11g
DSN         = Oracle11g
ServerName  = NAME
```

`Driver` and `DSN` must be the same as the `odbcinst.ini` section title.  
The `ServerName` value will be the instance name as configures in the `tnsnames.ora`.

### 6. Environment Variables

Create a shell script `/etc/profile.d/oracle.sh`:

```bash
#!/bin/bash
export TNS_ADMIN=/etc/oracle
export LD_LIBRARY_PATH=/usr/lib/oracle/11.2/client64/lib:/opt/hcl/domino/notes/latest/linux:$LD_LIBRARY_PATH
```

Ensure this is executable and available at login.

### 7. Domino systemd integration (Daniel Nashed Start Script)

When starting Domino as a service, I'm using [Daniel Nashed' start script](https://nashcom.github.io/domino-startscript/), bash profile is not loaded. Instead, edit the file `/etc/systemd/system/domino.service` adding the required environment variables:

```ini
Environment=ORACLE_HOME=/usr/lib/oracle/11.2/client64
Environment=LD_LIBRARY_PATH=/usr/lib/oracle/11.2/client64/lib:/opt/hcl/domino/notes/latest/linux:$LD_LIBRARY_PATH
Environment=TNS_ADMIN=/etc/oracle
```

### 8. Check dependencies

To ensure the Oracle ODBC library links correctly:

```bash
ldd /usr/lib/oracle/11.2/client64/lib/libsqora.so.11.1
```

### 9. Test Connection with Domino

Use HCL's `dctest` tool (run as `notes` user):

```bash
/opt/hcl/domino/notes/14000000/linux/dctest
```

Then choose `3 - ODBC` and provide DSN, username and password to test it.

## Troubleshooting

### Error: `libodbc.so` not found

Create a symbolic link:

```bash
ln -s /usr/lib64/libodbc.so.2 /usr/lib64/libodbc.so
```

### Instance member SERVER does not exists

System ODBC library is not found. Might be an `LD_LIBRARY_PATH` issue or a `libodbc.so` not found (hard-link as described above). Use `dctest` to troubleshoot.

### Test ODBC connection from the command line

Run as `notes` user:

```bash
isql -v DSNNAME USER PASS
```

### Notes

- Ensure the notes user has access to the environment variables and library paths at runtime.
- bash_profile is not used for systemd services — use `Environment=` in the service file instead.
