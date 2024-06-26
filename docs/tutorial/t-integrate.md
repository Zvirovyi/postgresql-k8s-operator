> [Charmed PostgreSQL K8s Tutorial](/t/9296) >  6. Integrate with other applications

# Integrate with other applications
[Integrations](https://juju.is/docs/sdk/integration), known as "relations" in Juju 2.9, are the easiest way to create a user for PostgreSQL in Charmed PostgreSQL K8s. 

Integrations automatically create a username, password, and database for the desired user/application. As mentioned earlier in [3. Access PostgreSQL](/t/13702) , it is a better practice to connect to PostgreSQL via a specific user rather than the admin user.

In this section, you will integrate your Charmed PostgreSQL K8s with another application.

## Summary
- [Deploy `data-integrator`](#heading--deploy-data-integrator)
- [Integrate with PostgreSQL](#heading--integrate-with-postgresq)
- [Access the related database](#heading--access-related-database)
- [Remove the user](#heading--remove-user)

---

<a href="#heading--deploy-data-integrator"><h2 id="heading--deploy-data-integrator"> Deploy <code>data-integrator</code> </h2></a>

In this tutorial, we will relate to the [`data-integrator` charm](https://charmhub.io/data-integrator). This is a bare-bones charm that allows for central management of database users, providing support for different kinds of data platforms (e.g. PostgreSQL, MySQL, MongoDB, Kafka, etc) with a consistent, opinionated and robust user experience. 

To deploy the `data-integrator` charm, run the following command:

```shell
juju deploy data-integrator --channel edge --config database-name=test-database
```

Expected output:
```
Located charm "data-integrator" in charm-hub, revision 6
Deploying "data-integrator" from charm-hub charm "data-integrator", revision 6 in channel edge on jammy
```

`juju status` will show a `blocked` state for the newly deployed charm. This is expected, since the applications have not yet been related (integrated).

Example `juju status` output:
```
Model     Controller  Cloud/Region        Version  SLA          Timestamp
tutorial  charm-dev   microk8s/localhost  2.9.42   unsupported  12:11:53+01:00

App              Version  Status   Scale  Charm            Channel    Rev  Address         Exposed  Message
data-integrator           waiting      1  data-integrator  edge       6    10.152.183.66   no       installing agent
postgresql-k8s            active       2  postgresql-k8s   14/stable  56   10.152.183.167  no

Unit                Workload    Agent  Address       Ports  Message
data-integrator/0*  blocked     idle   10.1.188.211         Please relate the data-integrator with the desired product
postgresql-k8s/0*   active      idle   10.1.188.206
postgresql-k8s/1    active      idle   10.1.188.209
```


<a href="#heading--integrate-with-postgresql"><h2 id="heading--integrate-with-postgresql"> Integrate with PostgreSQL </h2></a>

Now that the `data-integrator` charm has been set up, we can relate it to PostgreSQL. This will automatically create a username, password, and database for `data-integrator`.

Integrate the two applications with the following command:
```shell
juju integrate data-integrator postgresql-k8s
```
Wait for `juju status --watch 1s` to show all applications/units as `active`:
```
Model     Controller  Cloud/Region        Version  SLA          Timestamp
tutorial  charm-dev   microk8s/localhost  2.9.42   unsupported  12:12:12+01:00

App              Version  Status   Scale  Charm            Channel    Rev  Address         Exposed  Message
data-integrator           waiting      1  data-integrator  edge        6   10.152.183.66   no       installing agent
postgresql-k8s            active       2  postgresql-k8s   14/stable  56   10.152.183.167  no

Unit                Workload    Agent  Address       Ports  Message
data-integrator/0*  active      idle   10.1.188.211
postgresql-k8s/0*   active      idle   10.1.188.206
postgresql-k8s/1    active      idle   10.1.188.209
```

To retrieve information such as the username, password, and database, run the command
```shell
juju run data-integrator/leader get-credentials
```

This should output something like:
```yaml
unit-data-integrator-0:
  UnitId: data-integrator/0
  id: "12"
  results:
    ok: "True"
    postgresql:
      database: test-database
      endpoints: postgresql-k8s-primary.tutorial.svc.cluster.local:5432
      password: WHnROd8wqzQKzd4F
      read-only-endpoints: postgresql-k8s-replicas.tutorial.svc.cluster.local:5432
      username: relation_id_3
      version: "14.5"
  status: completed
  timing:
    completed: 2023-03-20 11:12:26 +0000 UTC
    enqueued: 2023-03-20 11:12:25 +0000 UTC
    started: 2023-03-20 11:12:26 +0000 UTC
```
Note that your hostnames, usernames, and passwords will likely be different.

<a href="#heading--access-related-database"><h2 id="heading--access-related-database"> Access the related database </h2></a>

Use `endpoints`, `username`, `password` from above to connect newly created database `test-database` on the PostgreSQL server:
```shell
> psql --host=10.1.188.206 --username=relation_id_3 --password test-database
Password:
...
test-database=> \l
...
 test-database | operator | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/operator              +
               |          |          |             |             | operator=CTc/operator     +
               |          |          |             |             | relation_id_3=CTc/operator
...
```

The newly created database `test-database` is also available on all other PostgreSQL cluster members:
```shell
> psql --host=10.89.49.209 --username=relation-3 --password --list
...
 test-database | operator | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/operator              +
               |          |          |             |             | operator=CTc/operator     +
               |          |          |             |             | relation_id_3=CTc/operator
...
```

When you relate two applications, Charmed PostgreSQL K8s automatically sets up a new user and database for you.
Note the database name we specified when we first deployed the `data-integrator` charm: `--config database-name=test-database`.


<a href="#heading--remove-user"><h2 id="heading--remove-user"> Remove the user </h2></a>

Removing the integration automatically removes the user that was created when the integration was created. Enter the following to remove the integration:
```shell
juju remove-relation postgresql-k8s data-integrator
```

Now try again to connect to the same PostgreSQL you just used in the previous section:
```shell
> psql --host=10.1.188.206 --username=relation_id_3 --password --list
```

This will output an error message like the one shown below:
```
psql: error: connection to server at "10.1.188.206", port 5432 failed: FATAL:  password authentication failed for user "relation_id_3"
```
This is because the user no longer exists, as expected. Remember, `juju remove-relation postgresql-k8s data-integrator` also removes the user.

Data remains on the server at this stage.

If you want to create a user again, integrate the applications again:
```shell
juju integrate data-integrator postgresql-k8s
```
Re-integrating generates a new user and password:
```shell
juju run data-integrator/leader get-credentials
```
You can then connect to the database with these new credentials.
From here you will see all of your data is still present in the database.

**Next step:** [7. Enable TLS](/t/9302)