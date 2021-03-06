---
layout: "docs"
page_title: "Secret Backend: mongodb"
sidebar_current: "docs-secrets-mongodb"
description: |-
  The mongodb secret backend for Vault generates database credentials to access MongoDB.
---

# MongoDB Secret Backend

Name: `mongodb`

The `mongodb` secret backend for Vault generates MongoDB database credentials
dynamically based on configured roles. This means that services that need
to access a MongoDB database no longer need to hard-code credentials: they
can request them from Vault and use Vault's leasing mechanism to more easily
roll them.

Additionally, it introduces a new ability: with every service accessing
the database with unique credentials, it makes auditing much easier when
questionable data access is discovered: you can track it down to the specific
instance of a service based on the MongoDB username.

Vault makes use of its own internal revocation system to ensure that users
become invalid within a reasonable time of the lease expiring.

This page will show a quick start for this backend. For detailed documentation
on every path, use `vault path-help` after mounting the backend.

## Quick Start

The first step to using the mongodb backend is to mount it.
Unlike the `generic` backend, the `mongodb` backend is not mounted by default.

```
$ vault mount mongodb
Successfully mounted 'mongodb' at 'mongodb'!
```

Next, we must tell Vault how to connect to MongoDB. This is done by providing
a standard connection string (URI):

```
$ vault write mongodb/config/connection uri="mongodb://admin:Password!@mongodb.acme.com:27017/admin?ssl=true"
Key	Value
---	-----

The following warnings were returned from the Vault server:
* Read access to this endpoint should be controlled via ACLs as it will return the connection URI as it is, including passwords, if any.
```

In this case, we've configured Vault with the username `admin` and password
`Password!`, connecting to an instance at `mongodb.acme.com` on port `27017`
with TLS. The user must have privileges to manage users and their roles in the
databases Vault will manage users in. The built-in role `userAdminAnyDatabase`
is the simplest way to grant the necessary permissions if we want Vault to
manage all users in all databases.

Optionally, we can configure the lease settings for the credentials generated
by Vault. This is done by writing to the `config/lease` key:

```
$ vault write mongodb/config/lease ttl=1h max_ttl=24h
Success! Data written to: mongodb/config/lease
```

This restricts each user to being valid or leased for 1 hour at a time, with
a maximum total use period of 24 hours. This forces an application to renew
its credentials at least hourly and to recycle them once per day.

The next step is to configure a role. A role is a logical name that maps
to a policy used to generate MongoDB credentials for that role.

Note that MongoDB also uses roles. The roles you define in Vault are distinct
from the built-in and user-defined roles in MongoDB. In fact, when defining
a Vault role you may specify the MongoDB roles that should be assigned to
users created for that Vault role.

For example, let's create a "readonly" role:

```
$ vault write mongodb/roles/readonly db=foo roles='[ "readWrite", { "role": "read", "db": "bar" } ]'
Success! Data written to: mongodb/roles/readonly
```

By writing to the `roles/readonly` path we are defining the `readonly` role.
Each time Vault is asked for credentials for this role, it will create a
user in the specified MongoDB database with the MongoDB roles provided. The
username and password of each user created will be dynamically generated by
Vault. Just like when creating a user directly using `db.createUser`, the
`roles` JSON array can specify both built-in roles and user-defined roles
for both the database the user is created in and for other databases. Please
consult the MongoDB documentation for more details on Role-Based Access
Control in MongoDB. In this example, Vault will create a user in the `foo`
database with the `readWrite` built-in role on that database and the `read`
built-in role on the `bar` database.

To generate a new set of credentials for a given role, we simply read from
the credentials path for that role:

```
$ vault read mongodb/creds/readonly
Key            	Value
---            	-----
lease_id       	mongodb/creds/readonly/91685212-3040-7dde-48b1-df997c5dc8e7
lease_duration 	3600
lease_renewable	true
db             	foo
password       	c3faa86d-0f93-9649-de91-c431765e62dd
username       	vault-token-48729def-b0ca-2b17-d7b9-3ca7cb86f0ae
```

