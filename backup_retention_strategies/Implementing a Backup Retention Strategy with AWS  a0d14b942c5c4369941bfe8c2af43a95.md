# Implementing a Backup Retention Strategy with AWS Cloud Services

# Introduction

In today's fast-paced and technology-driven world, ensuring the availability and integrity of data is of utmost importance for businesses. Data loss or corruption can cause significant financial and reputational damage, making it essential for companies to implement an effective backup retention strategy.

One of the most critical aspects of data management is ensuring that backups are available when needed and are stored securely for a specified period. With the advent of cloud services like AWS (Amazon Web Services), businesses can leverage scalable and reliable solutions to safeguard their data. AWS offers a range of backup and retention services, including Amazon S3 and Amazon EBS, to help organizations meet their backup and retention needs.

This guide focuses specifically on implementing an effective backup retention strategy using AWS S3 and EBS within a Kubernetes environment. We'll explore how to configure these services within a pod configuration and a cronjob to ensure frequent transfer of data. We'll cover topics such as backup frequency and retention periods. By the end of this guide, you'll have a solid understanding of how to implement an effective backup retention strategy using AWS S3 and EBS within a Kubernetes environment.

# Overview of Backup Retention Strategy

A backup retention strategy involves creating and storing backups of data for a specified period to ensure they are available when needed. It defines the frequency of backups, the retention period, and the storage location. It is essential to regularly test backups to ensure that they can be restored successfully.

![backup_retention_draw_one.drawio.png](Implementing%20a%20Backup%20Retention%20Strategy%20with%20AWS%20%20a0d14b942c5c4369941bfe8c2af43a95/backup_retention_draw_one.drawio.png)

# Components

Let's dive deeper into the 3 components involved in the backup retention strategy schema depicted above.

- **K8s cluster**. Kubernetes, or k8s for short, is an open-source container orchestration platform. It provides a framework for deploying, scaling, and managing containerized applications. In the context of this backup retention strategy, k8s is used to manage the deployment of pods and cronjobs.
    - Pods are, in the context of Kubernetes, objects that manage one or more docker containers
    - Cronjobs are schedulable objects that run an image at a given time and date.
- **EBS volume**. EBS, or Elastic Block Store, is a scalable block storage service provided by AWS. It provides persistent block-level storage volumes that can be attached to EC2 instances. In the context of this backup retention strategy, EBS is used to store backups of data that have been created and maintained by the Kubernetes cluster.
- **S3 bucket**. S3, or Simple Storage Service, is an object storage service provided by AWS. It provides scalable and durable storage for objects such as images, videos, and data backups. We will be using it to store backups of data that have been created by the Kubernetes cluster particularly the cronjobs and stored on EBS volumes.

By leveraging these three components within a Kubernetes environment, organizations can implement an effective backup retention strategy that ensures the availability and integrity of their data.

# Configuring MongoDB Backup and Restore with AWS S3

The provided YAML configuration demonstrates how to configure a MongoDB deployment within a Kubernetes cluster to enable backup and restore functionality using AWS S3. Let's break down the configuration file and understand each component.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-mongo
spec:
  selector:
    matchLabels:
      app: my-mongo
  serviceName: pod-mongo-srv
  replicas: 1
  template:
    metadata:
      labels:
        app: my-mongo
        environment: production
    spec:
      terminationGracePeriodSeconds: 80
      containers:
        - name: my-mongo
          image: mongo
          command:
            - mongod
          args:
            - "--replSet=rs0"
            - "--bind_ip=0.0.0.0"
          ports:
            - containerPort: 27017
              name: pod-db
          volumeMounts:
            - name: pod-volumes
              mountPath: /data/db
            - name: backup
              mountPath: /data/backup
        - name: pod-mongo-sidecar
          image: morphy/k8s-mongo-sidecar
          env:
            - name: KUBERNETES_POD_LABELS
              value: "app=my-mongo,environment=production"
            - name: KUBERNETES_SERVICE_NAME
              value: "pod-mongo-srv"
      volumes:
        - name: pod-volumes
          persistentVolumeClaim:
            claimName: pod-pvc
        - name: backup
          emptyDir: {}
---
apiVersion: batch/v1
kind: Job
metadata:
  name: pod-mongo-restore
