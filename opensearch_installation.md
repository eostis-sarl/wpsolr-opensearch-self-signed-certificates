Configure and install Opensearch  with self-signed certificates
=======

We are going to use docker containers to host the different applications.
To install the docker engine, use the following link : https://docs.docker.com/engine/install/.

The opensearch official documentation can be found here : https://opensearch.org/docs/latest.

Opensearch uses a great amount of memory maps, so you have to add this line to <i>/etc/sysctl.conf</i> :

    vm.max_map_count = 262144

This will increase the amount of virtual memory maps of your host machine the docker container can use.

If you do intend on using the tls to secure your node, do the following.

Generating the self-signed certificates (optional)
-----------

Generate the certificates using the shell script generate_crt.sh.

    ./generate_crt.sh

or

    mkdir opensearch-certs
    cd opensearch-certs

Generate the Root CA : 

    openssl genrsa -out opensearch-server-1-ca-key.pem 2048
    openssl req -new -x509 -sha256 -key opensearch-server-1-ca-key.pem -subj "/C=AU/ST=Some-State/O=Internet Widgits Pty Ltd/CN=opensearch-server-1" -out opensearch-server-1-ca.pem -days 730

Generate the node certificate.

    openssl genrsa -out opensearch-server-1-key-temp.pem 2048
    openssl pkcs8 -inform PEM -outform PEM -in opensearch-server-1-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out opensearch-server-1-key.pem
    openssl req -new -key opensearch-server-1-key.pem -subj "/C=AU/ST=Some-State/O=Internet Widgits Pty Ltd/CN=opensearch-server-1" -out opensearch-server-1.csr
    openssl x509 -req -in opensearch-server-1.csr -CA opensearch-server-1-ca.pem -CAkey opensearch-server-1-ca-key.pem -CAcreateserial -sha256 -out node1.pem -days 730

Start the Opensearch service
-----------

Then you can start the containers. If you intend on using wordpress use the docker-compose.yml file, if not use the docker-compose-opensearch-only.yml file.

    cd path/to/docker-compose.yml
    sudo docker-compose up -d or sudo docker-compose -f specific-file-name up -d

Make sure the volumes or mounts in your dockerfile match the paths and names of the certificate files.

Configure the Opensearch security plugin
-----------

### Using simple http, no authentification

If you do not intend on using the tls to secure your node, add in <i>/usr/share/opensearch/config/opensearch.yml</i> this single line :

    plugins.security.disabled: true

<b><u>WARNING!</u> According to the official documentation at https://opensearch.org/docs/latest/security/configuration/disable: Disabling or removing the plugin exposes the configuration index for the Security plugin. If the index contains sensitive information, be sure to protect it through some other means. If you no longer need the index, delete it.</b>

### Using https with authentification
Configure the opensearch server by modifying the file on the container at /usr/share/opensearch/config/opensearch.yml: 

    plugins.security.ssl.transport.pemcert_filepath: opensearch-server-1.pem #formerly esnode.pem
    plugins.security.ssl.transport.pemkey_filepath: opensearch-server-1-key.pem #formerly esnode-key.pem
    plugins.security.ssl.transport.pemtrustedcas_filepath: opensearch-server-1-ca.pem #Using the newly generated one
    plugins.security.ssl.transport.enforce_hostname_verification: true #formerly false
    plugins.security.ssl.http.enabled: true
    plugins.security.ssl.http.pemcert_filepath: opensearch-server-1.pem #formerly esnode.pem
    plugins.security.ssl.http.pemkey_filepath: opensearch-server-1-key.pem #formerly esnode-key.pem
    plugins.security.ssl.http.pemtrustedcas_filepath: opensearch-server-1-ca.pem #using the newly generated one    
    plugins.security.allow_unsafe_democertificates: false #formerly true
    plugins.security.allow_default_init_securityindex: true 
        
    plugins.security.authcz.admin_dn:
      - "CN=opensearch-server-1,O=Internet Widgits Pty Ltd,ST=Some-State,C=AU"
    plugins.security.nodes_dn:
      - "CN=opensearch-server-1,O=Internet Widgits Pty Ltd,ST=Some-State,C=AU" # for each node add another line

Since plugins.security.allow_unsafe_democertificates has been set to true need to delete the demo certificates esnode.pem, esnode-key.pem in the config directory of opensearch-server-1 docker container to not cause any errors. 
Restart the container

Apply the opensearch security settings using the following command in the opensearch node docker container : 

    sudo docker exec -it opensearch-server-1 /bin/bash
    ./plugins/opensearch-security/tools/securityadmin.sh -cd config/opensearch-security -icl   -nhnv   -cacert config/root-ca.pem   -cert config/opensearch-server-1.pem -key config/opensearch-server-1-key.pem

Manage Opensearch 
-----------

You need to change the admin password 
To do that you need to access the opensearch docker container CLI using the following command :

    sudo docker exec -it opensearch-server-1 /bin/bash

Once you're in hash the password you want to use for admin using the script /usr/share/opensearch/plugins/opensearch-security/tools/hash.sh :

    ./plugins/opensearch-security/tools/hash.sh -p new_password

Copy the hash it returns and paste it in /usr/share/opensearch/config/opensearch-security/internal_users.yml 

    admin:
      hash: "new_hash"
      reserved: true
      backend_roles:
      - "admin"
 
Either do the same for the other users in the file or delete them. Otherwise they will have default password and your opensearch installation will be compromised.
Restart the container or opensearch service for the password change to apply

Now from your host machine  you can connect to the opensearch sevice using : 

    curl -u admin:new_password --cacert opensearch_certs/opensearch-server-1.pem https://localhost:9200

Or from your wordpress machine :

    curl -u admin:new_password --cacert /path/to/opensearch-server-1.pem https://opensearch-server-1:9200

#### Get all the users information.

    curl -XGET "https://localhost:9200/_plugins/_security/api/internal" --cacert opensearch_certs/opensearch-serverserver-1.pem -u admin:new_password

#### Create a user

    curl -XPUT "https://localhost:9200/_plugins/_security/api/internalusers/new_user" -H 'Content-Type: application/json' -d '
    {                    
      "password": "Password_0123!",
      "backend_roles": ["admin"],
      "attributes": {
        "attribute1": "value1",
        "attribute2": "value2"
      }
    }' --cacert opensearch_certs/opensearch-server-1.pem -u admin:new_password
    
Check a specific user (in this case "new_user") information

    curl -XGET "https://localhost:9200/_plugins/_security/api/internalusers/new_user" --cacert opensearch_certs/opensearch-server-1.pem -u new_user:Password_0123!
    
Delete the user if you want

    curl -XDELETE "https://localhost:9200/_plugins/_security/api/internalusers/new_user" --cacert opensearch_certs/opensearch-server-1.pem -u admin:new_password