By reading from the `creds/readonly` path, Vault has generated a new set of
credentials using the `readonly` role configuration. Here we see the
dynamically generated username and password, along with a one hour lease.

Using ACLs, it is possible to restrict using the `mongodb` backend such that
trusted operators can manage the role definitions, and both users and
applications are restricted in the credentials they are allowed to read.

## API

### /mongodb/config/connection
#### POST

<dl class="api">
  <dt>Description</dt>
  <dd>
    Configures the standard connection string (URI) used to communicate with MongoDB.
  </dd>

  <dt>Method</dt>
  <dd>POST</dd>

  <dt>URL</dt>
  <dd>`/mongodb/config/connection`</dd>

  <dt>Parameters</dt>
  <dd>
    <ul>
      <li>
        <span class="param">uri</span>
        <span class="param-flags">required</span>
        The MongoDB standard connection string (URI)
      </li>
    </ul>
  </dd>

  <dd>
    <ul>
      <li>
        <span class="param">verify_connection</span>
        <span class="param-flags">optional</span>
        If set, uri is verified by actually connecting to the database.
        Defaults to true.
      </li>
    </ul>
  </dd>

  <dt>Returns</dt>
  <dd>
    A `200` response code.
  </dd>
  <dd>

    ```javascript
    {
      "lease_id": "",
      "renewable": false,
      "lease_duration": 0,
      "data": null,
      "wrap_info": null,
      "warnings": [
        "Read access to this endpoint should be controlled via ACLs as it will return the connection URI as it is, including passwords, if any."
      ],
      "auth": null
    }
    ```

  </dd>
</dl>

#### GET

<dl class="api">
  <dt>Description</dt>
  <dd>
    Queries the connection configuration. Access to this endpoint should be controlled via ACLs as it will return the
    connection URI as it is, including passwords, if any.
  </dd>

  <dt>Method</dt>
  <dd>GET</dd>

  <dt>URL</dt>
  <dd>`/mongodb/config/connection`</dd>

  <dt>Parameters</dt>
  <dd>
     None
  </dd>

  <dt>Returns</dt>
  <dd>

    ```javascript
    {
      "lease_id": "",
      "renewable": false,
      "lease_duration": 0,
      "data": {
        "uri": "mongodb://admin:Password!@mongodb.acme.com:27017/admin?ssl=true"
      },
      "wrap_info": null,
      "warnings": null,
      "auth": null
    }
    ```

  </dd>
</dl>

### /mongodb/config/lease
#### POST

<dl class="api">
  <dt>Description</dt>
  <dd>
    Configures the default lease TTL settings for credentials generated by the mongodb backend.
  </dd>

  <dt>Method</dt>
  <dd>POST</dd>

  <dt>URL</dt>
  <dd>`/mongodb/config/lease`</dd>

  <dt>Parameters</dt>
  <dd>
    <ul>
      <li>
        <span class="param">ttl</span>
        <span class="param-flags">optional</span>
        The ttl value provided as a string duration
        with time suffix. Hour is the largest suffix.
      </li>
      <li>
        <span class="param">max_ttl</span>
        <span class="param-flags">optional</span>
        The maximum ttl value provided as a string duration
        with time suffix. Hour is the largest suffix.
      </li>
    </ul>
  </dd>

  <dt>Returns</dt>
  <dd>
    A `204` response code.
  </dd>
</dl>

#### GET

<dl class="api">
  <dt>Description</dt>
  <dd>
    Queries the lease configuration.
  </dd>

  <dt>Method</dt>
  <dd>GET</dd>

  <dt>URL</dt>
  <dd>`/mongodb/config/lease`</dd>

  <dt>Parameters</dt>
  <dd>
     None
  </dd>

  <dt>Returns</dt>
  <dd>

    ```javascript
    {
      "lease_id": "",
      "renewable": false,
      "lease_duration": 0,
      "data": {
        "max_ttl": 60,
        "ttl": 60
      },
      "wrap_info": null,
      "warnings": null,
      "auth": null
    }
    ```

  </dd>
