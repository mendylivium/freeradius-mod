# Dynamic Home Servers in v3

FreeRADIUS has some support for dynamic home servers, with certain
limitations.


## Configuration

The configuration needs to have dynamic home servers enabled, by
editing `proxy.conf`.

```
proxy server {
	...
	dynamic = true
	...
}
```

This configuration item causes the internal data structures to become
thread-safe for updates.  This change means that there will be more
lock contention on the data structures holding home servers.  As a
result, high-load proxy may see slowdowns.

Once dynamic home servers are enabled, they should be placed into
a subdirectory.  FreeRADIUS should be told which subdirectory the
home servers are located in:

```
proxy server {
	...
	dynamic = true
	...
	directory = ${raddb}/home_servers/
}
```

This directory should contain nothing other than definitions for
dynamic home servers.  These definitions are simply normal
`home_server` definitions:

```
home_server example.com {
	...
}
```

Each file in the directory should be named for the home server domain
name.  In the above example, the filename should be
`${raddb}/home_servers/example.com`.  The name of the home server in
the file should be the same as the filename which contains the home
server definition.

Each file in the directory should have one, and only one,
`home_server` definition.

Home servers which use RadSec can `$INCLUDE tls.conf` in this
directory to use a common site-local TLS configuration.  The script
`freeradius-naptr-to-home-server.sh` referenced below assumes that
this file exists.  If you are not using that file, it is safe to just
replace it with an empty file.

### The Control Socket

The virtual server `sites-enabled/control` *must* be enabled for
dynamic home servers to work.  The `radmin` program *must* have
read/write permission in order for dynamic home servers to work.

Please see that `sites-enabled/control` file for information on
configuring that virtual server.


## Starting FreeRADIUS

When FreeRADIUS starts, it will read each file in the
`${raddb}/home_servers/` directory.  The file will parsed in order to
define a dynamic `home_server`.


## Adding a new Home Server

In order to add a new home server while FreeRADIUS is running, simply
add a new `home_server` definition file to the
`${raddb}/home_servers/` directory.

Then, tell FreeRADIUS that the home server is available in a new file:

```
$ radmin -e "add home_server file /path/to/raddb/home_servers/example/com"
```

If all goes well, the home server will be added.  If there are issues,
`radmin` will print a descriptive error.

Once a dynamic home server has been added, it can be used just like
any other home server.


## Deleting a Home Server

Dynamically created home servers can be deleted via `radmin`.  Note
also that dynamic home servers which are loaded when FreeRADIUS starts
can be deleted.

```
$ radmin -e "del home_server file <name> <type>"
```

Note that this command deletes the home server by name and type, not
by filename.  This difference from the `add home_server` command is
due to internal limitations in the server core.

```
home_server <name> {
	type = <type>
}
```


## Listing a Home Server

It is possible to list all home servers and know which is dynamic or no.

```
$ radmin -e "show home_server list all"
```


## Limitations

Note that due to internal limitations, dynamic home servers are _not_
freed.  So repeatedly adding and deleted home servers _will_ cause
FreeRADIUS to gradually use more memory.

Other internal limitations means that it is impossible to add dynamic
home servers to a `home_server_pool`.  In short, dynamic home servers
exist by themselves, with no associated realm, pool, or failover
capability.


## Proxying to a Home Server

The new attribute `Home-Server-Name` controls proxying to a particular
home server.  The home server just has to exist, it does not need to
be a dynamic home server.

```
authorize {
	...

	update control {
		Home-Server-Name := "example.com"
	}
	...
}
```


## Checking if a Dynamic Home Server exists

You can see if a dynamic home server exists through the following
dynamic string expansion:

```
%{home_server_dynamic:name}
```

This expansion looks up the home server by name, and returns whether
or not the home server exists, and is dynamic.

The return values are:

* empty string - the home server does not exist.
* `0` - the home server exists, and is statically defined.
* `1` - the home server exists, and is dynamically defined

```
authorize {
	...
	if (User-Name =~ /@(.*)$/) {
		switch "%{home_server_dynamic:%{1}}" {
			case "1" {
				# Proxy to this one particular home server
				update control {
					&Home-Server-Name := "%{1}"
				}
			}

			case "0" {
				# Proxy with home server pool, failover, etc.
				update control {
					&Proxy-To-Realm := "%{1}"
				}
			}

			case {
				# no home server exists, ask DNS
				update control {
					# you can add a parameter for the NAPTR tag to look up, e.g. "aaa+auth:radius.tls.tcp" (RFC7585, OpenRoaming)
					# if the third parameter is omitted, it defaults to "x-eduroam:radius.tls"
					&Temp-Home-Server-String := `%{config:confdir}/mods-config/realm/freeradius-naptr-to-home-server.sh -d %{config:confdir} %{1}`
				}
				if ("%{control:Temp-Home-Server-String}" == "" ) {
					reject
				} else {
					update control {
						&Home-Server-Name := "%{1}"
					}
				}
			}
		}
	}
	...
}
```

## Maintenance of Dynamic Home Servers

Dynamic home servers are discovered from DNS, and DNS has TTLs. These
TTLs are not tracked by FreeRADIUS, as they are not available when
using the standard DNS APIs.

Dynamic realms should be regularly deleted, so that they can be
recreated with updated information.  The server should be restarted
with an empty home_server directory regularly, for two reasons:

* Entries in DNS may change over time, or be removed, and the server should learn this.
  If the entries are not removed, the server will not discover any changes.
* dynamic home servers are often RADIUS/TLS based with client and server certificates,
  and the server should refresh CRL information regularly

As a result, we recommend emptying the home_servers directory (except
for the `tls.conf` file), refreshing CRLs and then restarting the server
once per day.  e.g.

```
rm -f $(ls -1 raddb/home_servers | egrep -v tls.conf)
```
