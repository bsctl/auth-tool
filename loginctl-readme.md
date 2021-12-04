# OIDC Login CLI
This tool provides `loginctl` command line written in pure bash. It can be used to interact with an OIDC Identity Server, including local setups, on-premises, and cloud providers. It can be used as `kubectl` plugin or as standalone utility.

## Features

- [x] Authenticate using OIDC
    - [ ] Authorization Code Grant
    - [x] Authorization Code Grant with PKCE
    - [ ] Authorization with Resource Owner Password
- [x] Create `kubeconfig`
- [ ] Logout
- [ ] Token introspection
- [x] Show User Info
- [x] Access remote resouces
- [x] Configure login parameters
- [x] Store login parameters


## Prerequisites
We assume you have the OIDC Identity Server installed either local or remote setup. The server is reachable, for example at https://keycloak.127.0.0.1.nip.io/auth/realms/caas.

You have a realm called `caas` and an OIDC public client called `cli` supporting the Authorization Code Grant with PKCE code flow.

A public OIDC client is strongly recommended when the OIDC server is exposed to multiple users belonging to different organizations and it is unsafe to share the OIDC secret between all of them. Use the Authorization Code Grant with PKCE (Proof Key for Code Exchange) as countermeasure against the Authorization Code Interception Attack. This could happen when Client Applications do not have a secure backend and all OIDC dance happens in the user’s device, e.g with Single Page Applications or CLIs. In such cases, a potential malicious application can steal the Authorization Code.

The PKCE-enhanced Authorization Code Grant prevents the issue. It introduces a secret created by the Client Application that can be verified by the Authorization Server. This secret is called the Code Verifier, which is a cryptographically strong random string. Its length should be no less than 43 characters and no more than 128 characters, and Base64URL encoded. Additionally, the Client Application creates a transform value of the Code Verifier called the Code Challenge that is merely the SHA-256 hashed value of the Code Verifier which should also be Base64URL encoded. This way, a malicious attacker can only intercept the Authorization Code, and cannot exchange it for an Access Token without the Code Verifier.

Also, we assume you have a user already provisioned in the `caas` realm and you are able to login with known credentials, i.e. username and passowrd.

## Installation
Install dependencies if you don't have already in your system:

