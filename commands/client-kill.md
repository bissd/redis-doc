The `CLIENT KILL` command closes a given client connection. This command support two formats, the old format:

    CLIENT KILL addr:port

The `ip:port` should match a line returned by the `CLIENT LIST` command (`addr` field).

The new format:

    CLIENT KILL <filter> <value> ... ... <filter> <value>

With the new form it is possible to kill clients by different attributes
instead of killing just by address. The following filters are available:

* `CLIENT KILL ADDR ip:port`. This is exactly the same as the old three-arguments behavior.
* `CLIENT KILL LADDR ip:port`. Kill all clients connected to specified local (bind) address.
* `CLIENT KILL ID client-id`. Allows to kill a client by its unique `ID` field. Client `ID`'s are retrieved using the `CLIENT LIST` command.
* `CLIENT KILL TYPE type`, where *type* is one of `normal`, `master`, `replica` and `pubsub`. This closes the connections of **all the clients** in the specified class. Note that clients blocked into the `MONITOR` command are considered to belong to the `normal` class.
* `CLIENT KILL USER username`. Closes all the connections that are authenticated with the specified [ACL](/topics/acl) username, however it returns an error if the username does not map to an existing ACL user.
* `CLIENT KILL SKIPME yes/no`. By default this option is set to `yes`, that is, the client calling the command will not get killed, however setting this option to `no` will have the effect of also killing the client calling the command.

It is possible to provide multiple filters at the same time. The command will handle multiple filters via logical AND. For example:

    CLIENT KILL addr 127.0.0.1:12345 type pubsub

is valid and will kill only a pubsub client with the specified address. This format containing multiple filters is rarely useful currently.

When the new form is used the command no longer returns `OK` or an error, but instead the number of killed clients, that may be zero.

## CLIENT KILL and Redis Sentinel

Recent versions of Redis Sentinel (Redis 2.8.12 or greater) use CLIENT KILL
in order to kill clients when an instance is reconfigured, in order to
force clients to perform the handshake with one Sentinel again and update
its configuration.

## Notes

Due to the single-threaded nature of Redis, it is not possible to
kill a client connection while it is executing a command. From
the client point of view, the connection can never be closed
in the middle of the execution of a command. However, the client
will notice the connection has been closed only when the
next command is sent (and results in network error).
