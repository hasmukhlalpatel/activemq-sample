# activemq-sample
[ActiveMQ Artemis](https://artemis.apache.org/components/artemis/documentation/latest/docker.html) and dotnet sample


## Run container manually
```sh
docker run --detach --name mycontainer -p 61616:61616 -p 8161:8161 --rm apache/artemis:latest-alpine
```
### Attached to running a container
You can also use the shell command to interact with the running broker using the default username & password artemis, e.g.:
```sh
docker exec -it mycontainer /var/lib/artemis-instance/bin/artemis shell --user artemis --password artemis
```
You can view the container’s logs using:

```
docker logs -f mycontainer
```

Stop the container using:
```sh
docker stop mycontainer
```


## ActiveMQ Artemis with mTLS
setup for running ActiveMQ Artemis with mTLS using Docker Compose.
The key idea is: both the broker and client must present certificates signed by the same CA. The broker needs a keystore (its own cert) and a truststore (the CA that signed client certs).

<img width="1410" height="704" alt="image" src="https://github.com/user-attachments/assets/95cfe825-0e59-4e31-9fe5-45494bdc2417" />

### Generate certificates
Run this once to produce your CA, broker cert, and client cert. All signed by the same CA root.
```sh
#!/bin/bash
mkdir -p certs && cd certs

# 1. CA key + self-signed cert
openssl req -newkey rsa:4096 -keyout ca.key -x509 -days 3650 -out ca.crt \
  -subj "/CN=MyCA" -nodes

# 2. Broker key + CSR + sign with CA
openssl req -newkey rsa:4096 -keyout broker.key -out broker.csr \
  -subj "/CN=broker" -nodes
openssl x509 -req -in broker.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out broker.crt -days 1825

# 3. Client key + CSR + sign with CA
openssl req -newkey rsa:4096 -keyout client.key -out client.csr \
  -subj "/CN=client" -nodes
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out client.crt -days 1825

# 4. Pack into PKCS#12 keystores
PASS=changeit

openssl pkcs12 -export -in broker.crt -inkey broker.key \
  -out broker.p12 -name broker -passout pass:$PASS

openssl pkcs12 -export -in client.crt -inkey client.key \
  -out client.p12 -name client -passout pass:$PASS

# 5. Truststore (CA cert imported into a PKCS#12)
keytool -importcert -noprompt -alias ca -file ca.crt \
  -storetype PKCS12 -keystore truststore.p12 -storepass $PASS

echo "Done. Files in ./certs/"
```

### Artemis broker config (`broker.xml` snippet)
Place this in ./config/broker.xml. The critical part is the <acceptor> with SSL params:
```xml
<acceptor name="amqp-ssl">
  tcp://0.0.0.0:61616?
  sslEnabled=true;
  keyStorePath=/var/lib/artemis-instance/etc/certs/broker.p12;
  keyStorePassword=changeit;
  trustStorePath=/var/lib/artemis-instance/etc/certs/truststore.p12;
  trustStorePassword=changeit;
  needClientAuth=true;
  protocols=AMQP,CORE
</acceptor>
```
`needClientAuth=true` is what enforces mTLS — the broker rejects connections without a valid client cert.

### docker-compose.yml
```yaml
services:
  artemis:
    image: apache/activemq-artemis:latest-alpine
    container_name: artemis-mtls
    ports:
      - "61616:61616"   # AMQP/CORE (mTLS)
      - "8161:8161"     # Web console (plain HTTP for admin — lock this down in prod)
    environment:
      ARTEMIS_USER: admin
      ARTEMIS_PASSWORD: admin
      # Tell Artemis to use your custom broker.xml
      JAVA_OPTS: >-
        -Djavax.net.ssl.keyStore=/var/lib/artemis-instance/etc/certs/broker.p12
        -Djavax.net.ssl.keyStorePassword=changeit
        -Djavax.net.ssl.trustStore=/var/lib/artemis-instance/etc/certs/truststore.p12
        -Djavax.net.ssl.trustStorePassword=changeit
    volumes:
      - ./certs:/var/lib/artemis-instance/etc/certs:ro
      - ./config/broker.xml:/var/lib/artemis-instance/etc/broker.xml:ro
      - artemis-data:/var/lib/artemis-instance/data
    restart: unless-stopped

volumes:
  artemis-data:
```

### Verify the handshake
```sh
# Start the broker
docker compose up -d

# Test mTLS with openssl s_client
openssl s_client \
  -connect localhost:61616 \
  -cert certs/client.crt \
  -key certs/client.key \
  -CAfile certs/ca.crt \
  -showcerts

# Should show: Verify return code: 0 (ok)
```
### .NET client connection string
```csharp
var factory = new ConnectionFactory("amqps://localhost:61616");
factory.SSL.ClientCertPath = "certs/client.p12";
factory.SSL.ClientCertPassword = "changeit";
factory.SSL.RemoteCertRevalidationCallback = (_, cert, _, _) =>
    cert?.Issuer.Contains("MyCA") == true;
```
