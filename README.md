# Custom AMQ
There are three ways to add custom broker configuration to the official AMQ 7 image:

- [Set BROKER_XML environment variable with your custom broker.xml](#environment-variable).
- [Mount ConfigMap resources hosting any custom configuration file](#configmap-resource).
- [Use S2I procedure if your customization requirements are more complex](#s2i-procedure).

In your custom `broker.xml`, there are a number of variables that you must use and that will be
replaced at runtime with the actual values: in addition to `AMQ_NAME` and `BROKER_IP`, you have
`AMQ_CLUSTER_USER` and `AMQ_CLUSTER_PASSWORD` in case of clustering. Note that you should never
change any configuration that is managed by the Operator.

## Parameters
```sh
GITHUB_USER="fvaleri"
API_ENDPOINT="https://$(crc ip):6443"
ADMIN_NAME="kubeadmin"
ADMIN_PASS="8rynV-SeYLc-h8Ij7-YPYcz"
USER_NAME="developer"
USER_PASS="developer"
NAMESPACE="broker"
AMQ_NAME="my-broker"
REG_SECRET="registry-secret"
REG_USER="***"
REG_PASS="***"
```

## Environment variable
```sh
oc login -u $USER_NAME -p $USER_PASS $API_ENDPOINT
oc project $NAMESPACE

# set the BROKER_XML env variable
oc set env statefulset $AMQ_NAME-ss BROKER_XML="`cat config/broker.xml`"

oc delete pods -l ActiveMQArtemis=$AMQ_NAME
```

## ConfigMap resource
```sh
oc login -u $USER_NAME -p $USER_PASS $API_ENDPOINT
oc project $NAMESPACE

oc create configmap broker-config --from-file=config/broker.xml

# mount the ConfigMap into the AMQ image
oc set volume statefulset $AMQ_NAME-ss --add --overwrite \
    --name=broker-config \
    --mount-path=/opt/amq/conf/broker.xml \
    --sub-path=broker.xml \
    --source='{"configMap":{"name":"broker-config"}}'

oc delete pods -l ActiveMQArtemis=$AMQ_NAME
```

## S2I Procedure

### Image build
```sh
# fork, clone, add your custom files in config folder and push
git clone git@github.com:$GITHUB_USER/custom-amq.git && cd custom-amq
git commit -am "My custom config" && git push

# login and create a new project
oc login -u $USER_NAME -p $USER_PASS $API_ENDPOINT
oc new-project $NAMESPACE

# authenticate to the registry
oc create secret docker-registry $REG_SECRET \
    --docker-server="registry.redhat.io" \
    --docker-username="$REG_USER" \
    --docker-password="$REG_PASS"
oc secrets link default $REG_SECRET --for=pull
oc secrets link builder $REG_SECRET --for=pull
oc secrets link deployer $REG_SECRET --for=pull

# start the custom image build (should end with: Push successful)
oc import-image amq7/amq-broker:7.7 --confirm --from=registry.redhat.io/amq7/amq-broker:7.7 -n $NAMESPACE
oc new-build registry.redhat.io/amq7/amq-broker:7.7~https://github.com/$GITHUB_USER/custom-amq.git && \
    oc set build-secret --pull bc/custom-amq $REG_SECRET && \
    oc start-build custom-amq

# check the build and get the image repository
oc logs -f bc/custom-amq
oc get is | grep custom-amq | awk '{print $2}'
```

### Operator deploy
```sh
# login and select the project
oc login -u $ADMIN_NAME -p $ADMIN_PASS $API_ENDPOINT
oc project $NAMESPACE

# deploy the operator as cluster-admin
oc apply -f deploy/service_account.yaml
oc apply -f deploy/role.yaml
oc apply -f deploy/role_binding.yaml
oc apply -f $deploy_dir/crds/broker_activemqartemis_crd.yaml
oc apply -f $deploy_dir/crds/broker_activemqartemisaddress_crd.yaml
oc apply -f $deploy_dir/crds/broker_activemqartemisscaledown_crd.yaml
oc secrets link amq-broker-operator $REG_SECRET --for=pull
oc apply -f deploy/operator.yaml

# deploy the broker as user
oc login -u $USER_NAME -p $USER_PASS $API_ENDPOINT
oc apply -f - <<EOF
apiVersion: broker.amq.io/v2alpha2
kind: ActiveMQArtemis
metadata:
  name: my-broker
spec:
  deploymentPlan:
    size: 2
    image: image-registry.openshift-image-registry.svc:5000/$NAMESPACE/custom-amq
    requireLogin: true
    persistenceEnabled: true
    messageMigration: true
    journalType: nio
  console:
    expose: true
  acceptors:
    - name: all
      protocols: all
      port: 61617
EOF
```

### Image rebuild
```sh
# do some configuration changes
git commit -am "Config update" && git push

# login and select the project
oc login -u $USER_NAME -p $USER_PASS $API_ENDPOINT
oc project $NAMESPACE

# start a new build and trigger a rolling update
oc start-build custom-amq --follow
oc patch statefulset my-broker-ss -p \
   "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"last-restart\":\"`date +'%s'`\"}}}}}"
```
