# gke-helm-chart
=====working steps uday=====

1.Set up the Identity Pool: 


1.$ gcloud iam workload-identity-pools create "ghpool" \
    --project="buoyant-site-480804-k1" \
    --location="global" \
    --display-name="GitHub Actions Workload Identity Pool"

2. $ gcloud iam workload-identity-pools providers create-oidc "github-provider" \
    --project="buoyant-site-480804-k1" \
    --location="global" \
    --workload-identity-pool="ghpool" \
    --display-name="GitHub OIDC Provider" \
    --issuer-uri="https://token.actions.githubusercontent.com" \
    --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
    --attribute-condition="attribute.repository=='udaykirantesting/gke-helm-chart'"


2.1 this step to get pool id 
 export POOL_ID=$(gcloud iam workload-identity-pools describe "ghpool" \
  --location="global" --format="value(name)" --project="buoyant-site-480804-k1 ")


echo "${POOL_ID}"


3.sample command we need to replace 
Bind GitHub to the service account: gcloud iam service-accounts add-iam-policy-binding "gha-cloudrun-deployer@iapvm123.iam.gserviceaccount.com" \ --role="roles/iam.workloadIdentityUser" \ --member="principalSet://iam.googleapis.com/${POOL_ID}/attribute.repository/lucasFernandesj/cicd" \ --project="iapvm123"


$gcloud iam service-accounts add-iam-policy-binding "github-actions@buoyant-site-480804-k1.iam.gserviceaccount.com" \
--role="roles/iam.workloadIdentityUser" \
--member="principalSet://iam.googleapis.com/projects/553423796220/locations/global/workloadIdentityPools/ghpool/attribute.repository/udaykirantesting/gke-helm-chart" \
--project="buoyant-site-480804-k1"


=======
This is used for sending the service account key alert notification through github actions

->Firstly we need to enable the required API's gcloud services enable iam.googleapis.com
cloudfunctions.googleapis.com
cloudbuild.googleapis.com
cloudresourcemanager.googleapis.com

->We need to create the pub/sub topic for sending notifications gcloud pubsub topics create my-topic

->We need to create one service account or we can use the existing service account. Here I am using the existing service account which is already available in the project. Here I am using "github-actions-test@prefab-mapper-466606-d8.iam.gserviceaccount.com"

->Next we need to create Workload Identity Pool gcloud iam workload-identity-pools create "sa-github-pool"
--location="global"
--display-name="SA key"

->In the Workload Identity Pool we need to create the Provider. In the below one we need to replace the WIF pool name and Github username and repo name. gcloud iam workload-identity-pools providers create-oidc "github-provider"
--location="global"
--workload-identity-pool="sa-github-pool"
--display-name="SA Key Provider"
--issuer-uri="https://token.actions.githubusercontent.com"
--attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"
--attribute-condition="attribute.repository == 'Sai-Kiran1426/service-account-key'"

->Allow GitHub Repo to Assume the Identity. In the below one, replace the project number, WIF pool name, Github username and reponame. gcloud iam service-accounts add-iam-policy-binding
github-actions@guido-460817.iam.gserviceaccount.com
--role="roles/iam.workloadIdentityUser"
--member="principalSet://iam.googleapis.com/projects/495059377885/locations/global/workloadIdentityPools/sa-github-pool/attribute.repository/Sai-Kiran1426/service-account-key"

gcloud projects add-iam-policy-binding prefab-mapper-466606-d8
--member="serviceAccount:github-actions-test@prefab-mapper-466606-d8.iam.gserviceaccount.com"
--role="roles/cloudfunctions.developer"

->Grant Permissions to the Service Account gcloud projects add-iam-policy-binding prefab-mapper-466606-d8
--member="serviceAccount:github-actions-test@prefab-mapper-466606-d8.iam.gserviceaccount.com"
--role="roles/cloudfunctions.admin"

gcloud projects add-iam-policy-binding prefab-mapper-466606-d8
--member="serviceAccount:github-actions-test@prefab-mapper-466606-d8.iam.gserviceaccount.com"
--role="roles/iam.serviceAccountTokenCreator"

gcloud projects add-iam-policy-binding prefab-mapper-466606-d8
--member="serviceAccount:github-actions-test@prefab-mapper-466606-d8.iam.gserviceaccount.com"
--role="roles/iam.serviceAccountTokenCreator"

->Now we need to create app password as we are using gmail here. In the case of organization we already have SMTP credentials there is no need to add app password. Here is the link to create app password for gmail. https://myaccount.google.com/apppasswords?rapt=AEjHL4OhzDKJs0aBn7jpdmBGiXYWuBDLvCFfEbw2ToQ7QOKj4RWiu-nqhgF-ZDPlVxQMcnElaKNUxIbG66-HHEebr_x-FbCi4tGDtTl_ww-NwDvnS5q4Cd0

Once the app password is created we need to note that password and save it.

->Now go to the github repo settings. In settings click on the Secrets and variables which is present in the security tab. In secrets and variables click on actions. In the actions we need to add new repository secret. Add the below details EMAIL_USERNAME: saikiran.m9498@gmail.com EMAIL_PASSWORD: (The one which is generated through app password) EMAIL_SENDER: saikiran.m9498@gmail.com EMAIL_SMTP: smtp.gmail.com EMAIL_RECIPIENTS: saikiran.m9498@gmail.com

->After adding the above details to the repo, now you need to create deploy.yaml file. This file needs to be saved like .github/workflows/deploy.yaml

->Once the deploy.yaml file is added then now go to the actions tab. The actions get triggered automatically and wait till the successful job is done.

->We need to test whether the trigger is running as expected or not. Run the below one gcloud pubsub topics publish my-topic --message="trigger" In the above one we need to replace the topic name.

->If it is suucessful we will be getting the email alert notification to the email we mentioned.

->Now we need to create a Cloud Scheduler Job to run at a particular time daily. gcloud scheduler jobs create pubsub daily-alert-email
--schedule="0 10 * * *"
--time-zone="Asia/Kolkata"
--topic=my-topic
--message-body="trigger"
--location=us-central1

->Grant Scheduler permission to publish to Pub/Sub gcloud scheduler jobs describe daily-alert-email --location=us-central1

->Create the missing Cloud Scheduler service account gcloud iam service-accounts create cloud-scheduler
--display-name="Cloud Scheduler Service Account"

->Give that service account Pub/Sub publishing rights gcloud pubsub topics add-iam-policy-binding my-topic
--member="serviceAccount:cloud-scheduler@prefab-mapper-466606-d8.iam.gserviceaccount.com"
--role="roles/pubsub.publisher"

->Now update the schedule gcloud scheduler jobs update pubsub daily-alert-email
--location=us-central1
--time-zone="Asia/Kolkata"
--schedule="0 10 * * *"
--topic=my-topic
--message-body="trigger"

->Now check if job is set: gcloud scheduler jobs describe daily-alert-email --location=us-central1

Look for: schedule: 0 10 * * * timeZone: Asia/Kolkata

->Now the Cloud scheduler is set at everyday 10 AM. It will be triggered and run everyday at 10 AM and you'll be getting the alert email.
