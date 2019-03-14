# Vault SSH Secrets Engine

   The Vault SSH secrets engine is a great way to provide secure authentication and authorization for accessing machines via SSH protocol.
   In this example we will setup a one-time SSH password and also set a time-to-live value for it.
 
 ----
 
In the previous module we have installed Vault.Now we have to start it up with the following command:

```shell
  $ vault server -dev
```

This will give you back information about the server. You will need the Api Address, the Unseal Key and the Root Token later for configuration.

```shell
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: (not set)
                   Mlock: supported: false, enabled: false
                 Storage: inmem
                 Version: Vault v1.0.2
             Version Sha: 37a1dc9c477c1c68c022d2084550f25bf20cac33

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

You can check the Vault UI by hitting http://127.0.0.1:8200 to your browser. It will request you to pass your Root Token. With giving that, you can login to the Vault UI.

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: 1+yv+v5mz+aSCK67X6slL3ECxb4UDL8ujWZU/ONBpn0=
Root Token: s.XmpNPoi9sRhYtdKHaQhkHP6x

Development mode should NOT be used in production installations!

==> Vault server started! Log data will stream in below:

```

As Vault has advised in the server startup output, we should set the following environment variables:

```shell
  $ export VAULT_ADDR="http://127.0.0.1:8200"
```

You will now need to assign the Root Token value to VAULT_TOKEN:

```shell
  $ export VAULT_TOKEN=s.XmpNPoi9sRhYtdKHaQhkHP6x
```

Now we need to unseal Vault using the unseal key:

```shell
  $ vault operator unseal
   Unseal Key (will be hidden):
   Key             Value
   ---             -----
   Seal Type       shamir
   Sealed          false
   Total Shares    1
   Threshold       1
   Version         1.0.1
   Cluster Name    vault-cluster-36379e64
   Cluster ID      5b9ab5c2-fd50-46d8-c016-3411c04570bc
   HA Enabled      false
```

# Auditing Vault

There is a high chance, that you will need information about who and when accessed you instances. Vault has a solution for that. In the next section we will learn how to enable Vault's auditing device. By default, this feature is disabled.

```shell
   $ vault audit enable syslog
   Success! Enabled the syslog audit device at: syslog/
```

From now on you will have logs from any actions made by Vault.

----

# One-Time Password

