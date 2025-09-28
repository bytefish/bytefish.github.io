title: Using Postgres Logical Replication for Data Change Events in .NET
date: 2024-04-02 12:26
tags: postgres, dotnet
category: dotnet
slug: postgres_logical_replication_dotnet
author: Philipp Wagner
summary: This article shows how to implement an ASP.NET Core BackgroundService to listen for Logical Replication Events sent by Postgres.

[PostgreSQL NOTIFY command]: https://www.postgresql.org/docs/current/sql-notify.html

In my last post we have seen how to use the [PostgreSQL NOTIFY command] to send Data Change notifications 
to connected client applications. But what happens, if you client application is down and you miss a 
notification? Exactely... it's not sent again.

But PostgreSQL provides a feature called Logical Replication, which was originally developed to keep PostgreSQL 
standby replicas synchronized. PostgreSQL buffers the changes in its Write Ahead Log (WAL) as long as we haven't 
acknowledged them. Unacknowledged Notifications will be re-sent to our application.

The Npgsql documentation writes ...

> Replication allows a client to receive a continuous stream of updates from a PostgreSQL database, providing a 
> near-realtime view of all changes as they occur. While this feature was originally developed to keep PostgreSQL 
> standby replicas in sync with a primary, it can be used by arbitrary client applications.
>
> Replication can be used anywhere where a constant change feed of database changes is required; for example, an external 
> application can be notified in near-realtime of any changes that occurred in a particular database table. This can be 
> useful for external auditing purposes, for replicating certain data somewhere else, for implement the outbox pattern 
> (see Additional resources below), and various other usages.

All code can be found in a Git Repository at:

