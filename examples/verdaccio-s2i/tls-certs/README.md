# TLS With the cert-manager Operator and Edge Routes

### Prerequisites

The following example assumes that your domain names being certified are managed by
[GCP Cloud Domains](https://cloud.google.com/dns/docs/tutorials/create-domain-tutorial),
and that all domains are configured with DNS tables via
[GCP Cloud DNS](https://cloud.google.com/dns/).
Additionally, your corresponding GCP Project will require a
[service account](https://cloud.google.com/iam/docs/service-accounts)
created with DNS-Administrator privileges.

If you prefer to use a non-GCP DNS provider,
several alternative DNS providers are supported by `cert-manager`: refer to
[this list](https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers)
to learn more.
Some additional
[third-party](https://cert-manager.io/docs/configuration/acme/dns01/#webhook)
providers are also configurable.

It is also useful to understand the basics of the `cert-manager` operator and the `cert-utils` operator.
You can learn more about `cert-manager` from the project
[documentation](https://cert-manager.io/docs/).
The documentation for the `cert-utils` operator can be found
[here](https://github.com/redhat-cop/cert-utils-operator#cert-utils-operator).

### Deploying the `cert-manager` and `cert-utils` operators

The verdaccio-openshift edge route can be populated by TLS certificates from any secret.
However, the OpenShift
[OperatorHub](https://docs.openshift.com/container-platform/4.5/operators/understanding/olm-understanding-operatorhub.html)
provides tools that can make TLS certificate generation and
management easier.
Two operators we will use in this example are the
[cert-manager](https://cert-manager.io/docs/)
operator and the
[cert-utils](https://github.com/redhat-cop/cert-utils-operator#cert-utils-operator)
operator.
You can subscribe to these operators (as cluster admin) from OpenShift OperatorHub and
deploy operator instances into your OpenShift project (as a developer).
Note that at this time, the `cert-utils` operator installs directly into a particular
project namespace, so be sure to specify the correct OpenShift project when you install it
(there is no separate deploy-instance step).

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

Once this `Certificate` resource is created in your namespace,
the `cert-manager` operator will detect it and engage `letsencrypt` to request a TLS certificate,
using the DNS table entries you created with your provider (GCP or otherwise)
to respond to the request challenge.
If it is successful, it will store this new TLS cert in a Secret named `verdaccio-cert`.

### Populating an edge Route with the `cert-utils` operator

The verdaccio server can consume a TLS cert directly,
however in OpenShift the idiomatic way to expose services is via Service and Route resources.
By default, a Route is created with an "anonymous" address, which you may very well not
have ownership of.
In this situation you will not be in a position to create a TLS cert for that address.
One way to work around this is to assign your own address.
The following is an example Route resource that exposes verdaccio via the url
`verdaccio.manyangled.dev`, which I happen to own and host via GCP
(you can substitute domain names you own).
Note that this Route resource is already a component of the file
`.../verdaccio-openshift/examples/verdaccio-s2i/verdaccio-s2i-example.yaml`

```yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: verdaccio-s2i-example
  annotations:
    cert-utils-operator.redhat-cop.io/certs-from-secret: verdaccio-cert
spec:
  host: verdaccio.manyangled.dev
  to:
    kind: Service
    name: verdaccio-s2i-example
    weight: 100
  port:
    targetPort: 4873
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None
```

The Route example above demonstrates the following:

- A custom host: `host: verdaccio.manyangled.dev`
- Edge termination: `termination: edge` which receives incoming `https` backed by our TLS cert and forwards it to our verdaccio server as unencrypted `http`
- An annotation `cert-utils-operator.redhat-cop.io/certs-from-secret: verdaccio-cert` which tells the `cert-utils` operator to populate this edge route with TLS certs from the Secret `verdaccio-cert`

In order for this edge route configuration to work,
we need to add a DNS CNAME table entry that maps `verdaccio.manyangled.dev` to our Route.
You can find the canonical hostname for your Route in the OpenShift console,
or from the command line like this:

```sh
$ echo $(oc get routes verdaccio-s2i-example -o jsonpath='{ .status.ingress[0].routerCanonicalHostname }')
apps.cluster-swift-2768.swift-2768.example.opentlc.com
```

OpenShift's cluster DNS maps this hostname as wildcard, and so it expects a hostname of the form
`<some-subhost>.apps.cluster-swift-2768.swift-2768.example.opentlc.com`.
You can choose any sub-host to use in your entry.
An example CNAME mapping might use `verdaccio` as the subhostname, like so:

`verdaccio.manyangled.dev -> verdaccio.apps.cluster-swift-2768.swift-2768.example.opentlc.com`

