# TLS With the cert-manager Operator and GCP

### Prerequisites

The following example assumes that your domain names being certified are managed by
[GCP Cloud Domains](https://cloud.google.com/dns/docs/tutorials/create-domain-tutorial),
and that all domains are configured with DNS tables via
[GCP Cloud DNS](https://cloud.google.com/dns/).
Additionally, your corresponding GCP Project will require a
[service account](https://cloud.google.com/iam/docs/service-accounts)
created with DNS-Administrator privileges.

It is also useful to understand the basics of the `cert-manager` operator.
You can learn more about `cert-manager` from the project
[documentation](https://cert-manager.io/docs/)

### Deploying the cert-manager operator

The verdaccio-openshift images can be configured to mount TLS certificates from any secret.
However, the OpenShift
[OperatorHub](https://docs.openshift.com/container-platform/4.5/operators/understanding/olm-understanding-operatorhub.html)
provides tools that can make TLS certificate generation and
management easier.
One popular example is the
[cert-manager](https://cert-manager.io/docs/)
operator.
You can subscribe to this operator (as cluster admin) from OpenShift OperatorHub and
deploy a cert-manager instance into your OpenShift project (as a developer).

Once you have deployed cert-manager into your OpenShift project,
you must configure one or more certificate Issuers.
`cert-manager` supports a
[variety](https://cert-manager.io/docs/configuration/#supported-issuer-types)
of Issuer types.
In this example we'll be creating an ACME DNS01 Issuer, which looks like the following example
(you will need to fill in `{{ your_email }}` and `{{ your_gcp_project_id }}`).

```yaml
apiVersion: cert-manager.io/v1alpha3
kind: Issuer
metadata:
  name: gcp-dns01-staging
spec:
  acme:
    email: {{ your_email }}
    privateKeySecretRef:
      name: gcp-dns01-staging-acme-key
    # this issuer uses the letsencrypt staging server for dev purposes
    server: 'https://acme-staging-v02.api.letsencrypt.org/directory'
    solvers:
      - dns01:
          clouddns:
            # Google Cloud Platform project ID
            # example sonic-unicorn-123456
            project: {{ your_gcp_project_id }}
            serviceAccountSecretRef:
              # service-account you create with DNS-ADMIN privs in Google Cloud Platform
              # choose key created in JSON format, download and store in secret:
              # oc create secret generic gcp-dns01-svc-acct-key --from-file key.json
              key: key.json
              name: gcp-dns01-svc-acct-key
```

As the comments in the above example indicate, this Issuer expects that a key to
your GCP service account with DNS-Admin privileges is installed to the secret named
`gcp-dns01-svc-acct-key`.
Once you create your service account, download your key into a file named `key.json`.
I saved my key into my `.ssh` directory:

```sh
$ cd /path/to/my/.ssh
$ oc create secret generic gcp-dns01-svc-acct-key --from-file key.json
```

This example is configured to work with the letsencrypt staging API,
which has no rate limit on TLS cert provisioning.
When you are ready to provision production keys, you can also install the
`gcp-dns01-production-issuer` Issuer from the yaml file in this directory.
Keep in mind that production certs are rate limited by letsencrypt.

### configuring a Certificate

We can now provision our domain names with certificates.
The following will cause `cert-manager` to provision a TLS cert
using the specified Issuer.

```yaml
apiVersion: cert-manager.io/v1alpha3
kind: Certificate
metadata:
  name: verdaccio-certificate
spec:
  # this secret is created by cert-manager, with the provisioned TLS certs
  secretName: verdaccio-cert
  # one or more domain names to certify
  # because this calls out to a DNS01 issuer, these must have
  # DNS tables via Google Cloud Platform. Note, the generic domain
  # (in this case manyangled.dev) must also have DNS table entry in GCP
  dnsNames:
    - verdaccio.manyangled.dev
  issuerRef:
    # change this to the production issuer when you are ready
    name: gcp-dns01-staging
```

As indicated in the comments above, both `verdaccio.manyangled.dev` and `manyangled.dev`
must have DNS tables created in GCP.
An implication is that GCP must also be the provider for `manyangled.dev`
