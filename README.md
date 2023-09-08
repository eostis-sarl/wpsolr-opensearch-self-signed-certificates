Configure and install Opensearch  with self-signed certificates
=======

OpenSearch is an open-source search and analytics platform designed for scalability and speed. It is built on a foundation of Elasticsearch and Kibana, two popular tools for search and data visualization. OpenSearch offers powerful search and analytical capabilities, making it suitable for a wide range of use cases, from text-based search engines to log and event data analysis.

We are going to use docker containers to host the different applications.
To install the docker engine, use the following link : https://docs.docker.com/engine/install/.

The opensearch official documentation can be found here : https://opensearch.org/docs/latest.

Opensearch uses a great amount of memory maps, so you have to add this line to <i>/etc/sysctl.conf</i> :

    vm.max_map_count = 262144

This will increase the amount of virtual memory maps of your host machine the docker container can use.

If you do intend on using the tls to secure your node, do the following.

Generating the self-signed certificates (optional)
-----------

We're going to consider that the Certifacte Authority (CA) has a CN called "CA" and that the CN of the node will be the hostname of the opensearch server (in this case "opensearch-server-1" so for each time its written in this doc, replace it with your own).

A CA server provides a user-friendly and efficient solution for generating and securely storing asymmetric key pairs. These key pairs are essential for tasks such as encryption, decryption, digital signing, and validation within a Public Key Infrastructure (PKI). The CA server is not something that needs to be active at all times, when it's not generating certs it can be turned off. So the CA CN can be whatever you want.

The Common Name (CN) of the node will be determined by the method your clients use to reach the server. If the client is within another Docker container on the same machine or network, the CN should match the container's name. However, if the client is on a different machine or network, the CN be the hostname of the machine hosting the container.

Enter the CNs like this : 

    CA_CN="CA"
    NODE_CN="opensearch-server-1"
    ADMIN_CN="admin"

So enter the following Distinguished Name (DN) with the following structure:

    CA_DN="/C=AU/ST=Some-State/O=Internet Widgits Pty Ltd/CN=CA"
    NODE_DN="/C=AU/ST=Some-State/O=Internet Widgits Pty Ltd/CN=opensearch-server-1"
    ADMIN_DN="/C=AU/ST=Some-State/O=Internet Widgits Pty Ltd/CN=admin"

If you only need the CN in the DN, you can just write it like this : 

    CA_DN="/CN=CA".
    NODE_DN="/CN=opensearch-server-1"
    ADMIN_DN="/CN=admin"

To generate the self signed certificates, use the following link : https://github.com/eostis-sarl/wpsolr-generate-self-signed-certificates.

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
Configure the opensearch server by modifying the file on the container at <i>/usr/share/opensearch/config/opensearch.yml</i>: 

    plugins.security.ssl.transport.pemcert_filepath: opensearch-server-1.pem #formerly esnode.pem
    plugins.security.ssl.transport.pemkey_filepath: opensearch-server-1-key.pem #formerly esnode-key.pem
    plugins.security.ssl.transport.pemtrustedcas_filepath: ca.pem #Using the newly generated one
    plugins.security.ssl.transport.enforce_hostname_verification: true #formerly false
    plugins.security.ssl.http.enabled: true
    plugins.security.ssl.http.pemcert_filepath: opensearch-server-1.pem #formerly esnode.pem
    plugins.security.ssl.http.pemkey_filepath: opensearch-server-1-key.pem #formerly esnode-key.pem
    plugins.security.ssl.http.pemtrustedcas_filepath: ca.pem #using the newly generated one    
    plugins.security.allow_unsafe_democertificates: false #formerly true
    plugins.security.allow_default_init_securityindex: true 
        
    plugins.security.authcz.admin_dn:
      - "CN=CA,O=Internet Widgits Pty Ltd,ST=Some-State,C=AU"
      - "CN=admin,O=Internet Widgits Pty Ltd,ST=Some-State,C=AU"
    plugins.security.nodes_dn:
      - "CN=opensearch-server-1,O=Internet Widgits Pty Ltd,ST=Some-State,C=AU" # for each node add another line

Since plugins.security.allow_unsafe_democertificates has been set to true, you need to delete the demo certificates esnode.pem, esnode-key.pem, kirk.pem, kirk-key.pem, root-ca.pem and root-ca.pem in the config directory of opensearch-server-1 docker container to not cause any errors. 

Restart the container.

Apply the opensearch security settings using the following command in the opensearch node docker container : 

    sudo docker exec -it opensearch-server-1 ./plugins/opensearch-security/tools/securityadmin.sh -cd config/opensearch-security -icl   -nhnv   -cacert config/ca.pem   -cert config/admin.pem -key config/admin-key.pem

Restart the container to apply the changes.

Manage Opensearch 
-----------

You need to change the admin password

Hash the password you want to use for admin using the script /usr/share/opensearch/plugins/opensearch-security/tools/hash.sh :

    sudo docker exec -it opensearch-server-1 ./plugins/opensearch-security/tools/hash.sh -p new_password

Copy the hash it returns and paste it in <i>/usr/share/opensearch/config/opensearch-security/internal_users.yml</i> :  

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

#### Get all the user's information.

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