- [curl](https://github.com/curl/curl)
- [jq](https://stedolan.github.io/jq/)
- openssl
- base64

Copy the `loginctl` script somewhere on your `PATH`, and set it executable:

```bash
$ chmod u+x loginctl
```

## Usage
Once you have installed `loginctl` you can see a list of the available commands by running:

```
$ loginctl -h
Usage: loginctl [OPTIONS]

    OPTIONS:
    --help          display usage
    --login         login to the OIDC Server
    --info          display user info
    --token         return id_token 
    --access        access a resource with id_token
    --kubeconfig    create kubeconfig
 
```

### Login to the Identity Server
To get the JWT token from the Identity Server, login

```
$ loginctl --login
[Tue Jan 12 18:26:21 CET 2021][INFO] Checking if prerequisites are installed
[Tue Jan 12 18:26:21 CET 2021][INFO] Starting OIDC login with Proof Key for Code Exchange
OIDC Server URL: https://keycloak.127.0.0.1.nip.io/auth/realms/caas
OIDC Client ID: cli
[Tue Jan 12 18:26:50 CET 2021][INFO] Getting OIDC configuration from https://keycloak.127.0.0.1.nip.io/auth/realms/caas
[Tue Jan 12 18:26:50 CET 2021][INFO] Generating PKCE Code Verifier and Challenge
[Tue Jan 12 18:26:50 CET 2021][INFO] Creating authorization URI

Go to the following link in your browser:

https://keycloak.127.0.0.1.nip.io:30443/auth/realms/caas/protocol/openid-connect/auth?response_type=code&client_id=cli&redirect_uri=urn:ietf:wg:oauth:2.0:oob&scope=openid+offline_access+profile&state=c3q5UsbdM4om5FLavXr3ncXzt9VMW8aFTBLHzMXELo&prompt=consent&code_challenge=LH0mImmfTMG16SrXqe4Mnws3U14Z6PcqV1pvBz82YmI&code_challenge_method=S256&access_type=offline

Enter verification code: **************

[Tue Jan 12 18:27:07 CET 2021][INFO] Requesting token from https://keycloak.127.0.0.1.nip.io/auth/realms/caas
[Tue Jan 12 18:27:07 CET 2021][INFO] Saving token to cache
[Tue Jan 12 18:27:07 CET 2021][INFO] Saving configuration to ~/.loginctl/oidc.conf
```

The initial login creates and stores configurations in the file `~/.loginctl/oidc.conf`

```bash
OIDC_SERVER=https://keycloak.127.0.0.1.nip.io/auth/realms/caas
OIDC_CLIENT_ID=cli
CODE_CHALLENGE_METHOD=S256
AUTH_ENDPOINT=https://keycloak.127.0.0.1.nip.io/auth/realms/caas/protocol/openid-connect/auth
TOKEN_ENDPOINT=https://keycloak.127.0.0.1.nip.io/auth/realms/caas/protocol/openid-connect/token
INTROSPECTION_ENDPOINT=https://keycloak.127.0.0.1.nip.io/auth/realms/caas/protocol/openid-connect/token/introspect
USERINFO_ENDPOINT=https://keycloak.127.0.0.1.nip.io/auth/realms/caas/protocol/openid-connect/userinfo
END_SESSION_ENDPOINT=https://keycloak.127.0.0.1.nip.io/auth/realms/caas/protocol/openid-connect/logout
```

You can start the login process at any time by simply running the command again:

```
$ loginctl --login
```

According to the OIDC specification, you get multiple tokens:

- `access_token`
- `refresh_token`
- `id_token`

All the tokens are locally cached into `~/.loginctl/oidc.cache` file

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJjanloOUdTSFdZejJQRW9TQVZpbENhczlMZm1TRU1IRG5aZ21MQnQ1MENRIn0...",
  "expires_in": 300,
  "refresh_expires_in": 0,
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICI5MzdlZWVkNy03OTBlLTRjN2ItOGE2Ni0zMGExNTY3ZTQ2NDMifQ...",
  "token_type": "Bearer",
  "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJjanloOUdTSFdZejJQRW9TQVZpbENhczlMZm1TRU1IRG5aZ21MQnQ1MENRIn0...",
  "not-before-policy": 0,
  "session_state": "7f6369b1-443f-40b1-b0f5-4d4b5235a296",
  "scope": "openid email profile offline_access groups"
}
```

### Show the ID_TOKEN
The `id_token` is in **JSON Web Token** (**JWT**) format. You can access the `id_token` by running the command: `loginctl --token`

```json
{
  "apiVersion": "client.authentication.k8s.io/v1beta1",
  "kind": "ExecCredential",
  "status": {
    "token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJjanloOUdTSFdZejJQRW9TQVZpbENhczlMZm1TRU1IRG5aZ21MQnQ1MENRIn0...",
    "expirationTimestamp": ""
  }
}
```

### Show the User Info
You can access the info about the logged user by running the command: `loginctl --info` 

```json
{
  "sub": "05c1ab55-473f-4089-a6ea-d7f295c7944e",
  "email_verified": true,
  "name": "Joe Doe",
  "groups": [],
  "preferred_username": "administrator",
  "given_name": "Joe",
  "family_name": "Doe"
}
```

### Create Kubernetes login
Kubernetes supports OIDC authentication. You can login to the APIs server of your Kubernetes cluster by sending a valid `id_token` to the APIs server.

> You MUST configure the token-based authentication on your Kubernetes in order to login with a OIDC token.

To create a `kubeconfig` file use the command: `loginctl --kubeconfig`.

The kubeconfig file is created in your current folder:

```yaml
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://control-plane:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: oidc
  name: oidc
current-context: oidc
kind: Config
preferences: {}
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - loginctl
      - --token
      command: kubectl
      env: null
```

The `loginctl` command line can be used as `kubectl` plugin by renaming it as `kubectl-loginctl` and placing it in your `PATH` so you can run:

```
$ kubectl loginctl --login
```

That's all, folks!