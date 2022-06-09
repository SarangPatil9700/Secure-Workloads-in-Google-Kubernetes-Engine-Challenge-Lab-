# Secure-Workloads-in-Google-Kubernetes-Engine-Challenge-Lab-
Task 0:

gsutil cp gs://spls/gsp335/gsp335.zip .

unzip gsp335.zip

# Create ENV VARs for Dynamic Variables:

DM_DEPLOYMENT=$(gcloud deployment-manager deployments list | grep NAME | cut -d ' ' -f2)

CLUSTER_NAME=$(gcloud deployment-manager deployments describe $DM_DEPLOYMENT | grep security-demo | cut -d ' ' -f2)

CLOUD_SQL_INSTANCE_NAME=$(gcloud deployment-manager deployments describe $DM_DEPLOYMENT | grep wordpress-db | cut -d ' ' -f2)

SA_NAME=$(gcloud deployment-manager deployments describe $DM_DEPLOYMENT | grep sa-wordpress | cut -d ' ' -f2)

Task 1:

gcloud container clusters create $CLUSTER_NAME  --machine-type n1-standard-4 --num-nodes 2 --zone us-central1-c --enable-network-policy
gcloud container clusters get-credentials $CLUSTER_NAME --zone us-central1-c


Task 2:

gcloud sql instances create $CLOUD_SQL_INSTANCE_NAME --region=us-central1
gcloud sql databases create wordpress --instance $CLOUD_SQL_INSTANCE_NAME
gcloud sql users create wordpress --instance=$CLOUD_SQL_INSTANCE_NAME --host=% --password='P@ssword!'


Task 3:

gcloud iam service-accounts create $SA_NAME --display-name $SA_NAME

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT    --role roles/cloudsql.client  --member serviceAccount:$SA_NAME@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com

gcloud iam service-accounts keys create key.json     --iam-account $SA_NAME@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com

kubectl create secret generic cloudsql-instance-credentials     --from-file key.json
kubectl create secret generic cloudsql-db-credentials \
   --from-literal username=wordpress \
   --from-literal password='P@ssword!'
   

Task 4:

kubectl apply -f volume.yaml

sed -i s/INSTANCE_CONNECTION_NAME/${GOOGLE_CLOUD_PROJECT}:us-central1:$CLOUD_SQL_INSTANCE_NAME/g wordpress.yaml

kubectl apply -f wordpress.yaml



helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx

kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.8.0/cert-manager.yaml

kubectl get svc

Check the service nginx-ingress-controller as an external IP address before continuing to the next step.

sed -i s/LAB_EMAIL_ADDRESS/$SA_NAME@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com/g issuer.yaml

kubectl apply -f issuer.yaml

If you face any internal error, execute the command again

. add_ip.sh

HOST_NAME=$(echo $USER | tr -d '_').labdns.xyz
sed -i s/HOST_NAME/${HOST_NAME}/g ingress.yaml

kubectl apply -f ingress.yaml


Task 6: 

kubectl apply -f network-policy.yaml

nano network-policy.yaml

--------- Add the following code at the last line-------

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: allow-nginx-access-to-internet
spec:
 podSelector:
   matchLabels:
     app.kubernetes.io/name: ingress-nginx
 policyTypes:
 - Ingress
 ingress:
 - {}


kubectl apply -f network-policy.yaml


gcloud services enable \
   container.googleapis.com \
   containeranalysis.googleapis.com \
   binaryauthorization.googleapis.com

gcloud container clusters update $CLUSTER_NAME --enable-binauthz --zone us-central1-c


Task 7: 

gcloud container binauthz policy export > bin-auth-policy.yaml

nano bin-auth-policy.yaml


#### Edit and add the four values and change
admissionWhitelistPatterns:
- namePattern: docker.io/library/wordpress:latest
- namePattern: us.gcr.io/k8s-artifacts-prod/ingress-nginx/*
- namePattern: gcr.io/cloudsql-docker/*
- namePattern: quay.io/jetstack/*
defaultAdmissionRule:
 enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
 evaluationMode: ALWAYS_DENY
globalPolicyEvaluationMode: ENABLE
name: projects/<Project_ID>/policy
##########

gcloud container binauthz policy import bin-auth-policy.yaml

Go to Security > Binary Authorization, click EDIT POLICY. Change Default rule to Disallow all images and SAVE POLICY.

