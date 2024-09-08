# **Automated Jenkins Backup to AWS S3 Using Jenkins Pipelines**

## **Overview**

This project automates the daily backup of the Jenkins `JENKINS_HOME` directory to an AWS S3 bucket using a Jenkins pipeline. It follows best practices by excluding volatile directories like logs and job workspaces and ensuring secure storage in S3 with encryption and versioning enabled. The process is scheduled to run automatically every day.

## **Features**

- 🛠 **Daily Automated Backups**: Scheduled using Jenkins' built-in cron functionality.
- 💡 **Exclude Volatile Files**: Avoids including files that are frequently changing to prevent conflicts during the backup.
- ☁️ **AWS S3 Integration**: Backups are securely stored in an AWS S3 bucket with versioning enabled.
- 🔒 **Secure Storage**: Backup files are encrypted using S3 server-side encryption (SSE-S3 or SSE-KMS).
- 🧹 **Cleanup**: Removes temporary files after successful backup and upload to S3.

## **Technologies Used**

- Jenkins
- AWS S3
- AWS CLI
- Groovy (Jenkins Pipeline)
- Bash (for tar commands)

## **Pre-requisites**

1. **Jenkins** installed on a server or local machine.
2. **AWS CLI** installed and configured with an IAM user or role that has access to S3.
3. **AWS S3 Bucket** for storing backups (with versioning enabled).
4. **AWS Credentials** added in Jenkins' global credentials:
   - **AWS_ACCESS_KEY_ID**
   - **AWS_SECRET_ACCESS_KEY**

## **Setup Instructions**

### Step 1: Install AWS CLI

Ensure the AWS CLI is installed on the Jenkins server:

```bash
sudo apt-get update
sudo apt-get install awscli -y
```

Configure the AWS CLI with your IAM user credentials:

```bash
aws configure
```

### Step 2: Create the S3 Bucket

Create an S3 bucket in AWS for storing the backups. Make sure to:
- Enable **versioning** for the bucket.
- Optionally, configure a **lifecycle policy** to automatically delete old backups after a certain period.

### Step 3: Add AWS Credentials to Jenkins

1. Go to **Manage Jenkins > Manage Credentials**.
2. Add your **AWS Access Key ID** and **AWS Secret Access Key** as credentials in Jenkins, with IDs `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

### Step 4: Create the Jenkins Pipeline

1. Go to Jenkins Dashboard and click on **New Item**.
2. Select **Pipeline** and name the job `Jenkins-Backup-to-S3`.
3. In the **Pipeline** section, copy and paste the following script:

```groovy
pipeline {
    agent any

    environment {
        // Backup configuration
        S3_BUCKET = 'jenkins-backup-ubuntu-server'
        JENKINS_HOME = '/var/lib/jenkins'
        BACKUP_DIR = '/tmp/jenkins_backup'
        DATE = sh(script: 'date +"%Y%m%d_%H%M%S"', returnStdout: true).trim()
    }

    stages {
        stage('Prepare Backup Directory') {
            steps {
                script {
                    sh "mkdir -p ${BACKUP_DIR}"
                }
            }
        }

        stage('Backup Jenkins Home Directory') {
            steps {
                script {
                    sh """
                    tar --exclude='${JENKINS_HOME}/jobs/*/workspace' \
                        --exclude='${JENKINS_HOME}/logs' \
                        --exclude='${JENKINS_HOME}/cache' \
                        --exclude='${JENKINS_HOME}/war' \
                        -czf ${BACKUP_DIR}/jenkins_backup_${DATE}.tar.gz -C ${JENKINS_HOME} .
                    """
                }
            }
        }

        stage('Upload Backup to S3') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "${AWS_CREDENTIALS_ID}"
                ]]) {
                    script {
                        sh """
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        aws s3 cp ${BACKUP_DIR}/jenkins_backup_${DATE}.tar.gz s3://${S3_BUCKET}/jenkins_backup_${DATE}.tar.gz
                        """
                    }
                }
            }
        }

        stage('Cleanup Backup Files') {
            steps {
                script {
                    sh "rm -rf ${BACKUP_DIR}"
                }
            }
        }
    }

    post {
        success {
            echo 'Backup completed and uploaded to S3 successfully!'
        }
        failure {
            echo 'Backup failed. Please check the logs for details.'
        }
    }
}
```

### Step 5: Automate with a Cron Job

To run this pipeline daily, go to the **Build Triggers** section in the job settings and enable **Build periodically**. Use the following cron expression to schedule it for daily backups at midnight:

```
H 0 * * *
```

## **How to Restore Jenkins**

If you need to restore Jenkins from a backup, follow these steps:

1. **Download the backup from S3**:
   ```bash
   aws s3 cp s3://jenkins-backup-ubuntu-server/jenkins_backup_YYYYMMDD_HHMMSS.tar.gz /tmp/jenkins_backup.tar.gz
   ```

2. **Extract the backup**:
   ```bash
   sudo tar -xzf /tmp/jenkins_backup.tar.gz -C /var/lib/jenkins
   ```

3. **Restart Jenkins**:
   ```bash
   sudo systemctl restart jenkins
   ```
