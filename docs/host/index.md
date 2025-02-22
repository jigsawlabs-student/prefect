---
description: Hosting Prefect Sever
icon: octicons/server-24
tags:
    - UI
    - dashboard
    - Prefect Server
    - Observability
    - Events
    - Serve
    - Database
    - SQLite 
---

# Hosting Prefect Server

After you install Prefect you have a Python SDK client that can communicate with [Prefect Cloud](https://app.prefect.cloud), the platform hosted by Prefect. You also have an [API server](/api-ref/) backed by a database and a UI. 

In this section you'll learn how to host your own Prefect Server.

![Prefect Server UI](/img/ui/flow-run-page-server.png)

Spin up a local Prefect server UI with the `prefect server start` CLI command in the terminal:

<div class="terminal">
```bash
$ prefect server start
```
</div>

Open the URL for the Prefect server UI ([http://127.0.0.1:4200](http://127.0.0.1:4200) by default) in a browser. 

![Viewing the orchestrated flow runs in the Prefect UI.](/img/tutorial/first-steps-ui.png)

Shut down the Prefect server with <kdb> ctrl </kbd> + <kdb> c </kbd> in the terminal.


### Differences between Prefect Server and Cloud

Prefect Server and Cloud share a base of features. Prefect Cloud also includes the following features that you can read about in the [Cloud](/cloud/) section of the docs. 

- [User accounts](#user-accounts) &mdash; personal accounts for working in Prefect Cloud. 
- [Workspaces](/cloud/workspaces/) &mdash; isolated environments to organize your flows, deployments, and flow runs.
- [Automations](/cloud/automations/) &mdash; configure triggers, actions, and notifications in response to real-time monitoring events.
- [Email notifications](/cloud/automations/) &mdash; send email alerts from Prefect's serves based on automation triggers.
- [Organizations](/cloud/organizations/) &mdash; user and workspace management features that enable collaboration for larger teams.
- [Service accounts](/cloud/users/service-accounts/) &mdash; configure API access for running agents or executing flow runs on remote infrastructure.
- [Custom role-based access controls (RBAC)](/cloud/users/roles/) &mdash; assign users granular permissions to perform certain activities within an organization or a workspace.
- [Single Sign-on (SSO)](/cloud/users/sso/) &mdash; authentication using your identity provider.
- [Audit Log](/cloud/users/audit-log/) &mdash; a record of user activities to monitor security and compliance.
- Collaborators &mdash; invite others to work in your [workspace](/cloud/workspaces/#workspace-collaborators) or [organization](/cloud/organizations/#organization-members).


### Configuring Prefect Server

Go to your terminal session and run this command to set the API URL to point to a Prefect server instance:

<div class='terminal'>
```bash
$ prefect config set PREFECT_API_URL="http://127.0.0.1:4200/api"
```
</div>


!!! tip "`PREFECT_API_URL` required when running Prefect inside a container"
    You must set the API Server address to use Prefect within a container, such a a Docker container. 

    You can save the API server address in a [Prefect profile](/concepts/settings/). Whenever that profile is active, the API endpoint will be be at that address.

    See [Profiles & Connfiguration](/concepts/settings/) for more information on profiles and configurable Prefect settings.

## Prefect Database

The Prefect database persists data to track the state of your flow runs and related Prefect concepts, including:

- Flow run and task run state
- Run history
- Logs
- Deployments
- Flow and task run concurrency limits
- Storage blocks for flow and task results
- Variables
- Artifacts
- Work pool and work queue configuration and status

Currently Prefect supports the following databases:

- SQLite: The default in Prefect, and our recommendation for lightweight, single-server deployments. SQLite requires essentially no setup.
- PostgreSQL: Best for connecting to external databases, but does require additional setup (such as Docker). Prefect uses the [`pg_trgm`](https://www.postgresql.org/docs/current/pgtrgm.html) extension, so it must be installed and enabled.

### Using the database

A local SQLite database is the default database and is configured upon Prefect installation. The database is located at `~/.prefect/prefect.db` by default.

To reset your database, run the CLI command:  

<div class="terminal">
```bash
prefect server database reset -y
```
</div>

This command will clear all data and reapply the schema.

### Database settings

Prefect provides several settings for configuring the database. Here are the default settings:

```bash
PREFECT_API_DATABASE_CONNECTION_URL='sqlite+aiosqlite:///${PREFECT_HOME}/prefect.db'
PREFECT_API_DATABASE_ECHO='False'
PREFECT_API_DATABASE_MIGRATE_ON_START='True'
PREFECT_API_DATABASE_PASSWORD='None'
```

You can save a setting to your active Prefect profile with `prefect config set`.


### Configuring a PostgreSQL database

To connect Prefect to a PostgreSQL database, you can set the following environment variable:

<div class="terminal">
```bash
prefect config set PREFECT_API_DATABASE_CONNECTION_URL="postgresql+asyncpg://postgres:yourTopSecretPassword@localhost:5432/prefect"
```
</div>

The above environment variable assumes that:

- You have a username called `postgres`
- Your password is set to `yourTopSecretPassword`
- Your database runs on the same host as the Prefect server instance, `localhost`
- You use the default PostgreSQL port `5432`
- Your PostgreSQL instance has a database called `prefect`

If you want to quickly start a PostgreSQL instance that can be used as your Prefect database, you can use the following command that will start a Docker container running PostgreSQL:

<div class="terminal">
```bash
docker run -d --name prefect-postgres -v prefectdb:/var/lib/postgresql/data -p 5432:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=yourTopSecretPassword -e POSTGRES_DB=prefect postgres:latest
```
</div>

The above command:

- Pulls the [latest](https://hub.docker.com/_/postgres?tab=tags) version of the official `postgres` Docker image, which is compatible with Prefect.
- Starts a container with the name `prefect-postgres`.
- Creates a database `prefect` with a user `postgres` and `yourTopSecretPassword` password.
- Mounts the PostgreSQL data to a Docker volume called `prefectdb` to provide persistence if you ever have to restart or rebuild that container.

You can inspect your profile to be sure that the environment variable has been set properly:

<div class="terminal">
```bash
prefect config view --show-sources
```
</div>

Start the Prefect server and it should from now on use your PostgreSQL database instance:

<div class="terminal">
```bash
prefect server start
```
</div>

### In-memory database

One of the benefits of SQLite is in-memory database support. 

To use an in-memory SQLite database, set the following environment variable:

<div class="terminal">
```bash
prefect config set PREFECT_API_DATABASE_CONNECTION_URL="sqlite+aiosqlite:///file::memory:?cache=shared&uri=true&check_same_thread=false"
```
</div>

!!! warning "Use SQLite database for testing only"
    SQLite is only supported by Prefect for testing purposes and is not compatible with multiprocessing.  

### Database versions

The following database versions are required for use with Prefect:

- SQLite 3.24 or newer
- PostgreSQL 13.0 or newer


### Migrations

Prefect uses [Alembic](https://alembic.sqlalchemy.org/en/latest/) to manage database migrations. Alembic is a 
database migration tool for usage with the SQLAlchemy Database Toolkit for Python. Alembic provides a framework for 
generating and applying schema changes to a database.

To apply migrations to your database you can run the following commands:

To upgrade:
<div class="terminal">
```bash
prefect server database upgrade -y
```
</div>
To downgrade:
<div class="terminal">
```bash
prefect server database downgrade -y
```
</div>

You can use the `-r` flag to specify a specific migration version to upgrade or downgrade to. 
For example, to downgrade to the previous migration version you can run:
<div class="terminal">
```bash
prefect server database downgrade -y -r -1
```
</div>
or to downgrade to a specific revision:
<div class="terminal">
```bash
prefect server database downgrade -y -r d20618ce678e
```
</div>


See the [contributing docs](/contributing/overview/#adding-database-migrations) for information on how to create new database migrations.


## Notifications

When you use [Prefect Cloud](/cloud/) you gain access to a hosted platform with Workspace & User controls, Events, and Automations. Prefect Cloud has an option for automation notifications. The more limited Notifications option is provided for the self-hosted Prefect Server.

Notifications enable you to set up alerts that are sent when a flow enters any state you specify. When your flow and task runs changes [state](/concepts/states/), Prefect notes the state change and checks whether the new state matches any notification policies. If it does, a new notification is queued.

Prefect supports sending notifications via:

- Slack message to a channel
- Microsoft Teams message to a channel
- Opsgenie to alerts
- PagerDuty to alerts
- Twilio to phone numbers
- Email (requires your own server)

!!! cloud-ad "Notifications in Prefect Cloud"
    Prefect Cloud uses the robust [Automations](/cloud/automations/) interface to enable notifications related to flow run state changes and work queue health.

### Configure notifications

To configure a notification in Prefect Server, go to the **Notifications** page and select **Create Notification** or the **+** button. 

![Creating a notification in the Prefect UI](/img/ui/create-slack-notification.png)

Notifications are structured just as you would describe them to someone. You can choose:

- Which run states should trigger a notification.
- Tags to filter which flow runs are covered by the notification.
- Whether to send an email, a Slack message, Microsoft Teams message, or other services.

For email notifications (supported on Prefect Cloud only), the configuration requires email addresses to which the message is sent.

For Slack notifications, the configuration requires webhook credentials for your Slack and the channel to which the message is sent.

For example, to get a Slack message if a flow with a `daily-etl` tag fails, the notification will read:

> If a run of any flow with **daily-etl** tag enters a **failed** state, send a notification to **my-slack-webhook**

When the conditions of the notification are triggered, you’ll receive a message:

> The **fuzzy-leopard** run of the **daily-etl** flow entered a **failed** state at **22-06-27 16:21:37 EST**.

On the **Notifications** page you can pause, edit, or delete any configured notification.

![Viewing all configured notifications in the Prefect UI](/img/ui/notifications.png)
