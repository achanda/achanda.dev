+++
title = 'Mapping Between Pgbouncer and Postgres Connection States'
date = 2024-10-27T11:21:48-05:00
draft = false
+++

Pgbouncer is a popular connection pooler for Postgres. Connections from the end user to pgbouncer 
are called client connections from it's point of view. Connections from pgbouncer to Postgres are 
called server connections. In a typical deployment, end users
connect to pgbouncer, which maintains a pool of persistent server connections. The pooler assigns
a server connection to a client connection based on the given pooling mode (that is beyond the scope
of this discussion). Thus, from Postgres' point of view, the client is actually pgbouncer which gets
it's own backend. Hence there is always a 1:1 correspondence between pgbouncer's server connection and
it's backend in postgres. A mental model for their relationship is useful for debugging pgbouncer
related issues. We will start with some definitions to establish that mapping.

Pgbouncer's server connection states are listed [here](https://github.com/pgbouncer/pgbouncer/blob/1759ec7aef1dfd78cd2671dfddb68f94139fb574/include/bouncer.h#L75-L83) 
and are as follows:

* SV_FREE: when a server has been destroyed and all resources related to it has been released
* SV_JUSTFREE: when a server has been marked as disconnected from a client, in this state resources allocated are not yet destroyed
* SV_LOGIN: when pgbouncer is attempting to login to postgres
* SV_BEING_CANCELED: when the client connection associated with a server connection has any pending cancel requests
* SV_IDLE: when a server connection is in the pool and ready to be reused
* SV_ACTIVE: when a server connection is associated with a client connection
* SV_ACTIVE_CANCEL: when pgbouncer has forwarded a cancel request from the connected client (to either postgres or a peer pgbouncer)
* SV_USED: deprecated state
* SV_TESTED: when pgbouncer has sent a test query to the server connection (this query defaults to `SELECT 1`)

Postgres' backend states are listed [here](https://github.com/postgres/postgres/blob/6b652e6ce85a977e4ca7b8cc045cf4f3457b2d7b/src/include/utils/backend_status.h#L24-L32) these are defined as follows:
* STATE_UNDEFINED: an initial state that all backends start from
* STATE_IDLE: when the backend is ready to execute a query
* STATE_RUNNING: when the backend is either running a query (in simple protocol mode) or handling a Parse/Bind/Execute message
* STATE_IDLEINTRANSACTION: when the backend is in an active transaction but not actually running a query
* STATE_FASTPATH: special status for a backend that is executing a [fast path function call](https://www.postgresql.org/docs/7.0/libpq-chapter22724.htm)
* STATE_IDLEINTRANSACTION_ABORTED: when the backend is either in an aborted transaction but not actually running a query
* STATE_DISABLED: when state tracking is disabled

For this discussion, we will ignore Postgres backends that do not serve an external client (archiver, replication etc.). Given this information, 
a mental model for the mapping between pgbouncer server connection states and postgres backend states is as follows:

| Pgbouncer server state    | Postgres backend state                                                                              |
| ------------------------- | ----------------------------------------------------------------------------------------------------|
| SV_FREE                   | STATE_UNDEFINED                                                                                     |
| SV_JUSTFREE               | STATE_IDLE                                                                                          |
| SV_LOGIN                  | STATE_RUNNING                                                                                       |
| SV_BEING_CANCELED         | STATE_RUNNING                                                                                       |
| SV_IDLE                   | STATE_IDLE                                                                                          |
| SV_ACTIVE                 | STATE_RUNNING<br>STATE_IDLEINTRANSACTION<br>STATE_FASTPATH<br>STATE_IDLEINTRANSACTION_ABORTED<br>STATE_DISABLED |
| SV_ACTIVE_CANCEL          | STATE_RUNNING                                                                                       |
| SV_USED                   | STATE_IDLE                                                                                          |
| SV_TESTED                 | STATE_RUNNING                                                                                       |