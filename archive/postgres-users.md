# Bootstrapping Postgres Users

Setting up database users for an app can be challenging if you don’t do it often. Good permissions add a layer of security and can minimize the chances of developer mistakes.

The three types of users we’ll cover are:

Type | Description | Read | Write | Modify
--- | --- | --- | --- | ---
migrations | Schema changes | ✓ | ✓ | ✓
app | Reading and writing data | ✓ | ✓ |
analytics | Data analysis and reporting | ✓ | |

Before we jump into it, there’s something you should know about new databases.

## New Databases

After creating a new database, all users can access it and create tables in the `public` schema. This isn’t what we want. To fix this, run:

```sql
REVOKE ALL ON DATABASE mydb FROM PUBLIC;

REVOKE ALL ON SCHEMA public FROM PUBLIC;
```

Be sure to replace `mydb` with your database name.

## Roles

PostgreSQL uses the concept of roles to manage privileges. Roles can be used to define groups and users. A user is simply a role with a password and permission to log in.

The approach we’ll take is to create a group and add users to it. This makes it easy to rotate credentials in the future: just add a second user to the group, set your app’s configuration to the new user, and remove the original one.

## Migrations

First, we need a group to manage the schema. We could use a superuser, but this isn’t a great idea, as superusers can access all databases, change permissions, and create new roles. Instead, let’s create a new group.

```sql
CREATE ROLE migrations;

GRANT CONNECT ON DATABASE mydb TO migrations;

GRANT ALL ON SCHEMA public TO migrations;

ALTER ROLE migrations SET lock_timeout TO '5s';
```

We set a lock timeout so migrations don’t disrupt normal database activity while attempting to acquire a lock.

Now, we can create a user who’s a member of the group.

```sql
CREATE ROLE migrator WITH LOGIN ENCRYPTED PASSWORD 'secret' IN ROLE migrations;

ALTER ROLE migrator SET role TO 'migrations';
```

The last statement ensures tables created by the user are owned by the group.

You can generate a nice password from the command line with:

```sh
cat /dev/urandom | LC_CTYPE=C tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1
```

## App

Next, let’s create a group for our app. It’ll need to read and write data but shouldn’t need to modify the schema or truncate tables. We also want to set a statement timeout to prevent long running queries from degrading database performance.

```sql
CREATE ROLE app;

GRANT CONNECT ON DATABASE mydb TO app;

GRANT USAGE ON SCHEMA public TO app;

GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app;

GRANT SELECT, USAGE ON ALL SEQUENCES IN SCHEMA public TO app;

ALTER DEFAULT PRIVILEGES FOR ROLE migrations IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app;

ALTER DEFAULT PRIVILEGES FOR ROLE migrations IN SCHEMA public
    GRANT SELECT, USAGE ON SEQUENCES TO app;

ALTER ROLE app SET statement_timeout TO '30s';
```

> **Note:** The default privileges statements reference the group used for migrations. If you use Amazon RDS, you must run these statements as the migrator user we created above (since you don’t have access to a true superuser).

Then, create a user with:

```sql
CREATE ROLE myapp WITH LOGIN ENCRYPTED PASSWORD 'secret' IN ROLE app;
```

## Analytics

Finally, let’s create a group to be used for data analysis, reporting, and business intelligence tools (like [Blazer](https://github.com/ankane/blazer), our open-source one). These users are often referred to as a *read-only users*. We don’t want them to be able to mistakenly update data.

```sql
CREATE ROLE analytics;

GRANT CONNECT ON DATABASE mydb TO analytics;

GRANT USAGE ON SCHEMA public TO analytics;

GRANT SELECT ON ALL TABLES IN SCHEMA public TO analytics;

ALTER DEFAULT PRIVILEGES FOR ROLE migrations IN SCHEMA public
    GRANT SELECT ON TABLES TO analytics;

ALTER ROLE analytics SET statement_timeout TO '3min';
```

Once again, creating a user is relatively straightforward.

```sql
CREATE ROLE bi WITH LOGIN ENCRYPTED PASSWORD 'secret' IN ROLE analytics;
```

## Summary

You now know how to create different types of Postgres users. Spending a bit of time upfront to configure your users can make them easier to manage in the long run. This should give you a nice foundation.
