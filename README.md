# Info

```shell

export ZONE=us-central1-a
export PROJECT_ID=makz-support-eap
export CLUSTER_NAME=pv-cluster-clone-1

# create a private cluster, enable beta SD, authorized networks
# network (subnetwork) has enable google access
# example:github.com/kmassada/gke-pv-cluster

gcloud beta container clusters create $CLUSTER_NAME \
 --project $PROJECT_ID \
 --zone $ZONE \
 --cluster-version "1.11.6-gke.11" \
 --enable-stackdriver-kubernetes \
 --enable-private-nodes \
 --enable-private-endpoint \
 --enable-ip-alias \
 --network neuron-z \
 --subnetwork subneuron-z \
 --enable-master-authorized-networks \
 --master-authorized-networks 10.128.0.0/16 \
 --addons HorizontalPodAutoscaling,HttpLoadBalancing \
 --enable-autoupgrade \
 --enable-autorepair

gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID

# For expendiency, create an SA that's an admin and save the token locally

kubectl create sa neuron-sa
kubectl create clusterrolebinding neuron-sa-admin-binding \
    --clusterrole cluster-admin \
    --serviceaccount default:neuron-sa
export SECRET_NAME=`kubectl get serviceaccount neuron-sa  -o json | jq -Mr '.secrets[].name'`
export TOKEN=`kubectl get secrets $SECRET_NAME -o json | jq -Mr '.data.token' | base64 -d`

# make that secret
export APPLICATION=neuron-prober
kubectl create secret generic $APPLICATION-token --from-literal=token=$TOKEN

# simple container with curl, needs to be pushed to your own <gcr bucket>
# REF https://github.com/giantswarm/tiny-tools
export GCR_BUCKET=makz-labs
docker pull giantswarm/tiny-tools
docker tag giantswarm/tiny-tools:latest gcr.io/$PROJECT_ID/$GCR_BUCKET/debug-container:latest
docker push gcr.io/$PROJECT_ID/$GCR_BUCKET/debug-container:latest

# Run container on the fly
cat << EOF > override.json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "spec": {
    "containers": [
      {
        "name": "debug-$APPLICATION",
        "image": "gcr.io/$PROJECT_ID/$GCR_BUCKET/debug-container:latest",
        "stdin": true,
        "tty": true,
        "args": [ "sh" ],
        "env": [
          {
            "name": "TOKEN",
            "valueFrom": {
              "secretKeyRef": {
                "key": "token",
                "name": "$APPLICATION-token"
              }
            }
          }
        ],
        "volumeMounts": [
          {
            "mountPath": "/var/run/secret/$APPLICATION",
            "name": "$APPLICATION-token"
          }
        ]
      }
    ],
    "volumes": [
      {
        "name": "$APPLICATION-token",
        "secret": {
          "secretName": "$APPLICATION-token"
        }
      }
    ]
  }
}
EOF

kubectl run debug-$APPLICATION --generator=run-pod/v1 -it --rm --overrides="`cat override.json`" --image=gcr.io/$PROJECT_ID/$GCR_BUCKET/debug-container:latest sh

/ # curl -k https://kubernetes.default.svc/api/v1/nodes?limit=1 
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "nodes is forbidden: User \"system:anonymous\" cannot list nodes at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "nodes"
  },
  "code": 403
}/ # 

/ # curl -k https://kubernetes.default.svc/api/v1/nodes?limit=1 -H "Authorization: Bearer $TOKEN"
{
    ....
}

```