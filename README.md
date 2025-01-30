# Running a Local CoreDNS Instance #

How to run a local CoreDNS instance, can be helpful when doing DNS-y things.

## Why? ##

* "/etc/hosts" for the modern world
* selective logging (queries within specific domains)
* filtering a typo-squatted domain
* experimenting/fiddling w/ TXT records
* localhost cookie-separation: logging in to multiple ssh-forwarded pfSense
  instances

* this repo provides basic configuration for a local CoreDNS instance, where
  local means running on the local workstation
  * currently targeted to macOS and its resolver(5) library

### Why CoreDNS? ###

* [CoreDNS](https://coredns.io/) is an easy to use, flexible and lightweight
  DNS _server_
  * "server": it does _not_ by default do recursive resolution or forward
    requests, though there are plugins for both of these features

#### CoreDNS Quick Start ####

* CoreDNS is designed around plugins - every CoreDNS function is implemented
  via a plugin - approx. 30 plugins are included in the standard build, and
  many more optional plugins are available
  * plugins are compiled in, statically - if you need a non-core plugin,
    you'll need to build it yourself
  * <https://coredns.io/plugins/>
  * use `coredns -plugins` to list the installed plugins

* `coredns` uses one main configuration file (default `./Corefile`)
  * plugins may reference other files as needed
  * config may be delegated out to other files and `import`ed; similarly
    "snippets" can be defined and re-used (DRY)

* a single `coredns` instance can handle multiple "servers" (zones on a given
  port) each defined in the one `Corefile`
  * the ordering of servers in the `Corefile`, and plugins within each server,
    is _not_ significant
    * requests are routed to the most specific zone
    * plugins are evaluated in the order defined in [plugin.cfg](https://github.com/coredns/coredns/blob/master/plugin.cfg)
      * a plugin may
        * process a query - coredns sends the response to the client
        * pass it on to the next plugin (optionally with some additional
          data)
        * provide a "fallthrough" response (typically `NXDOMAIN`) - the
          plugin processing continues, subsequent plugins may replace the
          fallthrough
        * if the query remains unprocessed at the end of the plugin chain,
          coredns will send a `SERVFAIL` back to the client

##### Notable Plugins #####

* `hosts`: serve zones from an `/etc/hosts` format file

* `forward`: proxies DNS messages to upstream resolvers, with health checks
  * via UDP, TCP and DNS-over-TLS, or refer to the local `/etc/resolv.conf`

* `reload`: allows automatic reload of a changed `Corefile`
  * some other plugins can similarly watch for changes (e.g. `hosts`)

#### What port? ###

* whilst you could just replace the entirety of your host's local DNS resolver
  with coredns it's preferable to run it in concert with your host's resolver;
  e.g. macOS' `/etc/resolver/` or `systemd-resolved`
* when picking a non-standard port: be aware that mDNS (incl. macOS'
  mDNSResponder) runs on 5353 so don't use that

## How do I get set up? ##

1. install `coredns` using your platform's package manager
1. create a `Corefile` config and any necessary support files
1. point your host's DNS to the new local server
1. profit!

### How do I get set up on macOS? ###

#### Install ####

```shell
brew install coredns
brew services start coredns
```

* installs a launchd service running in the user's GUI (logged in) domain, i.e.
  it'll run when the user logs in on the GUI console
  * creates `~/Library/LaunchAgents/homebrew.mxcl.coredns.plist`
  * to see launchd info (note `gui/` domain, not `user/`)

      ```shell
      launchctl print gui/$(id -u)/homebrew.mxcl.coredns
      ```

* installing the service as root (`sudo brew services ...`) will break
  subsequent brew operations, as (understandably) macOS requires daemons
  installed globally to be owned by the root user: launchd will change
  permissions on the coredns path to root and this in turn will cause trouble
  for brew

  * `WorkingDirectory`: `$(brew --prefix)`
  * stdout/err (i.e. logs): `$(brew --prefix)/var/log/coredns.log`

#### Configure CoreDNS ####

* create a `Corefile` (core config) and any necessary support files

#### Configure macOS' resolver(5) ####

* macOS provides a DNS search strategy that can use multiple DNS resolver
  clients: each client is configured via a file in `/etc/resolver/`
  * see `man 5 resolver`
  * these files are named after the domain, e.g. the file `example.com` would
    contain `nameserver` entry(ies) to direct `example.com` queries to
  * `man 5 resolver` for details
  * changes to the files in `/etc/resolver/` are applied immediately

    e.g.

    ```shell
    echo "nameserver 127.0.0.1" | sudo tee /etc/resolver/localhost
    echo "nameserver 127.0.0.1" | sudo tee /etc/resolver/example.net
    ```

* the in-use DNS configuration can be viewed via the `scutil` tool

```shell
scutil --dns
```

* macOS aggressively caches DNS and does not provide an explicit means to clear
    this cache, however it is flushed on network changes
  * anecdotally, killing the `mDNSResponder` process (which will automatically
    restart) also works

    ```shell
    sudo killall -HUP mDNSResponder
    ```

#### macOS-specific Notes ####

* note on `forward . /etc/resolv.conf`: resolv.conf is maintained by macOS for
  "legacy" services (really, only things that open `/etc/resolv.conf` directly:
  dig/host/nslookup)
  * macOS evidently populates `/etc/resolv.conf` with the first `nameserver`
    in the system config: `scutil --dns` -> resolver #1
  * so beware creating a forwarding loop: a local coredns instance should not
    be used as the #1 system resolver in conjunction with
    `forward . /etc/resolv.conf`

* to use the local coredns instance from within a docker container, set the
  container's `/etc/resolv.conf` to use the address that resolves from
  `host.docker.internal`

  e.g.

  ```shell
  getent hosts host.docker.internal | awk '{print "nameserver " $1}' \
    /etc/resolv.conf~ && mv /etc/resolv.conf~ /etc/resolv.conf
  ```

* caveats:
  * for Linux, the flag `--add-host=host.docker.internal:host-gateway` needs
    to be provided, to create the `host.docker.internal` name

  * unfortunately `podman` does not support this feature due to the way
    networking is set up; it's probably better to create a container coredns
    service and use that from other containers

## Examples ##

* refer to the `Corefile` in this repo for implementation of these examples

### Blacklisting a Domain ###

* avoid a typo-squatted domain by creating an empty server definition (no
  plugins), which will return `SERVFAIL` on any request
  * could return `NXDOMAIN` if desired via the `template` plugin; `SERVFAIL` is
    probably as it's obviously not "normal"

### Separating Cookies ###

* cookie scopes don't incorporate ports, so distinct HTTP services accessed
  via `localhost`/`127.0.0.1` will interfere with each other
  * e.g. multiple ssh-forwarded pfSense instances
  * solution: wildcard `*.localhost` to 127.0.0.1, and refer to each web app
    instance by a unique name, e.g. `https://pfsense1.localhost:8443`,
    `https://pfsense2.localhost:8444` and so on

* for the pfSense, a consistent naming scheme should be used, as these names
  must be defined in (System > Advanced > Admin Access > Alternate Hostnames),
  ref pfSense HTTP Referrer / `HTTP_REFERER` for details
