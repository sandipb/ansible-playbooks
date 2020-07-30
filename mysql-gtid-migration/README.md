# Automated MySQL GTID replication migration

<!-- TOC -->

- [Automated MySQL GTID replication migration](#automated-mysql-gtid-replication-migration)
    - [Overview](#overview)
    - [Setting up inventory](#setting-up-inventory)
    - [Usage](#usage)
    - [Checking cluster status](#checking-cluster-status)
    - [Important notes](#important-notes)
        - [Configuring the Python MySQL library package to be installed](#configuring-the-python-mysql-library-package-to-be-installed)
        - [Configuring root MySQL user credentials](#configuring-root-mysql-user-credentials)
        - [Changing where `my.cnf` is edited](#changing-where-mycnf-is-edited)

<!-- /TOC -->

## Overview

This playbook allows you to do an online migration of a MySQL cluster to GTID
based replication.

It is important to read up on the official steps here, which are being
automated:

https://dev.mysql.com/doc/refman/5.7/en/replication-mode-change-online-enable-gtids.html

The playbook allows you to migrate in four phases:

1. `enforce_gtid_consistency` is set to `WARN`: In this first phase, you are
   just asking MySQL to flag queries which cause problems in GTID mode.

   Ideally, after this phase, you should monitor the `mysqld` error logs for a
   short while to ensure that your app doesn't use any queries which can throw
   an error. Fix your app to stop all errors before moving to next phase.

   **NOTE:** `gtid_mode` is also set to `OFF` in this phase. This should be a
   no-op for everybody, and is provided as a rollback option from future phases.
   If you however run this phase on a host which has `gtid_mode` turned on, you
   will have trouble.

2. `enforce_gtid_consistency` is set to `ON`, and `gtid_mode` is set to
   `OFF_PERMISSIVE`: Make sure this phase runs successfully on all hosts and
   there are no errors in the logs before proceeding to the next stage.

3. `enforce_gtid_consistency` is set to `ON`, and `gtid_mode` is set to
   `ON_PERMISSIVE`: This will turn GTID on, but will still allow non-gtid
   transactions to be processed.

   Check the `ONGOING_ANONYMOUS_TRANSACTION_COUNT` status variable in all the
   hosts to be zero. Running the playbook in `--check` mode will show you this
   information from all the hosts. Once this value is zero on all hosts, and all
   slaves have caught up to the master, we are ready to move on to the next
   phase.

   **NOTE**: Check the warning in the official document about this stage. If you
   have non-replication based usages of the binlog, you should wait for all old
   binlog entries to expire which existed before the execution of this stage
   before moving on to the next stage.

4. In this stage, both `enforce_gtid_consistency` and `gtid_mode` will be set to
   `ON`.

   The MySQL configuration file at `/etc/my.cnf` will be edited to add these
   configuration there.

   Also, if the replication protocol on any slave is detected to not be in GTID
   mode, the replication will be turned off, GTID protocol enabled and
   replication turned back on.

## Setting up inventory

The playbook expects an inventory with two sections `leader` and `follower`,
with the MySQL leader in the first one and the followers in the second group.

```ini
[leader]
mysql-leader.example.com

[follower]
mysql-follower1.example.com
mysql-follower2.example.com
```

## Usage

```console
$ ansible-playbook gtid_migrate.yml -K -e gtid_migration_phase=<N>
...
```

Where `<N>` is a phase number between 1-4.

## Checking cluster status

You can safely run the playbook in check mode to see relevant information from
the target mysql hosts. For example:

```console
$ ansible-playbook gtid_migrate.yml --check
...
TASK [Current mysql variable and status | Display status] ********************
ok: [mysql-leader.example.com] => {
    "mysql_state": {
        "enforce_gtid_consistency": "ON",
        "gtid_mode": "ON",
        "is_leader": true,
        "master_auto_pos": "-1",
        "ongoing_txn_count": "0",
        "slave_lag": "-1"
    }
}
ok: [mysql-follower.example.com] => {
    "mysql_state": {
        "enforce_gtid_consistency": "ON",
        "gtid_mode": "ON",
        "is_leader": false,
        "master_auto_pos": "1",
        "ongoing_txn_count": "0",
        "slave_lag": "0"
    }
}
```

## Important notes

### Configuring the Python MySQL library package to be installed

The Ansible modules used in the playbook needs a package which provides the
`MySQLdb` or `PyMySQL` libraries.

The playbook will attempt to install the `MySQL-Python` package mentioned in the
`vars` section at the top for my RHEL7 installation. It will attempt to check
the presence of this package in every execution of the playbook. If you know you
already have the package installed and want to skip this step and save a couple
of seconds, run the playbook with the parameter `-e skip_install_package=yes`.

If you want to change the name or version of the package change the
`mysql_python_packages` variable accordingly. If your host is not RHEL7 but an
Ubuntu host, you would probably want to put in the equivalent deb package name
there.

### Configuring root MySQL user credentials

To keep things simple, I have assumed in this playbook that the root MySQL
password is stored in `/root/.my.cnf` or is empty.

If you plan to pass in the credentials of the root user in some other way, like
a vault-encoded one (which you should), then you will need to modify the
playbook accordingly.

### Changing where `my.cnf` is edited

Change the value of the playbook variable `my_cnf_insert_after` if you want the
GTID configuration parameters to be added somewhere else in your `/etc/my.cnf`.
The comments provide a failsafe location alternative.
