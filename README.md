# LANcache
Dynamically Cache Game Installs at LAN’s using [Nginx](http://nginx.org/)

# Requirements
Due to the features used in this configuration it requires [nginx 1.6.0+](http://nginx.org/).

# Nginx Configuration
In order to make the configuration more maintainable we’ve split the config up in to a number of smaller includes.

## machines/lancache-single.conf
In the machines directory you have `lancache-single.conf` which is the main `nginx.conf` file that sets up the events and http handler as well as the key features via includes: custom log format, cache definition and active vhosts.
```nginx
include lancache/log_format;
include lancache/caches;
include vhosts/*.conf;
```
## lancache/log_format
The custom log format adds three additional details to the standard combined log format `“$upstream_cache_status” “$host” “$http_range”`. These are useful for determine the efficiency of each segment the cache.

## lancache/caches
In order to support the expanding number of downloads supported by LANcache we’ve switched the config from static mirror to using nginx’s built in caching.

In our install we’re caching data to 8 x 1TB SSD’s configured in [ZFS](http://open-zfs.org/) RAIDZ2 so we have ~6TB of storage.

To ensure we don’t run out of space we’ve limited the main installs cache size to under 6TB with custom loader details to ensure we can init the cache quicker on restart.
The other cache zone is used for none install data so is limited to just 10GB.

We also set `proxy_temp_path` to a location on the same volume as the cache directories so that temporary files can be moved directly to the cache directory avoid a file copy which would put extra stress on the IO subsystem.
```nginx
proxy_cache_path /data/www/cache/installs levels=2:2 keys_zone=installs:500m inactive=120d max_size=972800m loader_files=1000 loader_sleep=50ms loader_threshold=300ms;
proxy_cache_path /data/www/cache/other levels=2:2 keys_zone=other:100m inactive=72h max_size=10240m;
proxy_temp_path /data/www/cache/tmp;
```
## sites/lancache-single.conf
Here we define individual server entries for each service we’ll be caching, we do this so that each service can configure how its cache works independently.
In order to allow for direct use of the configs in multiple setups without having to edit the config files themselves we made use of named entries for all listen addresses.

The example below shows the origin server entry which listens on `lancache-origin` and requires the spoof entries `akamai.cdn.ea.com`.

We use `server_name` as part of the `cache_key` to avoid cache collisions and so add `_` as the wildcard catch all to ensure all requests to this servers IP are processed.

For performance we configure the access log with buffering.
```nginx
# origin
server {
        listen lancache-origin accept_filter=httpready default;
        server_name origin _;
        # DNS entries:
        # lancache-origin akamai.cdn.ea.com
        access_log /data/www/logs/lancache-origin-access.log main buffer=128k flush=1m;
        error_log /data/www/logs/lancache-origin-error.log;
        include lancache/node-origin;
}
```
The include is where all the custom work is done, in this case `lancache/node-origin`. There are currently 5 different flavours of node: blizzard, default, origin, pass and steam.

## lancache/node-origin
Origin’s CDN is pretty bad in that currently prevents the caching of data, due to this we’re force to ignore the `Cache-Control` and `Expires` headers. The files themselves are very large 10GB+ and the client uses range requests to chunk the downloads to improve performance and provide realistic download restart points.

By default nginx proxy translates a `GET` request with a `Range` request to a full download by stripping the `Range` and `If-Range` request headers from the upstream request. It does this so subsequent range requests can be satisfied from the single download request. Unfortunately Origin’s CDN prevents this so we have to override this default behaviour by passing through the `Range` and `If-Range` headers. This means the upstream will reply with a `206` (partial content) instead of a `200` (OK) response and hence we must add the range to the cache key so that additional requests are correctly.

The final customisation for Origin is to use `$uri` in the `proxy_cache_key`, we do this as the Origin client uses a query string parameter `sauth=<key>`

## lancache/node-blizzard
Blizzard have large downloads too, so to ensure that requests are served quickly we cache `206` responses in the same way as for Origin.

## lancache/node-steam
All Steam downloads are located under `/depot/` so we have a custom location for that which ignores the `Expires` header as Steam sets a default `Expires` header.

We also store the content of requests `/serverlists/` as these requests served by `cs.steampowered.com` give us information about hosts used by Steam to process download request. The information in these files could help identify future DNS entries which need spoofing.

Finally the catch all / entry caches all other items according to the their headers.

## lancache/node-default
This is the default which is used for riot, hirez and sony it uses standard caching rules which caches based on the `proxy_cache_key "$server_name$request_uri"`

# Required DNS entries
All of the required DNS entries are for each service are documented their block server in `vhosts/lancache-single.conf`.

You’ll notice that each entry starts with `lancache-XXXX` this is entry used in the listen directive so no editing of the config is required for IP allocation to each service. As we’re creating multiple server entries and each is capturing hostnames using the `_` wildcard each service must have its own IP e.g.
```
lancache-steam = 10.10.100.100
lancache-riot = 10.10.100.101
lancache-blizzard = 10.10.100.102
lancache-hirez = 10.10.100.103
lancache-origin = 10.10.100.104
lancache-sony = 10.10.100.105
```
# LANcache @ [Insomnia 55](https://insomniagamingfestival.com/)
## Hardware Specification
* Dual Hex Core CPU’s
* 128GB RAM
* 8 x 1TB SSD’s ZFS RAIDZ2
* 2 x 1Gbps + 1 x 10Gbps NIC's
* OS – [FreeBSD 10.1](https://www.freebsd.org/)

## Stats
For those that are interested in stats @ [Insomnia 55](https://insomniagamingfestival.com/) LANcache 
* Processed 12.3 million downloads from the internet totalling 4.8TB
* Served 105.1 million downloads totalling 44.7TB to the LAN
* Peaked at 8Gbps to the LAN

For more information and discussion see [our blog](http://blog.multiplay.co.uk/2014/04/lancache-dynamically-caching-game-installs-at-lans-using-nginx/).
