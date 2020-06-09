# Custom AMQ image with S2I
You can build a custom AMQ image on top of the official one using the source-to-image procedure.
This way you can add your own configuration and binary files without the need to create a Dockerfile.

## Common: custom image build
Fork this repository using your GitHub account.
```sh
# set your own credentials here
GITHUB_USER="fvaleri"
API_ENDPOINT="https://$(crc ip):6443"
REG_INTERNAL="image-registry.openshift-image-registry.svc:5000"
ADMIN_NAME="kubeadmin"
ADMIN_PASS="8rynV-SeYLc-h8Ij7-YPYcz"
USER_NAME="developer"
USER_PASS="developer"
PROJECT_NAME="broker"
REG_SECRET="registry-secret"
REG_USER="***"
REG_PASS="***"

# clone, add your custom files in config folder and push
git clone git@github.com:$GITHUB_USER/amq-s2i.git && cd amq-s2i
git commit -am "My custom config" && git push

# login and create a new project
oc login -u $USER_NAME -p $USER_PASS $API_ENDPOINT
oc new-project $PROJECT_NAME

# authenticate to the registry
oc create secret docker-registry $REG_SECRET \
    --docker-server="registry.redhat.io" \
    --docker-username="$REG_USER" \
    --docker-password="$REG_PASS"
oc secrets link default $REG_SECRET --for=pull
oc secrets link builder $REG_SECRET --for=pull
oc secrets link deployer $REG_SECRET --for=pull

# start the custom image build (should end with: Push successful)
oc import-image amq7/amq-broker:7.7 --confirm --from=registry.redhat.io/amq7/amq-broker:7.7 -n $PROJECT_NAME
oc new-build registry.redhat.io/amq7/amq-broker:7.7~https://github.com/$GITHUB_USER/amq-s2i.git && \
    oc set build-secret --pull bc/amq-s2i $REG_SECRET && \
    oc start-build amq-s2i

# check the build and get the image repository
oc logs -f bc/amq-s2i
oc get is | grep amq-s2i | awk '{print $2}'
```

## Option A: broker deploy via Operator
Download `amq-broker-operator-7.7.0-ocp-install-examples.zip` from [RedHat developer portal](https://developers.redhat.com/products/amq/overview) and unzip.
You need a `cluster-admin` user to deploy the Operator and your configuration must not interfere with managed resources.
```sh
# deploy the operator
oc login -u $ADMIN_NAME -p $ADMIN_PASS $API_ENDPOINT
oc apply -f deploy/service_account.yaml
oc apply -f deploy/role.yaml
oc apply -f deploy/role_binding.yaml
oc apply -f $deploy_dir/crds/broker_activemqartemis_crd.yaml
oc apply -f $deploy_dir/crds/broker_activemqartemisaddress_crd.yaml
oc apply -f $deploy_dir/crds/broker_activemqartemisscaledown_crd.yaml
oc secrets link amq-broker-operator $REG_SECRET --for=pull
oc apply -f deploy/operator.yaml

# deploy the broker (set custom image repository here)
oc login -u $USER_NAME -p $USER_PASS $API_ENDPOINT
oc apply -f - <<EOF
apiVersion: broker.amq.io/v2alpha2
kind: ActiveMQArtemis
metadata:
  name: my-broker
spec:
  deploymentPlan:
    size: 1
    image: $REG_INTERNAL/$PROJECT_NAME/amq-s2i
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

## Option B: broker deploy via Template
Image stream and template files are hosted directly on GitHub.
```sh
IMAGE_STREAM="https://raw.githubusercontent.com/jboss-container-images/jboss-amq-7-broker-openshift-image/77-7.7.0.GA/amq-broker-7-image-streams.yaml"
TEMPLATE="https://raw.githubusercontent.com/jboss-container-images/jboss-amq-7-broker-openshift-image/77-7.7.0.GA/templates/amq-broker-77-basic.yaml"

# create the ServiceAccount and download the template
echo '{"kind":"ServiceAccount","apiVersion":"v1","metadata":{"name":"amq-service-account"}}' | oc create -f -
oc policy add-role-to-user view system:serviceaccount:$PROJECT_NAME:amq-service-account
curl -o /tmp/broker.yaml $TEMPLATE

# deploy the broker (set custom image repository here)
oc process -f /tmp/broker.yaml \
    -p APPLICATION_NAME="$PROJECT_NAME" \
    -p AMQ_EXTRA_ARGS="--require-login" \
    -p IMAGE_STREAM_NAMESPACE="$PROJECT_NAME" \
    -p IMAGE="$REG_INTERNAL/$PROJECT_NAME/amq-s2i" \
    | oc apply -f -
```

## Configuration update
Push and apply your configuration changes.
```sh
# do some configuration changes
git commit -am "Config update" && git push

# login and select the project
oc login -u $USER_NAME -p $USER_PASS $API_ENDPOINT
oc project $PROJECT_NAME

# start a new build and trigger a rolling update
oc start-build amq-s2i --follow
oc patch statefulset my-broker-ss -p \
   "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"last-restart\":\"`date +'%s'`\"}}}}}"
```
