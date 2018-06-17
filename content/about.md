---
title: "Cross-platform DNS Interface"
type: page
menu: {}
---

The Terraform DNS project aims to abstract over the various different DNS
hosting products supported by Terraform so that DNS can be used as a standard
building-block within your infrastructure, decoupled from other concerns.

DNS records always have the same high-level structure, regardless of which
service is hosting them. Some vendors offer additional features on top of
the basic DNS functionality, but there is a common baseline that can be
used across many different providers.

The Terraform DNS project defines a standard way to represent DNS recordsets
and then provides a selection of modules that accept that representation and
map it on to the relevant resource types of a specific provider.

# Recordset Type

The primary convention established by this project is an object type for
representing DNS recordsets in Terraform. Recordset objects have the
following attributes:

- `name` (string): the local name (relative to the containing zone name) for
  the recordset, as will appear in domain names referring to this record set.
  If the fully-qualified recordset name must match the containing domain,
  set the name to the empty string `""`.

- `type` (string): uppercase recordset type keyword, such as `"A"`, `"AAAA"`,
  `"CNAME"`, etc.

- `ttl` (number): whole number of seconds that caching resolvers may retain
  records from this recordset.

- `records` (list of strings): record data in a string format which depends on
  the recordset type.

For example, the following literal Terraform language expression is a valid
recordset object:

```hcl
{
  name    = "www"
  type    = "CNAME"
  ttl     = 3600
  records = [
    "webserver01",
    "webserver02",
    "webserver03",
  ]
}
```

The strings in `records`, as noted above, vary in format depending on the
record type:

- `A`: an IPv4 address in the conventional dotted-decimal syntax, like `10.0.2.32`.
- `AAAA`: an IPv6 address in the conventional colon-separated hexadecimal syntax.
- `CNAME`: either another relative name within the same zone, or a fully-qualified
  domain name ending with a period `.`. Hostnames must always be normalized to
  lowercase.
- `NS`: as with `CNAME`.
- `PTR`: as with `CNAME`.
- `TXT`: a sequence of characters surrounded by quotes. This means that a
  literal text record must include escaped quotes, like `"\"v=spf1\""`.
- `MX`: the MX record priority value as decimal digits, followed by exactly one
  space and then a hostname following the same conventions as for `CNAME`.
- `SRV`: the SRV record priority value as decimal digits, followed by exactly
  one space and then the record's weight value as decimal digits, followed by
  exactly one space and then the record's port number as decimal digits,
  followed by exactly one more space and then a hostname following the same
  conventions as for `CNAME`.

Depending on implementation details of the underlying providers, some DNS
recordset modules may accept alternative representations, but the
representations given above are required for consistent support across all
modules.

Due to limitations of the underlying services and Terraform providers, some
modules can accept only a subset of these record types, as described in their
respective README files.

Many DNS services treat the `NS` records for the domain as special, either
not allowing them to be customized at all or requiring more specialized
resources to be used. For this reason, using this format to represent
`NS` records with an empty `name` value is not recommended. Instead, configure
these records as part of creating the zone itself, using a vendor-specific
resource type directly.

## Recordset Module Interface

This project includes a number of _recordset modules_, which are normal
Terraform modules that accept lists of recordsets in the above format and
map them onto real DNS records in a specific underlying system.

The modules can be browsed via [the Terraform Registry index](https://registry.terraform.io/modules/terraformdns).

A recordset module must always accept a variable called `recordsets` which
is defined as follows:

```hcl
variable "recordsets" {
  type = list(object({
    name    = string
    type    = string
    ttl     = number
    records = list(string)
  }))
  description = "List of DNS recordset objects to manage, in the standard terraformdns structure."
}
```

A recordset module will usually declare one or more other provider-specific
input variables to identify the DNS zone in a vendor-specific way. When using
these modules, the user will set these selector arguments along with the list
of recordsets to create. For example, for the AWS Route53 module:

```hcl
module "dns_records" {
  source  = "terraformdns/route53-recordsets/aws"
  version = "..."

  route53_zone_id = aws_route53_zone.example.id

  recordsets = [
    {
      name    = "www"
      type    = "CNAME"
      ttl     = 3600
      records = [
        "webserver01",
        "webserver02",
        "webserver03",
      ]
    },
  ]
}
```

Each module will then perform transform operations on the list of recordsets
to properly configure instances of provider-specific resource types that
correspond to the requested recordsets.

## Decoupled Modular Infrastructure

A standard representation of DNS recordsets allows Terraform users to write
re-usable Terraform modules that require DNS records to exist while keeping
those modules agnostic to the DNS provider in use.

For example, a hypothetical `example-app` module might export a
`dns_recordsets` output value that conforms to the above structure, allowing
the calling module to then pass that list onto whichever recordset module
is appropriate for the target environment:

```hcl
module "app" {
  source = "./example-app"
}

module "dns_records" {
  source  = "terraformdns/dns-recordsets/azurerm"
  version = "..."

  resource_group_name = azurerm_resource_group.example.name
  dns_zone_name       = azurerm_dns_zone.example.name
  recordsets          = module.app.dns_recordsets
}
```

This approach allows the definition of the required recordsets to be separated
from their implementation. Since DNS is often a cross-cutting concern, produced
and consumed by many separate components in a modular infrastructure, this
abstraction allows the selection of DNS provider to happen in a central
location where it can more easily be changed later, without rewriting many
separate modules that could otherwise all depend on specific DNS providers.

# Limitations

As with all abstractions, the DNS recordset modules come with some tradeoffs.

By defining a standard interface to all modules, we are forced to reduce the
supported featureset to the lowest common denominator. In many cases this is
a fine compromise for DNS, since DNS is well-standardized throughout the
industry.

However, several DNS hosting vendors provide additional features that are not
part of the standard DNS featureset, and these cannot be accessed via the
recordset modules offered as part of this project.

If you wish to use these features, you must make your own tradeoff between
portability and functionality, and use the underlying provider features
directly in situations where this project's abstraction is not sufficient.
