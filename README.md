## Resources

* [Nike Streaming Platform Docs](https://docs.platforms.nike.com/nike-streaming/)
* [GEM API Documentation](https://developer.niketech.com/docs/projects/GEM)
* [GEM Source Code](https://github.com/nike-streaming-platform/ems)

## Environments

See [NSP Environments](https://docs.platforms.nike.com/nike-streaming/environments)

## Interfaces

* [SDK](https://sdks.platforms.nike.com/)
* [CLI](https://epctl.platforms.nike.com/)
* [Terraform](https://docs.platforms.nike.com/nike-streaming/clients/terraform)

## Common tasks

### Connecting to the API

#### Okta authentication

See [NSP Authentication](https://docs.platforms.nike.com/nike-streaming/authentication#okta)

### Create or update an HTTP push source

A dedicated HTTP endpoint can be created with the following command:
```bash
curl -X PUT \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    -H 'Content-Type: application/json' \
    https://api.gem-nonprod.platforms.nike.com/v1/sources/<NAME> \
    -d '{
        "settings": {
            "kafka.http.push": {}
        }
    }'
```

In order to obtain the URL to post data to, make a GET request to the same endpoint:
```bash
curl -X GET \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    -H 'Content-Type: application/json' \
    https://api.gem-nonprod.platforms.nike.com/v1/sources/<NAME>
```
The response will contain a `usage` section:
```json
{
    "source": {
      "type": "kafka.http.push",
      "name": "<NAME>",
      "usage": {
        "routes": [
          {
            "method": "POST",
            "url": {
              "internal": "https://internal-host/path",
              "external": "https://external-host/path"
            }
          }
        ]
      }
    }
}
```
where `method` contains the HTTP method to use when sending data to the endpoint at `url` (the internal URL is
available only on the Nike network while the external URL can be accessed from anywhere). If schema support has
been enabled using the `hasSchema` option in the source settings then the URL will not be returned until a
schema has been set for the stream.

See [HTTP Push Usage](https://docs.platforms.nike.com/nike-streaming/sources#http-push-usage)

##### Setting up authorization on the HTTP Push Source

See [HTTP Push Auth](https://docs.platforms.nike.com/nike-streaming/sources#http-push-auth)

##### Subscribe to LLSv2

Just don't do it.

### Create or update an OracleDB source

Data can be sourced from an OracleDB database with the following command:
```bash
curl -X PUT \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    -H 'Content-Type: application/json' \
    https://api.gem-nonprod.platforms.nike.com/v1/sources/<NAME> \
    -d '{
        "settings": {
            "kafka.connect.jdbc.oracledb": {
                "tables": [ "table_to_extract" ],
                "sid": "sid",
                "host": "db host",
                "port": 1521,
                "auth": {
                    "basic": {
                        "user": "db user",
                        "password": "db password"
                    }
                }
            }
        }
    }'
```
The user configured with `user` and `password` must have read access to any tables that should be pulled from the
database.

There are several important options described in the
[API documentation](https://developer.niketech.com/docs/projects/GEM?tab=api) for this source.

First, `tables` and `views` are used to give an explicit list of tables or views to extract from the database.

Second, `sequenceColumn` and `updateColumn` control how data is pulled from tables. `sequenceColumn` should be a
column whose values are purely incremented with each new insertion. This column will be used to determine when new
rows should be pulled from the table. `updateColumn` should be a column whose values are increasing with each
modification to a row; for example, an "updated at" timestamp. If both `sequenceColumn` and `updateColumn` are
specified then inserts as well as updates will be reflected in the stream of data. If pulling multiple tables with
different values of `updateColumn` and `sequenceColumn` then a new source must be created for each.

For large tables it is very important that the columns referenced by `updateColumn` and `sequenceColumn` have indexes.
In order to incrementally pull data from a table, queries will be constructed that filter and sort on these columns.
So for tables with many rows (millions or more) the queries can be very slow to run, causing data to be sourced
just as slowly.

Finally, query can be used to control at a more granular level the way that rows are pulled from the database. If this
is specified then `tables` and `views` should not be set. If this is specified and includes a WHERE clause then
`sequenceColumn` and `updateColumn` should not be set.

#### Current limitations

* Only inserts and possibly updates are pulled from the database. Deletes are not tracked.
* See [OracleDB source documentation](#create-or-update-an-oracledb-source) for a discussion of the request parameters
and limitations of the source connector.

### Create or update a PostgreSQL source

Data can be sourced from an PostgreSQL database with the following command:
```bash
curl -X PUT \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    -H 'Content-Type: application/json' \
    https://api.gem-nonprod.platforms.nike.comv1/sources/<NAME> \
    -d '{
        "settings": {
            "kafka.connect.jdbc.postgres": {
                "tables": [ "table_to_extract" ],
                "database": "db name",
                "host": "db host",
                "port": 1521,
                "auth": {
                    "basic": {
                        "user": "db user",
                        "password": "db password"
                    }
                }
            }
        }
    }'
```

### Create or update an IBM MQ source

Data can be sourced from an IBM MQ database with the following command:
```bash
curl -X PUT \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    -H 'Content-Type: application/json' \
    https://api.gem-nonprod.platforms.nike.com/v1/sources/<NAME> \
    -d '{
        "settings": {
            "kafka.connect.ibmmq": {
                "host": "ibmmq-host",
                "transport": {
                    "client": {
                        "channel": "SYSTEM.DEF.SVRCONN"
                    }
                },
                "destination": {
                    "queue": "my-queue"
                },
                "queueManager": "qm"
            }
        }
    }'
```

### Create or update a sink to S3

See [NSP S3 Sink](https://docs.platforms.nike.com/nike-streaming/sinks#s3)

### Create or update a sink to Snowflake with a JDBC connection

A sink of data to Snowflake from the stream with `source name` and `stream name` can be created using the following
command:
```bash
curl -X PUT \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    -H 'Content-Type: application/json' \
    https://api.gem-nonprod.platforms.nike.com/v1/sinks/<NAME> \
    -d '{
        "source": {
            "name": "source name",
            "stream": "stream name"
        },
        "settings": {
            "kafka.connect.jdbc.snowflake": {
                "table": "table_to_write",
                "database": "db name",
                "host": "nike.snowflakecomputing.com",
                "role": "db role",
                "warehouse": "db warehouse",
                "auth": {
                    "basic": {
                        "user": "db user",
                        "password": "db password"
                    }
                }
            }
        }
    }'
```
The user configured with `user` and `password` must have write access through the input `role` to the input table and
warehouse that should be pulled from the database. At Nike, that user and password should be an application account
as described in the [Snowflake Onboarding Process](https://confluence.nike.com/x/vXn3DQ); this will allow access to
Snowflake without going through MFA. Additional configuration is described in the
[API documentation](https://developer.niketech.com/docs/projects/GEM?tab=api).

#### Current limitations

* A username and password are required for the connection to Snowflake, but these are restricted due to Nike's SSO
policies. Please reach out to your onboarding contacts to set up this configuration.

### Create or update a sink to an HTTP endpoint

See [NSP HTTP Sink](https://docs.platforms.nike.com/nike-streaming/sinks#http)

### Update access to a source or sink

Each source and sink has associated with it two managed roles: an admin role and a read-only role. Individual users
as well as AD groups can be added to either or both roles. The current list of users and groups attached to the roles
for a source can be retrieved with:
```bash
curl -X GET \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    https://api.gem-nonprod.platforms.nike.com/v1/sources/<NAME>/roles
```
and for a sink with:
```bash
curl -X GET \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    https://api.gem-nonprod.platforms.nike.com/v1/sinks/<NAME>/roles
```

The users and groups can be _replaced_ for a source role with the following command (replacing \<ROLE\> with `readOnly`
or `admin` to set read-only and admin groups, respectively):
```bash
curl -X PUT \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    -H 'Content-Type: application/json' \
    https://api.gem-nonprod.platforms.nike.com/v1/sources/<NAME>/roles/<ROLE>/groups \
    -d '{
        "groups": ["user1", "adGroup1"]
    }'
```
Similarly, the groups associated with a sink are _replaced_ with:
```bash
curl -X PUT \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    -H 'Content-Type: application/json' \
    https://api.gem-nonprod.platforms.nike.com/v1/sinks/<NAME>/roles/<ROLE>/groups \
    -d '{
        "groups": ["user1", "adGroup1"]
    }'
```

The following list of permissions are associated with each source role:

| Action | Source admin role | Source read-only role |
|:------:|:-----------------:|:---------------------:|
| Sink data from source | x | x |
| Update source settings | x | |
| Update source description | x | |
| Update source tags | x | |
| Update source schema | x | |
| Update source alarms | x | |
| Delete source | x | |
| Pause source | x | |
| Resume source | x | |
| Restart source | x | |
| Describe source | x | x |
| Describe source settings | x | x |
| Describe source alarms | x | x |
| Describe source roles | x | |
| Update access to source | x | |
| Revoke sink from source | x | |

The following list of permissions are associated with each sink role:

| Action | Sink admin role | Sink read-only role |
|:------:|:---------------:|:-------------------:|
| Update sink settings | x | |
| Update sink transformers | x | |
| Update sink filter | x | |
| Update sink description | x | |
| Update sink tags | x | |
| Update sink alarms | x | |
| Delete sink | x | |
| Pause sink | x | |
| Resume sink | x | |
| Restart sink | x | |
| Describe sink | x | x |
| Describe sink settings | x | x |
| Describe sink filter | x | x |
| Describe sink alarms | x | x |
| Describe sink roles | x | |
| Update access to sink | x | |
|~|||
| Sink data from dead letter queue | x | x |
| Update dead letter queue settings | x | |
| Update dead letter queue description | x | |
| Update dead letter queue tags | x | |
| Update dead letter queue schema | x | |
| Update dead letter queue alarms | x | |
| Delete dead letter queue | x | |
| Pause dead letter queue | x | |
| Resume dead letter queue | x | |
| Restart dead letter queue | x | |
| Describe dead letter queue | x | x |
| Describe dead letter queue settings | x | x |
| Describe dead letter queue alarms | x | x |
| Describe dead letter queue roles | x | |
| Update access to dead letter queue | x | |
| Revoke sink from dead letter queue | x | |

\* Even though we list "Describe source" and "Describe sink" as permissions associated with each role, GEM currently
lets all users describe all sources and sinks.

#### Current limitation

* A user's AD groups are synchronized with the LDAP server every 15 minutes, so it can take up to this length of time
after a user is added to an AD group until that access is reflected in GEM.

### List sources and sinks

All sources, streams, and their schema can be listed using:
```bash
curl -X POST \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    https://api.gem-nonprod.platforms.nike.com/v1/sources/filters
```

All sinks can be listed using:
```bash
curl -X POST \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    https://api.gem-nonprod.platforms.nike.com/v1/sinks/filters
```

### Create or update alarms

The GEM API allows users to register alarms on a source or sink so that they can be alerted when the state of the alarm changes.

Users may be notified if an source/sink's state changes to one of the following: "error", "degraded", "running", "paused", "initializing", or "missingSchema".

Current configuration options allow users to be alarmed via a Slack Channel or a PagerDuty Integration.

#### Slack Channel Alarms
To configure an alarm that notifies a Slack Channel, a request must be submitted to configure the Incoming WebHooks Slack app.

1. To do this, navigate to the apps directory of your workspace, then select "Manage" in the top navbar, then "Custom Integrations" in the left sidebar.
2. "Incoming Webhooks" can be found in the list of integrations. Select it, then select "Request Configuration". A request will be submitted, and after a few moments the Slackbot should notify you that your request to install Incoming Webhooks has been approved. Go back to the App Directory and select Incoming Webhooks again. Then select "Add Configuration", and a new configuration form will appear.
3. Choose a channel or create a new one, and some setup instructions should appear.
4. Copy the "Webhook URL" here, as it will be used to configure your GEM alarm.
5. Now, a Slack alarm can be created using:
```bash
curl -X PUT \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    -H 'Content-Type: application/json' \
    https://api.gem-nonprod.platforms.nike.com/v1/sinks/<NAME>/alarms \
    -d '{
            "alarms": [{
                "events": {"running": true, "paused": true, "error": true},
                "resource": {
                    "type": "slack",
                    "slack": {
                        "webhookURL": "webhook url",
                        "channel": "#channel-name"
                    }
                }
            }]
        }'
```

#### PagerDuty Alarms
To configure an alarm that creates a PagerDuty Event, an Integration must be added to your PagerDuty service.

1. Navigate to your PagerDuty service, then select the "Integrations" tab, and then the "+ New Integration" button.
2. Name your integration, then select "Use our API directly". There will be an option to select which Events API version to use. Depending on your PagerDuty service's configuration, one or both of these options may be available. The GEM API accepts either option, so feel free to select either option that is available and then select the "Add Integration" button.
3. Your integration will now appear in the Integrations tab of your PagerDuty service. Copy the "Integration Key", as it will be used to configure your GEM alarm.
4. Now, a PagerDuty alarm can be created using:
```bash
curl -X PUT \
    -H "x-pretty-print: true" \
    -H "Authorization: Bearer <JWT>" \
    -H 'Content-Type: application/json' \
    https://api.gem-nonprod.platforms.nike.com/v1/sinks/<NAME>/alarms \
    -d '{
            "alarms": [{
                "events": {"running": true, "paused": true, "error": true},
                "resource": {
                    "type": "pagerduty",
                    "pagerduty": {
                        "apiVersion": "v2",
                        "integrationKey": "integration key"
                    }
                }
            }]
        }'
```