spec:
  backoffLimit: 3 # Set the maximum number of retries
  activeDeadlineSeconds: 600 # Set a timeout of 10 minutes
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: pod-mongo-restore
          image: alpine
          imagePullPolicy: IfNotPresent
          env:
            - name: BUCKET_NAME
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: BUCKET_NAME
            - name: AWS_REGION
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: AWS_REGION
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: AWS_ACCESS_KEY_ID
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: AWS_SECRET_ACCESS_KEY
          command:
            - /bin/sh
            - -c
          args:
            - |
              # Install necessary packages
              apk add --no-cache curl mongodb-tools python3 py3-pip

              # Install AWS CLI using pip
              pip3 install --upgrade awscli

              # Add mongo shell to the path
              export PATH=$PATH:/usr/bin

              # Configure user
              aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
              aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
              aws configure set default.region $AWS_REGION

              # Download the backup file only if it doesn't exist
              if [ ! -f /data/backup/pod-mongo.gz ]; then
                echo "No backup file found. Downloading from S3..."
                aws s3 cp s3://$BUCKET_NAME/pod-mongo.gz /data/backup/pod-mongo.gz
              fi

              # Wait for the database to be ready
              echo "Waiting for database to be ready..."
              while ! curl http://pod-mongo-srv:27017/ 2>&1 | grep 'It looks like you are trying to access MongoDB over HTTP on the native driver port.' > /dev/null; do
                sleep 1
              done

              # Restore the database
              echo "Restoring database..."
              mongorestore --gzip --archive=/data/backup/pod-mongo.gz --drop --host pod-mongo-srv

              # Mark as successful
              echo "Database initialization completed successfully at $(date)"

              # Exit successfully
              exit 0
          volumeMounts:
            - name: pod-volumes
              mountPath: /data/db
            - name: backup
              mountPath: /data/backup
      volumes:
        - name: pod-volumes
          persistentVolumeClaim:
            claimName: pod-pvc
        - name: backup
          emptyDir: {}
```

## StatefulSet Configuration for MongoDB Deployment

The StatefulSet configuration defines the deployment of MongoDB within the Kubernetes cluster. It ensures that the MongoDB pods are managed and maintained properly. Here are the important parts of the configuration:

- **metadata**: Specifies the name of the StatefulSet as `my-mongo`
- **selector**: Defines the labels used to select the pods that are part of this StatefulSet. In this case, the pods are selected based on the label `app=my-mongo`
- **replicas**: Specifies the number of replicas (in this case, 1) for the StatefulSet.
- **template**: Describes the pod template used for creating the MongoDB pods.
    - **metadata**: Provides labels for the pod, including `app=my-mongo` and `environment=production.`
    - **spec**: Specifies the specifications for the pod.
        - **containers**: Defines the containers running within the pod. In this case, there are two containers:
            - **my-mongo**: Runs the MongoDB image and sets the necessary configuration options for the MongoDB process. It mounts two volumes: `pod-volumes` and `backup`. The `pod-volumes` volume is used for storing the MongoDB data, while the `backup` volume is used for storing backups.
            - **pod-mongo-sidecar**: Runs a sidecar container that assists with the pod replication.
        - **volumes**: Specifies the volumes used by the pod. In this case, there are two volumes:
            - **pod-volumes**: Represents a persistent volume claim (PVC) named `pod-pvc` that provides persistent storage for MongoDB data.
            - **backup**: Represents an `emptyDir` volume, which is a temporary directory within the pod used for storing backups.

## Job Configuration for MongoDB Restore

The Job configuration demonstrates how to restore a MongoDB backup using an AWS S3 bucket. Here are the important parts of the configuration:

- **metadata**: Specifies the name of the Job as `pod-mongo-restore`
- **spec**: Defines the specifications for the Job.
    - **backoffLimit**: Sets the maximum number of retries for the Job.
    - **activeDeadlineSeconds**: Sets a timeout of 10 minutes for the Job.
    - **template**: Describes the pod template used for running the Job.
        - **spec**: Specifies the specifications for the pod running the Job.
            - **restartPolicy**: Sets the restart policy to `Never` to ensure that the pod does not restart automatically. So, it only runs once after success.
            - **containers**: Defines the container running within the pod.
                - **pod-mongo-restore**: Runs an Alpine Linux image and installs necessary packages like curl, MongoDB-tools, and AWS CLI. It mounts the `pod-volumes` and `backup` volumes used by the MongoDB pod.
                - **env**: Specifies the environment variables required for accessing the AWS S3 bucket, such as the bucket name, AWS region, access key ID, and secret access key.
                - **command** and **args**: Execute a series of commands within the container. The commands involve installing dependencies, configuring the AWS CLI, downloading the backup file from the S3 bucket if it doesn't exist, waiting for the database to be ready, and finally restoring the database using the MongoDB restore command.
            - **volumes**: Specifies the volumes used by the pod. It includes the same `pod-volumes` and `backup` volumes as in the StatefulSet configuration.

In the `command` and `args` section of the cronjob configuration, an Alpine image has been used to permit us to run a set of bash commands that allowed to:

- Setup AWS CLI and install MongoDB tools.

```yaml
# Install dependencies and tools
apk add --no-cache curl mongodb-tools python3 py3-pip

# Install AWS CLI using pip
pip3 install --upgrade awscli

# Add mongo shell to the path
export PATH=$PATH:/usr/bin

# Configure user
aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
aws configure set default.region $AWS_REGION
```

- Download the backup file from the S3 bucket if it does not exist.

```yaml
# Download the backup file only if it doesn't exist
if [ ! -f /data/backup/pod-mongo.gz ]; then
	echo "No backup file found. Downloading from S3..."
  aws s3 cp s3://$BUCKET_NAME/pod-mongo.gz /data/backup/pod-mongo.gz
