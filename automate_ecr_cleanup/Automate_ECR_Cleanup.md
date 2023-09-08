# Automate ECR Cleanup: Keeping Your Container Registry Tidy

## Introduction
In a well-set-up CI/CD pipeline, where a new image is pushed to the container registry after each successful build, the number of images in the registry can grow very quickly. This can lead to cost increases and make it difficult to find the right image when you need it. Therefore, it is important to have a strategy to keep your container registry tidy and avoid unnecessary costs. While most container registries have a retention policy that allows you to delete images after a certain period of time, AWS ECR is no exception with its [lifecycle policy](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html). However, we will dive into a more flexible and customizable solution that will allow you to delete images based on your own criteria using Kubernetes cronjobs. This solution is not limited to AWS ECR and can be extended to other container registries of different cloud providers. In this blog, we will focus on AWS ECR.

## Prerequisites
- A Kubernetes cluster
- A container registry (AWS ECR in our case)
- A Kubernetes cronjob that will run the cleanup script

## Kubernetes Cluster
Clusters are powerful tools that allow you to manage multiple containers and their resources. They consist of two main components: the master and the nodes. The master is the control plane of the cluster and is responsible for managing the nodes and the pods. The nodes are the worker machines that run the containers and are managed by the master. There are many ways to create a Kubernetes cluster. You can create one on your local machine using [Minikube](https://minikube.sigs.k8s.io/docs/start/), or you can use a cloud provider like AWS, Azure, or GCP. Since we are only interested in the cleanup process, we will not go into details about how to create a Kubernetes cluster. However, if you want to learn more about Kubernetes, you can check out the Minikube documentation alongside the Kubernetes documentation. (Hopefully, I will write a blog about Kubernetes soon.)

## Understanding the Cronjob Components
To automate a task in Kubernetes, we use cronjobs. A cronjob is a Kubernetes resource that creates a job on a time-based schedule. It is similar to the cron utility in Unix-like operating systems. A cronjob consists of the following components:
- A schedule: a cron expression that defines when the job should run
- A job template: a template that defines the job that will be created by the cronjob
- Restart policy: defines the restart policy for the pod
- Concurrency policy: defines how the cronjob handles concurrent executions of the same job

## Cleanup Script
The cleanup script is a Bash script that will be executed by the cronjob. It will be responsible for deleting old images from the container registry. The script will be executed in a container that has the AWS CLI installed. It will use the AWS CLI to authenticate with the container registry and delete old images. 

#### CronJob Definition
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: tasks-img-cleanup
spec:
  schedule: "10 6 */3 * *"
```

In this section, we define the CronJob itself. It has a name, "tasks-img-cleanup," and its schedule is set using a cron expression. Here, it's configured to run every three days at 6:10 AM.

#### Job Template
```yaml
  jobTemplate:
    spec:
      template:
        spec:
```

The `jobTemplate` specifies the template for the job that the CronJob will create.

#### Container Definition
```yaml
          containers:
            - name: tasks-img-cleanup
              image: amazon/aws-cli
              envFrom:
                - secretRef:
                    name: aws-credentials
              command: ["bash", "-c"]
              args:
                - |
```

In this part, we define a container named `tasks-img-cleanup` that will run within the job. Here's what each field does:

- `name`: Specifies the name of the container.
- `image`: Specifies the Docker image to be used for this container. In this case, we use `amazon/aws-cli`, which includes the AWS CLI.
- `envFrom`: Specifies environment variables sourced from a Kubernetes secret named `aws-credentials`.
- `command` and `args`: Define the command to be executed within the container. In this case, a Bash shell is used with a multi-line script.

#### Cleanup Script
```yaml
                  # Set variables
                  ECR_REGISTRY=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                  REPO_NAME=tasks-srv
                  IMAGE_RETENTION_COUNT=2
```

This section sets environment variables within the container, including the ECR registry URL, repository name, and image retention count.

#### Authentication
```yaml
                  # Authenticate with ECR using AWS credentials from the secret
                  aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                  aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                  aws configure set default.region $AWS_REGION
```

Here, we authenticate the AWS CLI with the provided AWS credentials from the Kubernetes secret.

#### Listing Images
```yaml
                  # Get list of images in repository
                  IMAGES=$(aws ecr list-images --repository-name $REPO_NAME --region $AWS_REGION --query 'imageIds[].imageDigest' --output text | sort)
```

This line retrieves a list of images in the specified ECR repository and sorts them.

#### Calculating Images to Delete
```yaml
                  # Determine the number of images to delete
                  IMAGES_TO_DELETE=$(expr $(echo "$IMAGES" | wc -w) - $IMAGE_RETENTION_COUNT)
```

It calculates the number of images to delete based on the retention count.

#### Printing Image List
```yaml
                  # Print list of images to delete
                  echo "Images in $REPO_NAME:"
                  echo "$IMAGES"
```

These lines print the list of images in the repository to the container's log.

#### Deleting Old Images
```yaml
                  # Delete old images
                  if [[ $IMAGES_TO_DELETE -gt 0 ]]; then
                    IMAGES_TO_DELETE_LIST=$(echo "$IMAGES" | tail -n $IMAGES_TO_DELETE)
                    echo "Deleting $IMAGES_TO_DELETE image(s):"
                    echo "$IMAGES_TO_DELETE_LIST"
                    for imageDigest in $IMAGES_TO_DELETE_LIST

; do
                      aws ecr batch-delete-image --repository-name $REPO_NAME --region $AWS_REGION --image-ids imageDigest=$imageDigest
                    done
                  else
                    echo "No images to delete."
                  fi
```

This section deletes old images if there are more images than the retention count. It prints the images to be deleted and then iterates through them to delete each one.

#### Restart Policy and Concurrency Policy
```yaml
          restartPolicy: OnFailure
    concurrencyPolicy: Forbid
```

Finally, we define the restart policy for the pod. It specifies that the pod should be restarted if it fails. The concurrency policy is set to `Forbid`, which means concurrent executions of the same job are not allowed.

This CronJob configuration automates ECR cleanup, ensuring that your container registry remains organized and cost-effective. It uses Kubernetes cronjobs to schedule regular cleanup tasks, making it a valuable addition to your CI/CD pipeline.


