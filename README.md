Redis Shield
============

A Redis wrapper to enable "shielded mode" (AKA: "`EVALSHA`-only mode")

---

## Quick reference

```
Usage: redis-shield OPTION... [FILE]...
Apply command renaming to each Lua FILE, apply rename-command directives
to the given Redis server, and load the renamed scripts; return a list of
space separated hash / FILE pairs.

Mandatory options:
  -s SERVER-SCRIPT   use the given script as the server
  -c CONFIG          use the given file as base configuration
  -e CLI-SCRIPT      use the given script as the cli interface

Additional options:
  -m MAP-FILE   write the hash / FILE mapping to the given file
                  instead of stdout
  -n            use newlines as terminators for hash / FILE mapping
  -z            use nulls as terminators for hash / FILE mapping
  -x COMMAND    exclude COMMAND from renaming (this option may be given
                  multiple times)
  -d COMMAND    disable COMMAND, ie. rename to '' (this option may be given
                  multiple times)
  -p PATTERN    use PATTERN as the replacement token pattern, the default
                  pattern is '__%s__'; this must be a printf-friendly
                  format string with a single '%s' occurence and no other
                  format specifier; commands are always given in uppercase
  -b BITS       bit length of the rename tags (will be rounded up to a
                  multiple of 4), defaults to 1024
  -t TIMEOUT    time in seconds to wait for server availability (defaults
                  to 5)

  --            signal the end of command line options

  -h            display this help and exit
  -v            output version information and exit

The SERVER-SCRIPT is any script able to start a Redis server being given
a configuration file, to be read from stdin if '-' is specified instead.

The CLI-SCRIPT is any script able to connect to the Redis server,
recognizing the '-p', '-s', and '-a' options for specifying port, socket,
and authentication; additionally, it should support the '-x' switch
instructing the script to read its last argument from stdin. Note that
redis-cli fulfills the requirements.

Exit status:
 0  if OK,
 1  if command line problems (eg. invalid pattern),
 2  if server timeout reached.
```

### Examples

Redis install under `/opt`:

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               *.lua
```

Only process "template" files:

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               *.lua.template
```

Custom pattern:

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               -p '::%s::'
               *.lua
```

Exclude inocuous commands from renaming (eg. `ping`, `echo`, `time`):

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               -x ping -x echo -x time
               *.lua
```

Disable dangerous commands (eg. `keys`, `config`, `flushall`, `flushdb`, `shutdown`, `debug`):

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               -d keys -d config -d flushall -d flushdb -d shutdown -d debug
               *.lua
```

Both of the above:

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               -x ping -x echo -x time
               -d keys -d config -d flushall -d flushdb -d shutdown -d debug
               *.lua
```

Write script map to a file instead of `stdout`:

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               -m /tmp/map.file
               *.lua
```

Write script map to a file instead of `stdout`, and terminate lines with a null (`\0`) byte:

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               -m /tmp/map.file
               -z
               *.lua
```

Be paranoid about random bits used (viz. use 4096):

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               -b 4096
               *.lua
```

Wait for the server for a whole day (ie. 86400 seconds):

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               -t 86400
               *.lua
```

## Description

`redis-shield` is a Bash script intended to provide `EVALSHA`-only mode for Redis. `EVALSHA`-only mode means that the only command available to a Redis client is `EVALSHA`, this allows for setups where the system administrator or developer sets up a number of Lua scripts for execution on the Redis server prividing the _only_ channels by which the application is to access and modify data; think of it as the public interface of the Redis server in this case.

### Requirements

In order to make this happen, `redis-shield` requires you to adhere to a simple convention:

> When calling Redis commands inside a Lua script (with `redis.call(...)`), instead of using the command's name (eg. '`SET`'), use a distinctive pattern (eg. '`__SET__`', the default one).

Additionally, every Lua script you care to run in the server _must_ be written to an actual file (ie. if you're merely caling `EVAL` with a constant string, just place that string in a file and apply the convention above). This makes `redis-shield` incompatible with any application that builds its scripts _on-the-fly_, but I find that to be a questionable practice anyway (that is, in the general case, I'm sure there are very valid and interesting reasons to do that every now and then).

Finally, you'll need to implement an abstraction layer of sorts, mapping script names (ie. file names) to `SHA1` hashes and calling that instead of just throwing a command at the Redis connection.
