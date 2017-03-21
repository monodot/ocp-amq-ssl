# amq-ssl-demo

Demonstrating standing up an AMQ broker with 2-way SSL in OpenShift, and then testing with a client.

## Broker

To build and deploy from local source:

    $ cd ocp-amq-ssl/broker
    $ s2i build . registry.access.redhat.com/jboss-amq-6/amq62-openshift ocp-amq-ssl-broker --copy
    $ docker login -u admin -p admin123 localhost:8123
    $ docker tag ocp-amq-ssl-broker localhost:8123/amq/ocp-amq-ssl-broker:1
    $ docker push localhost:8123/amq/ocp-amq-ssl-broker:1
    $ export NEXUS_IP=`docker inspect --format '{{ .NetworkSettings.IPAddress }}' nexus`
    $ oc secrets new-dockercfg my-secret \
        --docker-server=$NEXUS_IP:8123 --docker-username=admin \
        --docker-password=admin123 --docker-email=a@a.com

Create self-signed keys for 2-way SSL (broker truststore contains client certificate, client truststore contains broker certificate):

    $ keytool -genkey -alias broker -keypass changeit -keyalg RSA -keystore broker.ks -dname "CN=broker,L=Gimmerton" -storepass changeit
    $ keytool -genkey -alias client -keypass changeit -keyalg RSA -keystore client.ks -dname "CN=client,L=Gimmerton" -storepass changeit

    $ keytool -export -alias broker -keystore broker.ks -file broker_cert -storepass changeit
    $ keytool -import -alias broker -keystore client.ts -file broker_cert -storepass changeit -noprompt

    $ keytool -export -alias client -keystore client.ks -file client_cert -storepass changeit
    $ keytool -import -alias client -keystore broker.ts -file client_cert -storepass changeit -noprompt

Add broker keystores into a secret:

    $ oc secrets new amq-app-secret broker.ks broker.ts
    $ oc create sa amq-service-account
    $ oc policy add-role-to-user view -z amq-service-account
    $ oc secrets add sa/amq-service-account secret/amq-app-secret
    $ oc secrets add sa/amq-service-account secret/my-secret --for=pull

Create broker resources from template:

    $ oc process -f amq62-ssl.json \
        AMQ_TRUSTSTORE_PASSWORD=changeit \
        AMQ_KEYSTORE_PASSWORD=changeit -o json \
        | bash -c "jq '(.items[] | select(.kind == \"DeploymentConfig\") | .spec.template.spec.containers[0].image) = \"${NEXUS_IP}:8123/amq/ocp-amq-ssl-broker:1\"'" \
        | bash -c "jq '(.items[] | select(.kind == \"DeploymentConfig\") | .spec.triggers) = [{ \"type\": \"ConfigChange\" }]'" \
        | oc create -f -

NOW: Create a Route in the OpenShift Console, and set TLS termination type to **passthrough**.

