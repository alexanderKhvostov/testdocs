# Managing Tenants Guide

This guide will show you how to:

* Add a new tenant to a running cluster
* Delete a tenant

## Introduction

Opstrace supports multiple, secured tenants to logically isolate concerns on otherwise shared infrastructure.
If you are not familiar with what a "tenant" is, see our [tenant key concepts](../../references/concepts.md#tenants) documentation.

Tenants are presented in the Cluster Admin section of our UI:

![tenant overview page](../../assets/tenants-guide-overview-1.png)

Let's walk through an example of adding a new tenant named `newtenant` to a running Opstrace instance named `showdown`.

<!-- TODO link to the integrations guide when it exists
If you’re coming from the [quick start](../../quickstart.md), and haven’t yet sent data to one of your tenants, stay tuned for our forthcoming integrations guide to make that process easy.
-->

### 1) Create a new RSA Key Pair

Create a new RSA key pair and store it in a file with this command:

```bash
./opstrace ta-create-keypair ./custom-keypair.pem
```

Note: The `ta-` prefix represents the idea of "tenant API authentication."
All `ta-*` commands offered by the Opstrace CLI are new and should be thought of as experimental (command names and signatures are subject to potentially big changes in the future).

After running this command, you have a local file `./custom-keypair.pem` in your file system, with locked-down file permissions.
It is important to understand that this file contains a secret, the _private_ key.

### 2) Create the Token

The following command creates a new authentication token, signed with the private key of the key pair generated in the first step:

```bash
./opstrace ta-create-token showdown newtenant custom-keypair.pem > token-showdown-newtenant.jwt
```

The token (emitted via `stdout` and captured in the file `token-showdown-newtenant.jwt`) is a standards-compliant JSON Web Token (JWT), implementing Opstrace-specific conventions:

* in the JWT header, it encodes an ID of the public key via which it can be
  cryptographically validated.
* in the JWT payload section, it encodes the name of the associated tenant.

For this token to become useful, an Opstrace instance needs to be configured with the _public key_ corresponding to the private key that was used to sign the token.
Let's do that.

### 3) Add the Public Key to a Running Opstrace Instance

You can think of this step as adding a new trust anchor to the trust store of a running Opstrace instance.
Just like with X.509 certificates, trusting a public key means that authentication proof signed with the correspond _private_ key is accepted (reminder: in [public-key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography), verification of authentication proof only ever needs the non-sensitive public key material and not the private key data—that's the beauty).

So, to make an existing Opstrace instance _trust_ the authentication token generated in the previous step, we have to put the public key into the instance:

```bash
./opstrace ta-pubkeys-add aws showdown custom-keypair.pem
```

## Add a New Tenant with the UI

Visit `https://showdown.opstrace.io/cluster/tenants` and press the `Add Tenant` button.
Type the name of the new tenant (here: `newtenant`).

![add tenant gif](../../assets/tenants-guide-add.gif)

**Warning**: we do not yet do strict tenant name validation.
To make sure things work, please keep the name lower case `[a-z]` for now.

The Opstrace controller running in the Opstrace instance will start a number of new components and initiate a DNS reconfiguration.

Effectively, we're now waiting for the DNS name `cortex.newtenant.showdown.opstrace.io` to become available.

We can probe that from our point of view with `curl`:

```bash
curl https://cortex.newtenant.showdown.opstrace.io/api/v1/labels
```

It should take about 5 minutes for DNS name resolution errors to disappear.
Next up, expect an HTTP response with status code `401`, showing the error message
`Authorization header missing` in the response body.

## Test Tenant API Authentication

Let's add said header and make an example API call against the Cortex API for the new tenant:

```bash
$ curl -vH "Authorization: Bearer $(cat token-showdown-newtenant.jwt)" \
    https://cortex.newtenant.showdown.opstrace.io/api/v1/labels
...
< HTTP/2 200
...
{"status":"success","data":[]}
```