fi
```

- Restore the backup file to the MongoDB database.

```yaml
# Restore the database
echo "Restoring database..."
mongorestore --gzip --archive=/data/backup/pod-mongo.gz --drop --host pod-mongo-srv

# Mark as successful
echo "Database initialization completed successfully at $(date)"

# Exit successfully
exit 0
```

By combining the StatefulSet and Job configurations, we are making sure that any moment the pod is destroyed or freshly restarted if a backup has been saved in the S3 bucket that it will be loaded/restored to the newly deployed Kubernetes object.

# Automating MongoDB Backup with Cron Jobs

To regularly pull a backup from the S3 bucket or save a new backup, we can use a cron job scheduled at a specific time. In our case, the task will consist of saving a backup file to the S3 bucket. Let's take a look at what our Kubernetes configuration file will look like.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pod-db-cron
spec:
  # Run every midnight at 1:00 AM
  schedule: "0 1 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            # Create a backup of the database then upload it to S3 bucket
            - name: pod-db-cron
              image: alpine
              env:
                - name: BUCKET_NAME
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: BUCKET_NAME
                - name: AWS_REGION
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: AWS_REGION
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: AWS_ACCESS_KEY_ID
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: AWS_SECRET_ACCESS_KEY
              command:
                - /bin/sh
                - -c
              args:
                - |
                  # Install necessary packages
                  apk add --no-cache curl mongodb-tools python3 py3-pip

                  # Install AWS CLI using pip
                  pip3 install --upgrade awscli

                  # Create a backup of the database
                  mongodump --host pod-mongo-srv --archive=/tmp/pod-mongo.gz --gzip

                  # Configure user
                  aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                  aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                  aws configure set default.region $AWS_REGION

                  # Upload the backup to S3 bucket
                  aws s3 cp /tmp/pod-mongo.gz s3://$BUCKET_NAME/pod-mongo.gz --region $AWS_REGION

                  # Mark as successful
                  echo "Pod Backup completed successfully at $(date)"

                  # Exit with success
                  exit 0
              volumeMounts:
                - name: pod-volumes
                  mountPath: /tmp
          restartPolicy: OnFailure
          volumes:
            - name: pod-volumes
              persistentVolumeClaim:
                claimName: pod-pvc
```

The CronJob configuration specifies the schedule for running the backup job. In this case, the schedule is set to run every midnight at 1:00 AM using the cron expression "0 1 * * *". You can adjust the schedule as per your requirements.

The configuration creates a container named `pod-db-cron` using an Alpine Linux image. Here's a breakdown of the important sections:

- **env**: Specifies the environment variables required for accessing the AWS S3 bucket, such as the bucket name, AWS region, access key ID, and secret access key. The values are retrieved from a secret named aws-credentials.
- **command and args**: Execute a series of commands within the container to perform the backup:
    1. Install necessary packages like curl, MongoDB-tools, and AWS CLI.
    2. Create a backup of the MongoDB database using the `mongodump` command, specifying the hostname of the MongoDB service as `pod-mongo-srv`.`
    3. Configure the AWS CLI with the provided credentials.
    4. Upload the backup file to the S3 bucket using the `aws s3 cp` command.
    5. Mark the backup process as successful by echoing a success message.
    6. Exit the container with a success status.
- **volumeMounts**: Mounts the `pod-volumes` persistent volume claim (PVC) to the /tmp directory within the container. This allows the backup file to be stored temporarily before being uploaded to the S3 bucket.

The CronJob configuration ensures that the backup job runs according to the specified schedule. If the job fails (e.g., due to an error during the backup process), it will be restarted according to the restart policy `OnFailure`.

With this CronJob configuration, the backup process can be automated for your MongoDB deployment within the Kubernetes cluster. The backups will be created at the scheduled time and uploaded to the specified AWS S3 bucket, ensuring that your data is regularly backed up and accessible for restoration when needed, as we have seen in the previous section.

*Note: You can add a timestamp or any identifier to save alongside the backup if you don’t want to overwrite the previous one, therefore you might such thing as a Backup retention policy to regularly clean your S3 registry.*

# Conclusion

Implementing an effective backup retention strategy is crucial for maintaining data availability and integrity. With AWS S3 and EBS, you can leverage cloud services to store and manage backups securely. By integrating these services into a Kubernetes environment, you can automate backup and restore processes using StatefulSets, Jobs, and CronJobs.

In this comprehensive guide, we discussed the components of a backup retention strategy, including the Kubernetes cluster, EBS volumes, and S3 buckets. We explored the configuration of MongoDB backup and restore using AWS S3, including StatefulSet and Job configurations. Finally, we demonstrated how to automate the backup process using CronJobs.

By following these guidelines, you can establish a robust backup retention strategy using AWS cloud services within your Kubernetes environment. This ensures that your data is protected, and easily restorable, and minimizes the risk of data loss or corruption.