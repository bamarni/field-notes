This is a parameterized way to install Kafka in strict mode, using a [custom zookeeper](kafka-zookeeper.md).

*If not using a custom zookeeper, use [kafka](kafka.md)*

# Install security CLI
```bash
dcos package install dcos-enterprise-cli --cli --yes
```

# Create key and service account
```bash
#!/bin/bash

# Do not specify a leading slash ('/')
export SERVICE_NAME="kafka"
# Alternate placement example:
# export SERVICE_NAME="dev-stage/path/kafka-with-custom-zookeeper"
export PACKAGE_NAME="kafka"
export PACKAGE_VERSION="2.3.0-1.1.0"

# Either directly specify ZK_URI, or specify the service_name for kafka-zookeeper
export ZK_SERVICE_NAME="kafka-zookeeper"
# export ZK_SERVICE_NAME="dev-stage/path/kafka-zookeeper"
export ZK_SERVICE_DNS_NAME=$(echo ${ZK_SERVICE_NAME} | sed "s|/||g")
export ZK_URI="zookeeper-0-server.${ZK_SERVICE_DNS_NAME}.autoip.dcos.thisdcos.directory:1140,zookeeper-1-server.${ZK_SERVICE_DNS_NAME}.autoip.dcos.thisdcos.directory:1140,zookeeper-2-server.${ZK_SERVICE_DNS_NAME}.autoip.dcos.thisdcos.directory:1140"

# principal is SERVICE_NAME with slashes replaced with '__'
export PRINCIPAL=$(echo ${SERVICE_NAME} | sed "s|/|__|g")

# dns is generated from SERVICE_NAME with slashes removed
export SERVICE_DNS_NAME="$(echo ${SERVICE_NAME} | sed 's|/||g')"

export SERVICE_ACCOUNT_SECRET="${SERVICE_NAME}/sa"
export SERVICE_ROLE="${PRINCIPAL}-role"

# Used for filenames
export PACKAGE_OPTIONS_FILE="${PRINCIPAL}-options.json"
export PERMISSION_LIST_FILE="${PRINCIPAL}-permissions.txt"
export ENDPOINT_FILE="${PRINCIPAL}-endpoints.txt"

dcos security org service-accounts keypair ${PRINCIPAL}-private.pem ${PRINCIPAL}-public.pem
dcos security org service-accounts create -p ${PRINCIPAL}-public.pem ${PRINCIPAL}
dcos security secrets create-sa-secret --strict ${PRINCIPAL}-private.pem ${PRINCIPAL} ${SERVICE_ACCOUNT_SECRET}

# These may not all be necessary, but it does work.
# The 'role' permissions grant permission to create a reservation - need create only
# The 'principal' permissions grant permission to delete a reservation - need delete only
tee ${PERMISSION_LIST_FILE} <<-'EOF'
dcos:mesos:master:framework:role:SERVICE_ROLE       create
dcos:mesos:master:reservation:role:SERVICE_ROLE     create
dcos:mesos:master:volume:role:SERVICE_ROLE          create
dcos:mesos:master:task:user:nobody                  create
dcos:mesos:master:reservation:principal:PRINCIPAL   delete
dcos:mesos:master:volume:principal:PRINCIPAL        delete
dcos:secrets:default:/SERVICE_NAME/*                      full
dcos:secrets:list:default:/SERVICE_NAME                   read
dcos:adminrouter:ops:ca:rw                                full                    
dcos:adminrouter:ops:ca:ro                                full
EOF

sed -i "s|SERVICE_NAME|${SERVICE_NAME}|g" ${PERMISSION_LIST_FILE}
sed -i "s|SERVICE_ROLE|${SERVICE_ROLE}|g" ${PERMISSION_LIST_FILE}
sed -i "s|PRINCIPAL|${PRINCIPAL}|g" ${PERMISSION_LIST_FILE}

while read p; do
dcos security org users grant ${PRINCIPAL} $p
done < ${PERMISSION_LIST_FILE}

tee ${PACKAGE_OPTIONS_FILE} <<-'EOF'
{
  "service": {
    "name": "SERVICE_NAME",
    "service_account":"PRINCIPAL",
    "service_account_secret": "SERVICE_ACCOUNT_SECRET",
    "security": {
      "transport_encryption": {
        "enabled": true
      }
    }
  },
  "kafka": {
    "kafka_zookeeper_uri": "ZK_URI"
  }
}
EOF

sed -i "s|SERVICE_ACCOUNT_SECRET|${SERVICE_ACCOUNT_SECRET}|g" ${PACKAGE_OPTIONS_FILE}
sed -i "s|PRINCIPAL|${PRINCIPAL}|g" ${PACKAGE_OPTIONS_FILE}
sed -i "s|SERVICE_NAME|${SERVICE_NAME}|g" ${PACKAGE_OPTIONS_FILE}
sed -i "s|ZK_URI|${ZK_URI}|g" ${PACKAGE_OPTIONS_FILE}

dcos package install ${PACKAGE_NAME} --package-version=${PACKAGE_VERSION} --options=${PACKAGE_OPTIONS_FILE} --yes --app

echo "zookeeper-0-server.${SERVICE_DNS_NAME}.autoip.dcos.thisdcos.directory:1140,zookeeper-1-server.${SERVICE_DNS_NAME}.autoip.dcos.thisdcos.directory:1140,zookeeper-2-server.${SERVICE_DNS_NAME}.autoip.dcos.thisdcos.directory:1140" > ${ENDPOINT_FILE}
```

# Create topics
```bash
dcos kafka --name=${SERVICE_NAME} topic create
```