</dl>

### /mongodb/roles/\<name\>
#### POST

<dl class="api">
  <dt>Description</dt>
  <dd>
    Creates or updates a role definition.
  </dd>

  <dt>Method</dt>
  <dd>POST</dd>

  <dt>URL</dt>
  <dd>`/mongodb/roles/<name>`</dd>

  <dt>Parameters</dt>
  <dd>
    <ul>
      <li>
        <span class="param">db</span>
        <span class="param-flags">required</span>
        The name of the database users should be created in for this role.
      </li>
      <li>
        <span class="param">roles</span>
        <span class="param-flags">optional</span>
        MongoDB roles to assign to the users generated for this role.
      </li>
    </ul>
  </dd>

  <dt>Returns</dt>
  <dd>
    A `204` response code.
  </dd>
</dl>

#### GET

<dl class="api">
  <dt>Description</dt>
  <dd>
    Queries the role definition.
  </dd>

  <dt>Method</dt>
  <dd>GET</dd>

  <dt>URL</dt>
  <dd>`/mongodb/roles/<name>`</dd>

  <dt>Parameters</dt>
  <dd>
     None
  </dd>

  <dt>Returns</dt>
  <dd>

    ```javascript
    {
      "lease_id": "",
      "renewable": false,
      "lease_duration": 0,
      "data": {
        "db": "foo",
        "roles": "[\"readWrite\",{\"db\":\"bar\",\"role\":\"read\"}]"
      },
      "wrap_info": null,
      "warnings": null,
      "auth": null
    }
    ```

  </dd>
</dl>

#### LIST

<dl class="api">
  <dt>Description</dt>
  <dd>
    Returns a list of available roles. Only the role names are returned, not
    any values.
  </dd>

  <dt>Method</dt>
  <dd>LIST/GET</dd>

  <dt>URL</dt>
  <dd>`/mongodb/roles` (LIST) or `/mongodb/roles/?list=true` (GET)</dd>

  <dt>Parameters</dt>
  <dd>
     None
  </dd>

  <dt>Returns</dt>
  <dd>

    ```javascript
    {
      "lease_id": "",
      "renewable": false,
      "lease_duration": 0,
      "data": {
        "keys": [
          "dev",
          "prod"
        ]
      },
      "wrap_info": null,
      "warnings": null,
      "auth": null
    }
    ```

  </dd>
</dl>

#### DELETE

<dl class="api">
  <dt>Description</dt>
  <dd>
    Deletes the role definition.
  </dd>

  <dt>Method</dt>
  <dd>DELETE</dd>

  <dt>URL</dt>
  <dd>`/mongodb/roles/<name>`</dd>

  <dt>Parameters</dt>
  <dd>
     None
  </dd>

  <dt>Returns</dt>
  <dd>
    A `204` response code.
  </dd>
</dl>

### /mongodb/creds/
#### GET

<dl class="api">
  <dt>Description</dt>
  <dd>
    Generates a new set of dynamic credentials based on the named role.
  </dd>

  <dt>Method</dt>
  <dd>GET</dd>

  <dt>URL</dt>
  <dd>`/mongodb/creds/<name>`</dd>

  <dt>Parameters</dt>
  <dd>
     None
  </dd>

  <dt>Returns</dt>
  <dd>

    ```javascript
    {
      "lease_id": "mongodb/creds/readonly/e64e79d8-9f56-e379-a7c5-373f9b4ee3d8",
      "renewable": true,
      "lease_duration": 3600,
      "data": {
        "db": "foo",
        "password": "de0f7b50-d700-54e5-4e81-5c3724283999",
        "username": "vault-token-b32098cb-7ff2-dcf5-83cd-d5887cedf81b"
      },
      "wrap_info": null,
      "warnings": null,
      "auth": null
    }
    ```

  </dd>
</dl>
