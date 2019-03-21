FROM vault:latest

ENV VAULT_ADDR="http://127.0.0.1:8200"
ENV VAULT_TOKEN=$HOME/.vault-token

RUN vault audit enable syslog	\
	vault secrets enable ssh	\
	vault write ssh/roles/otp_key_role \
      key_type=otp \
      default_user=avatao \
      cidr_list=x.x.x.x/y,m.m.m.m/n
	
CMD ["vault","ssh ","-role","otp_key_role","-mode","otp","avatao@x.x.x.x"]