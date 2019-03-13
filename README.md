# avatao

# Vault SSH Secrets Engine
#
#   The Vault SSH secrets engine is a great way to provide secure authentication and authorization for accessing machines via SSH protocol.
#   In this example we will setup a one-time SSH password and also set a time-to-live value for it.
# 
# As we have already isntalled and started up Vault in the previous module, we will start with initializing and unsealing it. We need to set the follwing environment variables;
#
  $ export VAULT_ADDR="http://127.0.0.1:8200"
  $ export VAULT_TOKEN=`cat ${2} |jq -r .root_token`
