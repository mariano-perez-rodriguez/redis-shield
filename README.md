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
  -g            try to guess 'redis.call' / 'redis.pcall' appearances
  -o            don't try to guess 'redis.call' / 'redis.pcall' appearances,
                  use patterns only

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

Try to guess where replacements should happen:

```
$ redis-shield -s /opt/redis/redis-server
               -c /opt/redis/redis.conf
               -e /opt/redis/redis-cli
               -g
               *.lua
```

## Description

`redis-shield` is a Bash script intended to provide `EVALSHA`-only mode for Redis. `EVALSHA`-only mode means that the only command available to a Redis client is `EVALSHA`, this allows for setups where the system administrator or developer sets up a number of Lua scripts for execution on the Redis server prividing the _only_ channels by which the application is to access and modify data; think of it as the public interface of the Redis server in this case.

### Requirements

>> **NOTE** Starting from version 0.2, `redis-shield` is able to _guess_ some occurrences of `redis.call` and `redis.pcall`. Now (given the `-g` switch) `redis-shield` will look for patterns of the form:
>>
>> _W_ **redis** _S_ **.** _S_ _P_ **call** _S_ **(** _S_ _Q_ **_TOKEN_** _Q_ _S_ ,
>>
>> where _S_ stands for arbitrary whitespace, _W_ stands for a word boundary, _Q_ is either a single or double quote (both forms are allowed in Lua, `redis-shield` only looks for matching _Q_ s), and _P_ is an optional "P" / "p" (in order to catch `pcall` as well); **_TOKEN_** is any Redis command being looked for.
>>
>> This is the most "generous" regex that Lua would swallow without choking, but it may need further tuning.

In order to make this happen, `redis-shield` requires you to adhere to a simple convention:

> When calling Redis commands inside a Lua script (with `redis.call(...)`), instead of using the command's name (eg. '`SET`'), use a distinctive pattern (eg. '`__SET__`', the default one).

Additionally, every Lua script you care to run in the server _must_ be written to an actual file (ie. if you're merely caling `EVAL` with a constant string, just place that string in a file and apply the convention above). This makes `redis-shield` incompatible with any application that builds its scripts _on-the-fly_, but I find that to be a questionable practice anyway (that is, in the general case, I'm sure there are very valid and interesting reasons to do that every now and then).

Finally, you'll need to implement an abstraction layer of sorts, mapping script names (ie. file names) to `SHA1` hashes and calling that instead of just throwing a command at the Redis connection.

### Worked Example

Suppose you have the following Lua script (`zset2set.lua`):

```lua
--[[

Time complexity: O(N + M) where N is the size of dest (might be 0),
    and M the size of source
Space complexity: O(M) where is the size of source

Convert a ZSET into a SET of its members alone (ie. discard scores).
Note that dest and source may be the same keys, this has the effect of
    removing scores from a ZSET.

USAGE: ZSET2SET dest source

RETURN: number of elements in dest

]]--

-- initialize counter
local i = 0

-- get the source ZSET
local src = redis.call('zrange', KEYS[2], 0, -1)

-- clean up destination SET
redis.call('del', KEYS[1])
-- look for each element in the source
for _, v in pairs(src) do
  -- increment the count
  i = i + 1
  -- add it
  redis.call('sadd', KEYS[1], v)
end

-- return the count
return i
```

This will convert a `ZSET` into the `SET` of its members.

Now, lets choose a _pattern_. We can use whatever we want, but let's stick with the default (ie. `__%s__`) one for now. We'll need to rename every appearence of a Redis command to match:

```lua
--[[

Time complexity: O(N + M) where N is the size of dest (might be 0),
    and M the size of source
Space complexity: O(M) where is the size of source

Convert a ZSET into a SET of its members alone (ie. discard scores).
Note that dest and source may be the same keys, this has the effect of
    removing scores from a ZSET.

USAGE: ZSET2SET dest source

RETURN: number of elements in dest

]]--

-- initialize counter
local i = 0

-- get the source ZSET
local src = redis.call('__ZRANGE__', KEYS[2], 0, -1)

-- clean up destination SET
redis.call('__DEL__', KEYS[1])
-- look for each element in the source
for _, v in pairs(src) do
  -- increment the count
  i = i + 1
  -- add it
  redis.call('__SADD__', KEYS[1], v)
end

-- return the count
return i
```

You're done! Now running `redis-shield` will launch the Redis server, rename _all_ the commands (except for `EVALSHA`), apply the renaming to our file, and load it into the script cache, returning something like:

```
da39a3ee5e6b4b0d3255bfef95601890afd80709 /path/to/zset2set.lua
```

(that's the `SHA1` of the empty string btw :wink: ).

Now it's up to you to pick that output up and perform a mapping like:

Script|Hash
-----------|----
`zset2set`|`da39a3ee5e6b4b0d3255bfef95601890afd80709`

so that your application can make use of it using `EVALSHA`.

Simple, right? :smile:

Not so simple? No problem! Run `redis-shield` with the `-g` (ie. guess) option: it will find the occurrences for you _most of the time_ (as with every other tool, blindly relying on its results without checking them is doomed to failure).

## Rationale

Normally, an application (_your_ application) stands between Redis and the _big bad world_ :registered:. But what if all security mechanisms fail and a malicious attacker gains access to the environment your application run on? It is under that attack scenario that `redis-shield` was designed: to provide a tool to ensure _consistency_, even in the face of an attack that strong. Note that is _consistency_ what is "secured", not the actual contents of the Redis dataset (Redis has no way of telling the attacker apart from your application).

In a normal case, your application will surely have an abstraction layer acting as an interface towards Redis, in order to provide for a "domain specific language" of sorts, so that you can say something along the lines of (`PHP` code):

```php
$user123 = Users::get(123);
$user123->lastLogin = time();
$user123->store();
```

instead of ([`phpredis`](https://github.com/nicolasff/phpredis) syntax):

```php
$redis->hSet('user:123', 'lastLogin', time());
```
the former being more "high-levely" and, thus, closer to the application domain.

What `redis-shield` does is basically provide you with the tools to implement something similar, but directly into Redis: you write the Lua scripts that act as interfaces to your data, and only allow calling them, thus only allowing modification through your (controlled) channels.

Incidentally, if an attacker were to gain access to the Redis pipe (be it a socket or a TCP connection), he would be unable to break the consistency (as imposed by your scripts) of the dataset. Do note though, that he may very well do a plethora of awful things to your data anyway: `redis-shield` is just that, a _shield_, your dataset is shielded from _inconsistency_, not turned into [Sigurd](http://en.wikipedia.org/wiki/Sigurd).
