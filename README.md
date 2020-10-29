# ElasticStack installation guide

## Startup
Primarily use 
[digitaloceans](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-elastic-stack-on-ubuntu-20-04)
guide as template. If intending to fix the security part, you dont really need to do the nginx security stuff. Doesn't make sense to have two password prompts, there are better ways to get more security.

Also [elasticsearchs](https://www.elastic.co/guide/en/elastic-stack-get-started/7.9/get-started-elastic-stack.html) guide is good, but I think the digital ocean guide pretty much is a superset of this... Not sure however, if the security guide that follows does not work check this one out. 

**Also important DO NOT start using kibana and dont run filebeat before adding the index files as it will fuck things up otherwise.**

No you dont have to configure logstash more than what is on that page(Yes this note is specifically for you Henrik.). All the configuration is done by filebeats using the commands specified at the end of the tutorial. 

## Getting started with security
If running elasticsearch in single-node mode you can follow ["Tutorial: Getting started with security"](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-getting-started.html) guide to setup the default users for kibana and elasticsearch, these are required when you enable the security feature of elasticsearch, for example alerts. 


After this you need to add a logstash_user with write_access to elasticsearch, and configure logstash to log in as that user. Follow this guideline(You can create the user from kibana which is a bit more neat than through the API IMO, but the privileges needed are noted here):<br>
https://www.elastic.co/guide/en/logstash/7.9/ls-security.html 


The elasticsearch-certutil command generates certificates that have no hostname information in them such as SAN fields. This means that you can use the certificate for every node in your cluster, but you must turn off hostname verification

## Logstash -> Elasticsearch <- Kibana HTTPS
Create certificates using the [guide](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/configuring-tls.html#node-certificates) on elastics site.

Then follow the [guide](https://www.elastic.co/guide/en/kibana/7.9/configuring-tls.html#configuring-tls-kib-es) for the kibana side of things

Finally follow the [guide](https://www.elastic.co/guide/en/logstash/7.9/ls-security.html) on logstash, this should be pretty similar to kibana. Only you specify things in the output.conf file instead.

The routine goes something like
1. Use `bin/elasticsearch-certutil ca` to create CA that can be used to generate node certificates. This is your root trust, a file called "elastic-stack-ca.p12".
2. Use `bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12` to create a certificate for the elasticsearch node. This will create (by default) a file called "elastic-certificates.p12". This is the key and the cert combined. This file should probably be password-protected, how to add this to the elasticsearch keystore is mentioned in the guide.
3. The command `bin/elasticsearch-certutil http` is not really necessary if you use your newly created selfsigned certificate but it is if you are using a real CA. Because you need to send a CSR to the CA to get a valid .crt file and key I think... 
4. Because the certificate is self-signed we have to extract a .PEM file(certificate chain?) to specify that we trust this certificate on the clients(logstash and kibana).
```
openssl pkcs12 -in elastic-certificates.p12 -cacerts -nokeys -out elasticsearch-ca.pem
```
5. Put the certificate chain on the logstash and kibana devices somewhere where they are accessible to the users kibana and logstash. 
6. This was the stuff that I added to my logstash, elasticsearch, and kibana:

30-elasticsearch-output.conf
```
...
        elasticsearch {
            ssl => true
            ssl_certificate_verification => false
            cacert => "/home/henrik/elasticsearch-ca.pem"
            hosts => ["https://localhost:9200"]
...
```
kibana.yml
```
elasticsearch.hosts: ["https://192.168.1.5:9200"]

...
xpack.encryptedSavedObjects.encryptionKey: '<32 char key>' #This is to allow alerts in kibana
elasticsearch.ssl.certificateAuthorities: [ "/etc/kibana/truststore/elasticsearch-ca.pem" ]
elasticsearch.ssl.verificationMode: "certificate"
```

elasticsearch.yml
```
...
xpack.security.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: "/etc/elasticsearch/certs/elastic-certificates.p12"
```

To enable detection you need to follow these [steps](https://www.elastic.co/guide/en/security/7.9/detections-permissions-section.html) which you should have done already if you have followed the guidelines above. 

## Other good links
https://www.elastic.co/guide/en/kibana/7.9/settings.html