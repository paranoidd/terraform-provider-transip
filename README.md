# Terraform Transip provider

[![Build Status](https://travis-ci.org/aequitas/terraform-provider-transip.svg?branch=master)](https://travis-ci.org/aequitas/terraform-provider-transip)

Provides resources for Transip resources using [Transip API](https://www.transip.eu/transip/api/)

Supported resources:

    - Domain name (data source, resource)
    - Domain name DNS records (resource)
    - VPS (data source, resource)

## Requirements

In order to use the provider you need a Transip account. For this account the API should be enabled and a private key should be created which is used for authentication (https://www.transip.eu/cp/account/api/).

## Installation

Download the latest binary release from the [Releases](https://github.com/aequitas/terraform-provider-transip/releases) page, unzip it to a location in `PATH` (eg: `/usr/local/bin/`).

## Notes

- The Transip API managed DNS Entries as a list property of a Domain object. In this implementation I have opted to give DNS entries their own resource `transip_dns_record` to make management more in line with other Terraform DNS Providers.

- Concurrently updating of DNS Entries is currently unstable. To improve reliability run Terraform with `-parallelism=1`.

- Not all resources (especially the VPS resource) have been thoroughly tested. Use with care.

## Example

Also see examples in: [examples/](https://github.com/aequitas/terraform-provider-transip/tree/master/examples).

```hcl
# Enable Transip API, whitelist your IP, create private key and provide it here
provider "transip" {
  account_name = "example"
  private_key  = <<EOF
  -----BEGIN PRIVATE KEY-----
  ...
  -----END PRIVATE KEY-----
  EOF
}

# Or simply leave the provider empty when using the environment variables TRANSIP_ACCOUNT_NAME and TRANSIP_PRIVATE_KEY
# provider "transip" { }

# Get an existing domain as data source
data "transip_domain" "example_com" {
  name = "example.com"
}

# Or create/import a (new) domain name to be managed by Terraform
# resource "transip_domain" "example_com" {
#     name = "example.com"
# }

# Simple CNAME record
resource "transip_dns_record" "www" {
  domain  = data.transip_domain.example_com.id
  name    = "www"
  type    = "CNAME"
  content = ["@"]
}

# VPS Server with setup script and DNS record
resource "transip_vps" "test" {
  name = "example"
  product_name = "vps-bladevps-x1"
  operating_system = "ubuntu-18.04"

  # Script to run to provision the VPS
  install_text = <<EOF
  # install and enable firewall and basic webserver
  apt update
  apt install -yqq ufw nginx
  ufw allow 22/tcp
  ufw allow 80/tcp
  ufw allow 443/tcp
  ufw --force enable
  EOF
}
resource "transip_dns_record" "vps" {
  domain = data.transip_domain.example_com.id
  name   = "vps"
  type   = "A"

  content = [ transip_vps.test.ip_address ]
}

# A record with multiple entries, eg: for round robin DNS
resource "transip_dns_record" "test" {
  domain = data.transip_domain.example_com.id
  name   = "test"
  type   = "A"

  content = [
    "203.0.113.1",
    "203.0.113.2",
  ]
}

# IPv6 record
resource "transip_dns_record" "testv6" {
  domain = data.transip_domain.example_com.id
  name   = "test"
  expire = 300
  type   = "AAAA"

  content = [
    "2001:db8::1",
  ]
}

# Get an existing VPS as datasource
data "transip_vps" "test" {
  name = "example"
}

# Set hostname for VPS using data source
resource "transip_dns_record" "vps" {
  domain = data.transip_domain.example_com.id
  name   = "vps"
  type   = "A"

  content = [data.transip_vps.test.ip_address]
}
resource "transip_dns_record" "vps" {
  domain = data.transip_domain.example_com.id
  name   = "vps"
  type   = "AAAA"

  content = [data.transip_vps.test.ipv6_addresses[0]]
}
```
