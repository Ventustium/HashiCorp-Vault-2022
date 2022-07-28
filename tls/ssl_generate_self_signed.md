# Use CFSSL to generate certificates

More about [CFSSL here]("https://github.com/cloudflare/cfssl")

```

cd tls

curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64 -o /usr/local/bin/cfssl && \
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64 -o /usr/local/bin/cfssljson && \
chmod +x /usr/local/bin/cfssl && \
chmod +x /usr/local/bin/cfssljson

#generate ca
cfssl gencert -initca ca-csr.json | cfssljson -bare ca 

#generate certificate in /tmp
 cfssl gencert \
 -ca=ca.pem \
 -ca-key=ca-key.pem \
 -config=ca-config.json \
 -hostname="vault,vault.vault.svc.cluster.local,vault.vault.svc,localhost,127.0.0.1" \
 -profile=default ca-csr.json | cfssljson -bare vault
```

view the files:

```
ls -l
```