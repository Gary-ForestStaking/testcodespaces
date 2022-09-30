# Instructions for installing Elasticsearch and Kibana. Generating keys and encrypting the logs

<img src="example.png" alt="Example diagram"/>

Update and upgrade

```bash
sudo apt update 
sudo apt upgrade
```

Install Required Dependencies

```bash
sudo apt-get install openjdk-17-jre wget apt-transport-https curl gpgv gpgsm gnupg-l10n gnupg dirmngr unzip -y
```

Install Elastic stack 8.x repository signing key

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
```

Run system update

```bash
sudo apt update
```

Install Elasticsearch MAKE SURE TO TAKE NOTE OF ELASTIC PASSWORD

```bash
sudo apt install elasticsearch
```

Install Kibana

```bash
sudo apt install kibana
```

Add JVM heap option to elasticsearch config. This should be half of the ram size ie 32gb ram on the machine it would be -Xms16g and -Xmx16g

```bash
echo '-Xms16g' | sudo tee -a /etc/elasticsearch/jvm.options
echo '-Xmx16g' | sudo tee -a /etc/elasticsearch/jvm.options
```



Create a folder to keep the https certificates in

```bash
sudo mkdir /etc/kibana/certs/
```

Make changes to the kibana.yml file to set the port and server host and set the elasticsearch host information. Just paste this block in one go

```bash
sudo bash -c 'cat <<"EOF" >> /etc/kibana/kibana.yml
server.port: 5601
server.host: "0.0.0.0"
EOF'
```

Generate Kibana Encryption keys and add them to the end of /etc/kibana/kibana.yml

```bash
sudo bash /usr/share/kibana/bin/kibana-encryption-keys generate
```


Create a new CA: use defaults and no password

```bash
sudo bash /usr/share/elasticsearch/bin/elasticsearch-certutil ca
```

Create new certificate for kibana website

```bash
sudo bash /usr/share/elasticsearch/bin/elasticsearch-certutil cert -pem -ca elastic-stack-ca.p12 -name kibana -dns siem
```

Unzip the certificate

```bash
sudo unzip /usr/share/elasticsearch/certificate-bundle.zip -d /etc/kibana/certs/
```

Add the certificates to Kibana

```bash
echo "server.ssl.certificate: /etc/kibana/certs/kibana/kibana.crt" | sudo tee -a /etc/kibana/kibana.yml
```

```bash
echo "server.ssl.key: /etc/kibana/certs/kibana/kibana.key" | sudo tee -a /etc/kibana/kibana.yml
```

```bash
echo "server.ssl.enabled: true" | sudo tee -a /etc/kibana/kibana.yml
```

Reload system services

```bash
sudo systemctl daemon-reload
```

Add elasicsearch and kibana to the startup automatically

```bash
sudo systemctl enable elasticsearch.service 
sudo systemctl enable kibana.service
```

Start all services elasicsearch and kibana

```bash
sudo systemctl start elasticsearch.service 
sudo systemctl start kibana.service
```

Browse kibana <https://IPADDRESSOFMACHINE:5601> this can take a few minutes so be patient. It will ask you for an enrolement token use the output of the command below

```bash
sudo bash /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

Verification code

```bash
sudo bash /usr/share/kibana/bin/kibana-verification-code
```

User is elastic and password is one you took note of before

Change password in kibana go to top left drop down > Stack Management > Users > elastic

Now lets add an elasticagent

Top left > Integrations > Enpoint Security
Give it an Integration Name and Description > Save and continue

Add Elastic Agent to your hosts > Run Standalone

This is an example use the version which matches kibana instalation currently its 8.2 logs wont work if there is a miss match between the siem and log forwarder.

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.2.3-linux-x86_64.tar.gz
tar xzvf elastic-agent-8.2.3-linux-x86_64.tar.gz
cd elastic-agent-8.2.3-linux-x86_64
sudo ./elastic-agent install
```

Replace /opt/Elastic/Agent/elastic-agent.yml
with the file elastic-agent.yml provided and replace the username and password. If you are pointing to mullvad and using socat change the port to the one in mullvad and the host to the endpoint configured

Restart the service

```bash
sudo systemctl restart elastic-agent.service
```

It might take 5 minutes for data to come through



