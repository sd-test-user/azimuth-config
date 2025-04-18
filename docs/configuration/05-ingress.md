# Ingress

As mentioned in the prerequisites, Azimuth and Zenith expect to be given control of entire
subdomain, e.g. `*.apps.example.org` and this domain must be assigned to a pre-allocated floating
IP using a wildcard DNS entry.

To tell `azimuth-ops` what domain it should use, simply set the following variable:

```yaml  title="environments/my-site/inventory/group_vars/all/variables.yml"
ingress_base_domain: apps.example.org
```

This will result in `azimuth-ops` using `portal.apps.example.org` for Azimuth and
`registrar.apps.example.org` for the Zenith registrar. If Harbor is enabled,
`registry.apps.example.org` will be used for the Harbor registry. Zenith will use domains
of the form `<random subdomain>.apps.example.org` for its services.

## Transport Layer Security (TLS)

TLS for Azimuth can be configured in two ways:

### Pre-existing wildcard certificate

If you have a pre-existing **wildcard** TLS certificate issued for the ingress base domain,
you can use this to provide TLS for Azimuth and Zenith services.

!!! tip

    At the time of writing this is the recommended mechanism for production deployments,
    despite the lack of automation, for two reasons:
    
      * It is not affected by [rate limits](https://letsencrypt.org/docs/rate-limits/)
      * Zenith services become available faster as it avoids the overhead of obtaining
        a certificate per service

!!! danger

    It is your responsibility to check for the expiry of the certificate and renew it.

    Consider using an external service to notify you when the certificate is approaching
    its expiry date.

To configure a pre-existing wildcard certificate for ingress, just create the following
files in your environment:

`environments/my-site/tls/tls.crt`
: Must contain the **full certificate chain**, with the most specific certificate at the
  top (e.g. the wildcard certificate), any intermediate certificates and the root CA at
  the bottom.

`environments/my-site/tls/tls.key`
: The corresponding private key for the wildcard certificate.

!!! danger

    The TLS key should be kept secret. If you want to keep it in Git - which is recommended -
    then it [must be encrypted](../repository/secrets.md).

When these files exist in an environment, `azimuth-ops` will automatically pick them up
and use them.

!!! info

    In the future, support for a wildcard certificate managed by
    [cert-manager](https://cert-manager.io/) may be implemented. However this will require
    the use of a
    [supported DNS provider](https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers)
    in order to fulfil the DNS01 challenge (wildcard certificates cannot be issued for an
    HTTP01 challenge).

### Automated with cert-manager

`azimuth-ops` is able to configure `Ingress` resources for Azimuth and Zenith services so
that their TLS certificates are managed automatically using [cert-manager](https://cert-manager.io/).

In this configuration, a certificate is issued *for each separate subdomain* by an
[ACME provider](https://en.wikipedia.org/wiki/Automatic_Certificate_Management_Environment)
using the
[HTTP-01 challenge type](https://letsencrypt.org/docs/challenge-types/#http-01-challenge).

By default, cert-manager is enabled and this mechanism will be used to issue TLS certificates
for Azimuth and Zenith services with no further configuration. The default ACME service is
[Let's Encrypt](https://letsencrypt.org/), which issues certificates that are trusted by
all major operating systems and browsers.

!!! danger  "Let's Encrypt rate limits"

    Let's Encrypt imposes [rate limits](https://letsencrypt.org/docs/rate-limits/) to ensure
    fair usage. At the time of writing, **the number of new certificates that can be issued
    is 50 per week per registed domain**. The "registered domain" is the part of the domain
    that is purchased from the registrar so, for an Azimuth deployment with an ingress base
    domain of `*.apps.example.org`, the Let's Encrypt rate limit is imposed on `example.org`.

    If there are a large number of Zenith services and the rate limit is reached, cert-manager
    will not be able to obtain a certificate and Azimuth and Zenith services will never become
    available.
