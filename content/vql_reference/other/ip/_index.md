---
title: ip
index: true
noTitle: true
no_edit: true
---



<div class="vql_item"></div>


## ip
<span class='vql_type label label-warning pull-right page-header'>Function</span>



<div class="vqlargs"></div>

Arg | Description | Type
----|-------------|-----
parse|Parse the IP as an IPv4 or IPv6 address.|string
netaddr4_le|A network order IPv4 address (as little endian).|int64
netaddr4_be|A network order IPv4 address (as big endian).|int64

### Description

Format an IP address.

Converts an ip address encoded in various ways. If the IP address is encoded
as 32 bit integer we can use netaddr4_le or netaddr4_be to print it in a
human readable way.

This function wraps the Golang net.IP library ( https://pkg.go.dev/net#IP ).
This makes it easy to deal with various IP address notations.

Returns an object of type `net.IP`, not a string. If you need to compare the
result of this function to an IP string then you should apply the `.String`
method to the result.

### Examples

1. Parse IPv4-mapped IPv6 addresses

```vql
SELECT ip(parse='0:0:0:0:0:FFFF:129.144.52.38') FROM scope()
```

Will return the string `129.144.52.38`

2. Get information about IP addresses

VQL will also expose the following attributes of the IP address:

- `IsGlobalUnicast`
- `IsInterfaceLocalMulticast`
- `IsLinkLocalMulticast`
- `IsLinkLocalUnicast`
- `IsLoopback`
- `IsMulticast`
- `IsPrivate`

```vql
SELECT ip(parse='192.168.1.2').IsPrivate FROM scope()
```

Returns "true" since this is a private address block.

### See also

- [cidr_contains]({{< ref "/vql_reference/other/cidr_contains/" >}}):
  Calculates if an IP address falls within a range of CIDR specified networks.
- [geoip]({{< ref "/vql_reference/other/geoip/" >}}): Lookup an IP Address
  using the MaxMind GeoIP database.