* [https://github.com/bytefish/npgsql-examples](https://github.com/bytefish/npgsql-examples)

## Table of contents ##

[TOC]

## Configuring Postgres ##

### Enable Logical Replication in the postgres.conf ###

We add a file `postgres/postgres.conf` next to our `docker-compose.yaml`. In the configuration 
we are enabling a logical replication by setting the following properties:

```ini
#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------

# - Settings -
wal_level = logical

#------------------------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------------------------

max_wal_senders = 10
max_replication_slots = 10
```

### Setting the postgres.conf in Docker Compose ###

In the `docker-compose.yaml` we need to mount the `postgres/postgres.conf` in our container and start postgres with the mounted config file.

```yaml
version: '3.8'

networks:
  openfga:

services:
  postgres:
    image: postgres:16
    container_name: postgres
    networks:
      - openfga
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - "./postgres/postgres.conf:/usr/local/etc/postgres/postgres.conf"
    command: "postgres -c config_file=/usr/local/etc/postgres/postgres.conf"
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 5s
      timeout: 5s
      retries: 5
      
```

If you are now running a `docker-compose up` you should have a Postgres 16 instance with Logical Replication enabled.

### Creating a Publication and Replication Slot ###

Now we'll need to create a Publication and Replication Slot. The Publication defines the group of tables in the database you 
wish to replicate. The Replication slot is going to hold the state of the replication stream, and basically buffer the 
messages.

```plsql
DO $$
BEGIN

IF NOT EXISTS (SELECT 1 FROM pg_catalog.pg_publication WHERE pubname = 'gitclub_pub') 
THEN

	CREATE PUBLICATION gitclub_pub FOR TABLE 
		gitclub.user,
		gitclub.organization, 
		gitclub.team,
		gitclub.repository,
		gitclub.issue, 
		gitclub.user_organization_role, 
		gitclub.user_team_role,
		gitclub.user_repository_role,
		gitclub.team_repository_role;

END IF;

IF NOT EXISTS (SELECT 1 from pg_catalog.pg_replication_slots WHERE slot_name = 'gitclub_slot') 
THEN

	-- Replication slot, which will hold the state of the replication stream:
	PERFORM pg_create_logical_replication_slot('gitclub_slot', 'pgoutput');

END IF;
END;

$$ LANGUAGE plpgsql;
```

## .NET Implementation ##

### Data Model ###

The Postgres Logical Replication Protocol and the Data Message Flow is described at:

* [https://www.postgresql.org/docs/current/protocol-logical-replication.html](https://www.postgresql.org/docs/current/protocol-logical-replication.html)

It starts with ...

> The logical replication protocol sends individual transactions one by one. This means that all 
> messages between a pair of Begin and Commit messages belong to the same transaction. [...] Every 
> sent transaction contains zero or more DML messages (Insert, Update, Delete).

So we start by defining a `ReplicationTransaction`, which contains a list of `DataChangeEvent`, that occured inside the transaction:

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

namespace GitClub.Infrastructure.Postgres.Wal.Models
{
    /// <summary>
    /// A Transaction sent by Postgres with all related DataChange Events.
    /// </summary>
    public record ReplicationTransaction
    {
        public List<DataChangeEvent> ReplicationDataEvents { get; } = [];
    }
}
```

> Before the first DML message for a given relation OID, a Relation message will be sent, describing the schema of 
> that relation. [...] Subsequently, a new Relation message will be sent if the relation's definition has changed 
> since the last Relation message was sent for it. (The protocol assumes that the client is capable of remembering 
> this metadata for as many relations as needed.)

So we add a `Relation`, that holds the metadata for a given relation.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

namespace GitClub.Infrastructure.Postgres.Wal.Models
{
    /// <summary>
    /// Postgres send a Relation Message during the Logical Replication.
    /// </summary>
    public record Relation
    {
        /// <summary>
        /// Gets or sets the Id of the Relation.
        /// </summary>
        public required uint RelationId { get; set; }

        /// <summary>
        /// Gets or sets the Namespace.
        /// </summary>
        public required string? Namespace { get; set; }

        /// <summary>
        /// Gets or sets the Name of the Relation.
        /// </summary>
        public required string RelationName { get; set; }

        /// <summary>
        /// Gets or sets the Server Clock time.
        /// </summary>
        public required DateTime ServerClock { get; set; }

        /// <summary>
        /// Gets or sets the Column Names.
        /// </summary>
        public required string[] ColumnNames { get; set; }
    }
}
```

The DML Messages (Insert, Update, Delete) come in various flavors, that depend on your configuration. The 
only thing all of them have in common is the `Relation` they operate on.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

namespace GitClub.Infrastructure.Postgres.Wal.Models
{
    /// <summary>
    /// Emitted, when a change to the Postgres tables occurs.
    /// </summary>
    public abstract record DataChangeEvent
    {
        /// <summary>
        /// Gets or sets the Relation.
        /// </summary>
        public required Relation Relation { get; set; }
    }
}
```

An Insert DML Message sends the new row values, which is decided to model as a `IDictionary<string, object?>`. While 
this is somewhat ugly, I just couldn't come up with a better abstraction and it simplifies the implementation.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

namespace GitClub.Infrastructure.Postgres.Wal.Models
{
    /// <summary>
    /// An insert event includes the new values.
    /// </summary>
    public record InsertDataChangeEvent : DataChangeEvent
    {
        /// <summary>
        /// Gets or sets the new column values.
        /// </summary>
        public required IDictionary<string, object?> NewValues { get; set; }
    }
}
```

You can configure Postgres Logical Replication to either send a DML Update Message, which contains the new row values only.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

namespace GitClub.Infrastructure.Postgres.Wal.Models
{
    /// <summary>
    /// A default update event only contains the new values.
    /// </summary>
    public record DefaultUpdateDataChangeEvent : DataChangeEvent
    {
        public required IDictionary<string, object?> NewValues { get; set; }
    }
}
```

Or you can configure it to send a Full DML Update Message, that includes both, the new and old values.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

namespace GitClub.Infrastructure.Postgres.Wal.Models
{
    /// <summary>
    /// A full update event contains the old and the new values.
    /// </summary>
    public record FullUpdateDataChangeEvent : DataChangeEvent
    {
        /// <summary>
        /// Gets or sets the new column values.
        /// </summary>
        public required IDictionary<string, object?> NewValues { get; set; }

        /// <summary>
        /// Gets or sets the old column values.
        /// </summary>
        public required IDictionary<string, object?> OldValues { get; set; }
    }
}
```

Depending on the configuration Postgres either sends a DML Delete Message with the Keys only.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

namespace GitClub.Infrastructure.Postgres.Wal.Models
{
    /// <summary>
    /// A key delete event contains only the keys, that have been deleted.
    /// </summary>
    public record KeyDeleteDataChangeEvent : DataChangeEvent
    {
        /// <summary>
        /// Gets or sets the keys having been deleted.
        /// </summary>
        public required IDictionary<string, object?> Keys { get; set; }
    }
}
```

### Streaming Transactions with their Data Change Events ###

As of now we'll need at least the database to connect to, and the names of the Publication and Replication Slot. For 
lack of a better name I put them into a `PostgresReplicationServiceOptions` class.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

using GitClub.Hosted;

namespace GitClub.Infrastructure.Postgres.Wal
{
    /// <summary>
    /// Options to configure the <see cref="PostgresNotificationService"/>.
    /// </summary>
    public class PostgresReplicationServiceOptions
    {
        /// <summary>
        /// Gets or sets the ConnectionString for the Replication Stream.
        /// </summary>
        public required string ConnectionString { get; set; }

        /// <summary>
        /// Gets or sets the PublicationName the Service is listening to.
        /// </summary>
        public required string PublicationName { get; set; }

        /// <summary>
        /// Gets or sets the ReplicationSlot the Service is listening to.
        /// </summary>
        public required string ReplicationSlotName { get; set; }
    }
}
```

The `PostgresReplicationService` now uses the Npgsql Logical Replication Protocol implementation to stream `ReplicationTransactions` to 
the consumer of the service. There are some things to be desired, but this is a good starting point I think.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

using GitClub.Infrastructure.Logging;
using GitClub.Infrastructure.Postgres.Wal.Models;
using Microsoft.Extensions.Options;
using Npgsql;
using Npgsql.Replication;
using Npgsql.Replication.PgOutput;
using Npgsql.Replication.PgOutput.Messages;
using System.Collections.Concurrent;
using System.Runtime.CompilerServices;

namespace GitClub.Infrastructure.Postgres.Wal
{
    /// <summary>
    /// This Service processes Replication Events published by a Postgres publication.
    /// </summary>
    public class PostgresReplicationService
    {
        /// <summary>
        /// Logger.
        /// </summary>
        private readonly ILogger<PostgresReplicationService> _logger;

        /// <summary>
        /// Options to configure the Wal Receiver.
        /// </summary>
        private readonly PostgresReplicationServiceOptions _options;

        /// <summary>
        /// Creates a new <see cref="PostgresReplicationService" />.
        /// </summary>
        /// <param name="logger">Logger to log messages</param>
        /// <param name="options">Options to configure the service</param>
        public PostgresReplicationService(ILogger<PostgresReplicationService> logger, IOptions<PostgresReplicationServiceOptions> options)
        {
            _logger = logger;
            _options = options.Value;
        }

        /// <summary>
        /// Instructs the server to start the Logical Streaming Replication Protocol (pgoutput logical decoding 
        /// plugin), starting at WAL location walLocation or at the slot's consistent point if walLocation isn't 
        /// specified. The server can reply with an error, for example if the requested section of the WAL has 
        /// already been recycled.
        /// </summary>
        /// <param name="cancellationToken">Cancellation Token to stop the Logical Replication</param>
        /// <returns>Replication Transactions</returns>
        /// <exception cref="InvalidOperationException">Thrown when a replication message can't be handled</exception>
        public async IAsyncEnumerable<ReplicationTransaction> StartReplicationAsync([EnumeratorCancellation] CancellationToken cancellationToken)
        {
            _logger.TraceMethodEntry();

            // Connection to subscribe to the logical replication slot. We are 
            // using NodaTime, but LogicalReplicationConnection has no TypeMappers, 
            // so we need to add them globally...
#pragma warning disable CS0618 // Type or member is obsolete
            NpgsqlConnection.GlobalTypeMapper.UseNodaTime();
#pragma warning restore CS0618 // Type or member is obsolete

            // This is the only way to create the Replication Connection, I have found no 
            // way to utilize the NpgsqlDataSource for it. There might be a way though.
            var replicationConnection = new LogicalReplicationConnection(_options.ConnectionString);

            // Open the Connection.
            await replicationConnection
                .Open(cancellationToken)
                .ConfigureAwait(false);

            // Reference to the Publication.
            var replicationPublication = new PgOutputReplicationOptions(_options.PublicationName, protocolVersion: 1, binary: true);

            // Reference to the Replication Slot.
            var replicationSlot = new PgOutputReplicationSlot(_options.ReplicationSlotName);

            // Postgres expects us to cache all relations.
            var relations = new ConcurrentDictionary<uint, Relation>();

            // The current transaction, which will be set, when we receive the first commit message.
            ReplicationTransaction transaction = null!;

            await foreach (var message in replicationConnection
                .StartReplication(replicationSlot, replicationPublication, cancellationToken)
                .ConfigureAwait(false))
            {
                _logger.LogDebug("Received Postgres WAL Message (Type = {WalMessageType}, ServerClock = {WalServerClock}, WalStart = {WalStart}, WalEnd = {WalEnd})",
                    message.GetType().Name, message.ServerClock, message.WalStart, message.WalEnd);

                if (message is BeginMessage beginMessage)
                {
                    transaction = new ReplicationTransaction();
                }
                else if (message is CommitMessage commitMessage)
                {
                    yield return transaction;
                }
                else if (message is RelationMessage relationMessage)
                {
                    relations[relationMessage.RelationId] = new Relation
                    {
                        RelationId = relationMessage.RelationId,
                        Namespace = relationMessage.Namespace,
                        RelationName = relationMessage.RelationName,
                        ServerClock = relationMessage.ServerClock,
                        ColumnNames = relationMessage.Columns
                            .Select(x => x.ColumnName)
                            .ToArray()
                    };
                }
                else if (message is InsertMessage insertMessage)
                {
                    var relation = relations[insertMessage.Relation.RelationId];

                    transaction.ReplicationDataEvents.Add(new InsertDataChangeEvent
                    {
                        Relation = relation,
                        NewValues = await ReadColumnValuesAsync(relation, insertMessage.NewRow, cancellationToken).ConfigureAwait(false)
                    });
                }
                else if (message is DefaultUpdateMessage defaultUpdateMessage)
                {
                    var relation = relations[defaultUpdateMessage.Relation.RelationId];

                    transaction.ReplicationDataEvents.Add(new DefaultUpdateDataChangeEvent
                    {
                        Relation = relation,
                        NewValues = await ReadColumnValuesAsync(relation, defaultUpdateMessage.NewRow, cancellationToken).ConfigureAwait(false),
                    });
                }
                else if (message is FullUpdateMessage fullUpdateMessage)
                {
                    var relation = relations[fullUpdateMessage.Relation.RelationId];

                    transaction.ReplicationDataEvents.Add(new FullUpdateDataChangeEvent
                    {
                        Relation = relation,
                        NewValues = await ReadColumnValuesAsync(relation, fullUpdateMessage.NewRow, cancellationToken).ConfigureAwait(false),
                        OldValues = await ReadColumnValuesAsync(relation, fullUpdateMessage.OldRow, cancellationToken).ConfigureAwait(false)
                    });
                }
                else if (message is KeyDeleteMessage keyDeleteMessage)
                {
                    var relation = relations[keyDeleteMessage.Relation.RelationId];

                    transaction.ReplicationDataEvents.Add(new KeyDeleteDataChangeEvent
                    {
                        Relation = relation,
                        Keys = await ReadColumnValuesAsync(relation, keyDeleteMessage.Key, cancellationToken).ConfigureAwait(false)
                    });
                }
                else if (message is FullDeleteMessage fullDeleteMessage)
                {
                    var relation = relations[fullDeleteMessage.Relation.RelationId];

                    transaction.ReplicationDataEvents.Add(new FullDeleteDataChangeEvent
                    {
                        Relation = relation,
                        OldValues = await ReadColumnValuesAsync(relation, fullDeleteMessage.OldRow, cancellationToken).ConfigureAwait(false)
                    });
                }
                else
                {
                    // We don't know what to do here and everything we could do... feels wrong. Throw 
                    // up to the consumer and let them handle the problem.
                    throw new InvalidOperationException($"Could not handle Message Type {message.GetType().Name}");
                }

                // Acknowledge the message.
                replicationConnection.SetReplicationStatus(message.WalEnd);
            }
        }

        private async ValueTask<IDictionary<string, object?>> ReadColumnValuesAsync(Relation relation, ReplicationTuple replicationTuple, CancellationToken cancellationToken)
        {
            _logger.TraceMethodEntry();

            var results = new ConcurrentDictionary<string, object?>();

            // We need to track the current Column:
            int columnIdx = 0;

            // Each "ReplicationTuple" consists of multiple "ReplicationValues", that we could iterate over.
            await foreach (var replicationValue in replicationTuple)
            {
                // These "ReplicationValues" do not carry the column name, so we resolve the column name
                // from the associated relation. This is going to throw, if we cannot find the column name, 
                // but it should throw... because it is exceptional.
                var column = relation.ColumnNames[columnIdx];

                // Get the column value and let Npgsql decide, how to map the value. You could register
                // type mappers for the LogicalReplicationConnection, so you can also automagically map 
                // unknown types.
                //
                // This is going to throw, if Npgsql fails to read the values.
                var value = await replicationValue.Get(cancellationToken).ConfigureAwait(false);

                // If we fail to add the value to the Results, there is not much we can do. Log it 
                // and go ahead.
                if (!results.TryAdd(column, value))
                {
                    _logger.LogInformation("Failed to map ReplicationValue for Column {ColumnName}", column);
                }

                // Process next column
                columnIdx++;
            }

            return results;
        }
    }
}
```

What's left is configuring the `PostgresReplicationService` in the `Program.cs`:

```csharp
// Configures the Postgres Replication Settings.
builder.Services.Configure<PostgresReplicationServiceOptions>(o =>
{
	var connectionString = builder.Configuration.GetConnectionString("ApplicationDatabase")!;

	o.ConnectionString = connectionString;
	o.PublicationName = "gitclub_pub";
	o.ReplicationSlotName = "gitclub_slot";
});

builder.Services.AddSingleton<PostgresReplicationService>();
```


### A BackgroundService to process Data Change Events ###

It's now rather simple to use the `PostgresReplicationService`. As an example I am implementing a `PostgresReplicationListener`, 
that is a `BackgroundService` and logs all Data Change Events as JSON. Again, you can omit the whole NodaTime configuration, if 
you aren't using the Noda Time library.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

// ...

namespace GitClub.Hosted
{
    /// <summary>
    /// This Listener waits for Data Change Events sent by Logical Replication.
    /// </summary>
    public class PostgresReplicationListener : BackgroundService
    {
        private readonly ILogger<PostgresReplicationListener> _logger;

        private readonly PostgresReplicationService _replicationService;

        public PostgresReplicationListener(ILogger<PostgresReplicationListener> logger, PostgresReplicationService replicationService)
        {
            _logger = logger;
            _replicationService = replicationService;
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.TraceMethodEntry();
            
            var jsonSerializerOptions = GetSerializerOptions();

            await foreach(var transaction in _replicationService.StartReplicationAsync(stoppingToken))
            {
                // Process the Received Transaction
                _logger.LogInformation("Received Transaction: {Transaction}", JsonSerializer.Serialize(transaction, jsonSerializerOptions));
            }
        }

        private JsonSerializerOptions GetSerializerOptions()
        {
            var options = new JsonSerializerOptions()
            {
                WriteIndented = true
            };

            return options.ConfigureForNodaTime(DateTimeZoneProviders.Tzdb);
        }
    }
}
```

### Running through an Example ###

In the `GitClub` application I have a `TeamService`, that's responsible for creating a `Team`. It writes 
a `Team` and a `UserTeamRole` to the database, which means the user creating the team is also set as the 
maintainer of the team.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

// ...

namespace GitClub.Services
{
    public class TeamService
    {
        private readonly ILogger<TeamService> _logger;

        private readonly ApplicationDbContext _applicationDbContext;
        private readonly AclService _aclService;

        public TeamService(ILogger<TeamService> logger, ApplicationDbContext applicationDbContext, AclService aclService)
        {
            _logger = logger;
            _applicationDbContext = applicationDbContext;
            _aclService = aclService;
        }

        public async Task<Team> CreateTeamAsync(Team team, CurrentUser currentUser, CancellationToken cancellationToken)
        {
            _logger.TraceMethodEntry();

            // ...
            
            // Add the new Team
            await _applicationDbContext
                .AddAsync(team, cancellationToken)
                .ConfigureAwait(false);

            // The creator is also the Maintainer
            var userTeamRole = new UserTeamRole
            {
                TeamId = team.Id,
                UserId = currentUser.UserId,
                Role = TeamRoleEnum.Maintainer,
                LastEditedBy = currentUser.UserId
            };

            await _applicationDbContext
                .AddAsync(userTeamRole, cancellationToken)
                .ConfigureAwait(false);

            await _applicationDbContext
                .SaveChangesAsync(cancellationToken)
                .ConfigureAwait(false);

            // ...

            return team;
        }
        
        // ...
    }
}
```

And if we now `POST` the required data to the `/Teams` Controller, that forwards the data to the `TeamService`:

```
POST {{GitClub_HostAddress}}/Teams
Content-Type: application/json

{
    "name": "Rockstar Developers #2",
    "organizationId": 1,
    "lastEditedBy": 1
}
```

I have enabled sensitive data logging, so we can see how EntityFramework Core inserts the data and we can see the `ReplicationTransaction` 
sent to us by Postgres, with all associated DML messages written to the logs as JSON:

```
[13:38:20 INF] Executed DbCommand (22ms) [Parameters=[@p0='38199', @p1='1', @p2='Rockstar Developers #2' (Nullable = false), @p3='1', @p4='38199', @p5='1', @p6='2', @p7='38199', @p8='1'], CommandType='Text', CommandTimeout='30']
INSERT INTO gitclub.team (team_id, last_edited_by, name, organization_id)
VALUES (@p0, @p1, @p2, @p3)
RETURNING xmin, sys_period;
INSERT INTO gitclub.user_team_role (user_team_role_id, last_edited_by, team_role_id, team_id, user_id)
VALUES (@p4, @p5, @p6, @p7, @p8)
RETURNING xmin, sys_period;
...
[13:38:21 INF] Received Transaction: {
  "ReplicationDataEvents": [
    {
      "$type": "insert",
      "NewValues": {
        "sys_period": {
          "Start": "2024-04-02T05:38:20.961471Z"
        },
        "last_edited_by": 1,
        "team_id": 38199,
        "name": "Rockstar Developers #2",
        "organization_id": 1
      },
      "Relation": {
        "RelationId": 16480,
        "Namespace": "gitclub",
        "RelationName": "team",
        "ServerClock": "2024-04-02T05:38:21.012045Z",
        "ColumnNames": [
          "team_id",
          "organization_id",
          "name",
          "last_edited_by",
          "sys_period"
        ]
      }
    },
    {
      "$type": "insert",
      "NewValues": {
        "sys_period": {
          "Start": "2024-04-02T05:38:20.961471Z"
        },
        "last_edited_by": 1,
        "user_id": 1,
        "team_role_id": 2,
        "team_id": 38199,
        "user_team_role_id": 38199
      },
      "Relation": {
        "RelationId": 16572,
        "Namespace": "gitclub",
        "RelationName": "user_team_role",
        "ServerClock": "2024-04-02T05:38:21.012224Z",
        "ColumnNames": [
          "user_team_role_id",
          "user_id",
          "team_id",
          "team_role_id",
          "last_edited_by",
          "sys_period"
        ]
      }
    }
  ]
}
```

A DML Delete Message with only the keys looks like this. We can see, that only the primary key properties are populated and all other columns are empty.

```
[13:45:35 INF] Received Transaction: {
  "ReplicationDataEvents": [
    {
      "$type": "key_delete",
      "Keys": {
        "sys_period": {},
        "last_edited_by": {},
        "repository_role_id": {},
        "repository_id": {},
        "team_repository_role_id": 38187,
        "team_id": {}
      },
      "Relation": {
        "RelationId": 16630,
        "Namespace": "gitclub",
        "RelationName": "team_repository_role",
        "ServerClock": "2024-04-02T05:42:09.120532Z",
        "ColumnNames": [
          "team_repository_role_id",
          "team_id",
          "repository_id",
          "repository_role_id",
          "last_edited_by",
          "sys_period"
        ]
      }
    }
  ]
}
```


## Conclusion ##

And that's it! 

In this article we have seen how easy it is to enable Logical Replication in PostgreSQL. By using Npgsql it took 
just a few lines of code to receive the DML messages in our .NET application. Some things are to be desired, such 
as better error handling, but I think it's a good start.

Enjoy your Data Change Events flowing in!
