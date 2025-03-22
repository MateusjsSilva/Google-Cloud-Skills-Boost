# Cloud Run Deployment and IAM Configuration

## Challenge Scenario
You have joined a cloud development team responsible for deploying and managing services in Google Cloud. Your task is to deploy multiple services using Cloud Run and configure IAM roles as required. Supervisors expect you to follow best practices and work efficiently without a step-by-step guide.

### Guidelines:
- All resources should be created in the default region unless otherwise specified.
- Use environment variables to manage service names and accounts.
- Optimize resource usage and follow naming conventions.
- Deploy services with appropriate IAM configurations.

## Task 1: Configure Google Cloud Environment
Before deploying any service, configure the Google Cloud project and set the default region.

### Commands:
```sh
export REGION=$(gcloud compute project-info describe \
--format="value(commonInstanceMetadata.items[google-compute-default-region])")

gcloud config set project $(gcloud projects list --format='value(PROJECT_ID)' --filter='qwiklabs-gcp')
gcloud config set run/region $REGION
gcloud config set run/platform managed
```

## Task 2: Clone the Repository
To deploy services, you need to clone the source repository.

### Commands:
```sh
git clone https://github.com/rosera/pet-theory.git && cd pet-theory/lab07
```

## Task 3: Deploy Billing Staging API
Deploy the staging version of the billing API using Cloud Run.

### Commands:
```sh
cd ~/pet-theory/lab07/unit-api-billing
gcloud builds submit --tag gcr.io/$DEVSHELL_PROJECT_ID/billing-staging-api:0.1
gcloud run deploy $PUBLIC_BILLING_SERVICE --image gcr.io/$DEVSHELL_PROJECT_ID/billing-staging-api:0.1 --quiet
```

## Task 4: Deploy Frontend Staging
Deploy the staging version of the frontend.

### Commands:
```sh
cd ~/pet-theory/lab07/staging-frontend-billing
gcloud builds submit --tag gcr.io/$DEVSHELL_PROJECT_ID/frontend-staging:0.1
gcloud run deploy $FRONTEND_STAGING_SERVICE --image gcr.io/$DEVSHELL_PROJECT_ID/frontend-staging:0.1 --quiet
```

## Task 5: Update Billing Staging API
Update the staging API to a new version.

### Commands:
```sh
cd ~/pet-theory/lab07/staging-api-billing
gcloud builds submit --tag gcr.io/$DEVSHELL_PROJECT_ID/billing-staging-api:0.2
gcloud run deploy $PRIVATE_BILLING_SERVICE --image gcr.io/$DEVSHELL_PROJECT_ID/billing-staging-api:0.2 --quiet
```

## Task 6: Create Billing IAM Service Account
Create a service account for the billing service.

### Commands:
```sh
gcloud iam service-accounts create $BILLING_SERVICE_ACCOUNT --display-name "Billing Service Account Cloud Run"
```

## Task 7: Deploy Billing Production API
Deploy the production version of the billing API.

### Commands:
```sh
cd ~/pet-theory/lab07/prod-api-billing
gcloud builds submit --tag gcr.io/$DEVSHELL_PROJECT_ID/billing-prod-api:0.1
gcloud run deploy $BILLING_PROD_SERVICE --image gcr.io/$DEVSHELL_PROJECT_ID/billing-prod-api:0.1 --quiet
```

## Task 8: Create Frontend IAM Service Account
Create a service account for the frontend service.

### Commands:
```sh
gcloud iam service-accounts create $FRONTEND_SERVICE_ACCOUNT --display-name "Billing Service Account Cloud Run Invoker"
```

## Task 9: Deploy Frontend Production
Deploy the production version of the frontend.

### Commands:
```sh
cd ~/pet-theory/lab07/prod-frontend-billing
gcloud builds submit --tag gcr.io/$DEVSHELL_PROJECT_ID/frontend-prod:0.1
gcloud run deploy $FRONTEND_PRODUCTION_SERVICE --image gcr.io/$DEVSHELL_PROJECT_ID/frontend-prod:0.1 --quiet
```

## Task 10: Clean Up Unnecessary Files
Remove unwanted files from the project directory.

### Commands:
```sh
remove_files() {
    for file in *; do
        if [[ "$file" == gsp* || "$file" == arc* || "$file" == shell* ]]; then
            if [[ -f "$file" ]]; then
                rm "$file"
                echo "File removed: $file"
            fi
        fi
    done
}
remove_files
```

---
© Mateus Silva | 2025
