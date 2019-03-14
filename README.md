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

## Auditing Vault

There is a high chance, that you will need information about who and when accessed you instances. Vault has a solution for that. In the next section we will learn how to enable Vault's auditing device. By default, this feature is disabled.

```shell
   $ vault audit enable syslog
   Success! Enabled the syslog audit device at: syslog/
```

From now on you will have logs from any actions made by Vault.


## One-Time Password for SSH

We have arrived to configuring the One-Time Password solution for using SSH for logging in to our instances.
SSH Sercrets engine needs to be enabled.

```shell
   $ vault secrets enable ssh
   Successfully mounted 'ssh' at 'ssh'!
```

To create One-Time Password role the key_type parameter must be otp and the default_user must be also defined. It could be a specific user or user groups as well. In this example we will create an "avatao" user with otp. All of the machines represented by the role's CIDR list should have [Vault-SSH-Helper](https://github.com/hashicorp/vault-ssh-helper) properly installed and configured.

```shell
   $ vault write ssh/roles/otp_key_role \
      key_type=otp \
      default_user=avatao \
      cidr_list=x.x.x.x/y,m.m.m.m/n
   Success! Data written to: ssh/roles/otp_key_role
```

The next step is to create credential for loging in to our remote instance. For that, Vault uses write command.

```shell
   $ vault write ssh/creds/otp_key_role ip=x.x.x.x
   Key             Value
   lease_id        ssh/creds/otp_key_role/73bbf513-9606-4bec-816c-5a2f009765a5
   lease_duration  600
   lease_renewable false
   port            22
   username        avatao
   ip              x.x.x.x
   key             2f7e25a2-24c9-4b7b-0d35-27d5e5203a5c
   key_type        otp
```

The key will be our password, so you should pass that when you are establishing the SSH connection.

```shell
   $ ssh avatao@x.x.x.x
   Password: <Enter OTP>
   avatao@x.x.x.x:~$
```

We have successfully logged in to our instance with One-Time Password.
A single CLI command can be used to create a new OTP and invoke SSH with the correct parameters to connect to the host.

```shell
   $ vault ssh -role otp_key_role -mode otp avatao@x.x.x.x
   OTP for the session is `b4d47e1b-4879-5f4e-ce5c-7988d7986f37`
   [Note: Install `sshpass` to automate typing in OTP]
   Password: <Enter OTP>
```


## Signed SSH Certificates

Vault also also has solution for Signing SSH Certificates. There is an option to set a Time To Live (TTL) vslue for these certs. In the following example we will configure a role with these features.
Again, SSH Secrets engine needs to be mounted before use.

```shell
   $ vault secrets enable -path=ssh-client-signer ssh
   Successfully mounted 'ssh' at 'ssh-client-signer'!
```

This enables the SSH secrets engine at the path "ssh-client-signer".
Configure Vault with a CA for signing client keys using the /config/ca endpoint. If you do not have an internal CA, Vault can generate a keypair for you.

```shell
   $ vault write ssh-client-signer/config/ca generate_signing_key=true
   Key             Value
   ---             -----
   public_key      ssh-rsa AAAAB3NzaC1yc2EA...
```
Add the public key to all target host's SSH configuration. This process can be manual or automated using a configuration management tool. The public key is accessible via the API and does not require authentication.

```shell
   $ curl -o /etc/ssh/trusted-user-ca-keys.pem http://127.0.0.1:8200/v1/ssh-client-signer/public_key
   $ vault read -field=public_key ssh-client-signer/config/ca > /etc/ssh/trusted-user-ca-keys.pem
```

Add the path where the public key contents are stored to the SSH configuration file as the TrustedUserCAKeys option.

```shell
   # /etc/ssh/sshd_config
   # ...
   TrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem
```

Restart the SSH service to pick up the changes.

### Client SSH Authentication

Now we should create a role for signing client keys. You can specify here the Time To Live value, in our case it will be 30 minutes. "ttl": "30m0s"

```shell
   $ vault write ssh-client-signer/roles/my-role -<<"EOH"
   {
     "allow_user_certificates": true,
     "allowed_users": "*",
     "default_extensions": [
       {
         "permit-pty": ""
       }
     ],
     "key_type": "ca",
     "default_user": "ubuntu",
     "ttl": "30m0s"
   }
   EOH
```

After we have configured Vault, we need to perform the following steps on client's work station who will login to the instance managed by Vault.

Locate or generate the SSH public key. Usually this is ~/.ssh/id_rsa.pub. If you do not have an SSH keypair, generate one:

```shell
   $ ssh-keygen -t rsa -C "avatao@avatao.com"
```

Ask Vault to sign your public key. This file usually ends in .pub and the contents begin with ssh-rsa ....

```shell
$ vault write ssh-client-signer/sign/my-role \
    public_key=@$HOME/.ssh/id_rsa.pub

Key             Value
---             -----
serial_number   c73f26d2340276aa
signed_key      ssh-rsa-cert-v01@openssh.com AAAAHHNzaC1...
```

The result will include the serial and the signed key. This signed key is another public key.
To customize the signing options, use a JSON payload:

```shell
   $ vault write ssh-client-signer/sign/my-role -<<"EOH"
   {
     "public_key": "ssh-rsa AAA...",
     "valid_principals": "my-user",
     "key_id": "custom-prefix",
     "extension": {
       "permit-pty": ""
     }
   }
   EOH
```

Save the resulting signed, public key to disk. Limit permissions as needed.

```shell
$ vault write -field=signed_key ssh-client-signer/sign/my-role \
    public_key=@$HOME/.ssh/id_rsa.pub > signed-cert.pub
```

SSH into the host machine using the signed key. You must supply both the signed public key from Vault and the corresponding private key as authentication to the SSH call.

```shell
$ ssh -i signed-cert.pub -i ~/.ssh/id_rsa avatao@10.0.23.5
```

Here we are, logged in to our instance with Vault SSH Certificate.
