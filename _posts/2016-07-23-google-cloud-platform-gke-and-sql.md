---
layout: post
comments: true
title: Google Cloud Platform - Connecting GKE and Cloud SQL
preview: In this post I'll describe the process needed to setup a kubernetes pod on GKE (Google container engine) that has the secrets needed for your containers to connect to a Cloud SQL instance.
---

In this post I'll describe the process needed to setup a kubernetes pod on GKE (Google container engine) that has the secrets needed for your containers to connect to a Cloud SQL instance.

## Prerequisites

- Have `gcloud` already setup and configured.
- Have `kubectl` component installed (part of `gcloud`)

## Setting up GKE

Navigate in the cloud console over to the [container engine](https://console.cloud.google.com/kubernetes/list) section.  Here you can select 'Create a container cluster'.  From here the few important bits are:

- Under 'more' select multiple availability zones inside of your zone for resiliency.
- If using multiple availability zones adjust your instance count (this is per zone).
- Under 'Project access' enable Cloud SQL.

Finally if you are satisfied select 'create', this will take a little while to complete.

## Setting up Cloud SQL

First you will want to create a Cloud SQL instance that you will be connecting to.  This part is straight forward, navigate to the [SQL](https://console.cloud.google.com/sql/create-high-performance) section in the cloud console and follow through the creation of a Cloud SQL instance.  Make sure that you create the instance in the same availability zone that your GKE environment will run from.  By default all of your private networks will have access to the instance.

## Configuring GKE

Hopefully by this time your container environment has finished setting up.  

Before continuing you will need to have the container you want to run ready to push to Google's docker repository.  They have great documentation on how to do this [here](https://cloud.google.com/container-registry/docs/pushing).

Next you will want to get `kubectl` ready to use with the following gcloud command:

```bash
gcloud container clusters --zone [projects master zone eg us-central1-f] --project [project-id] get-credentials [container cluster name]
```

Hopefully you get output like

```
Fetching cluster endpoint and auth data.
kubeconfig entry generated for [container cluster name].
```

Now perform a deployment of your container like so:

```bash
run [deployment name] --image=gcr.io/[project id]/[container name]:[version]
```

Hopefully you get output like

```
deployment "[deployment name]" created
```

At this point our container is running on GKE, and we have a separate Cloud SQL instance spun up that we can talk to.

## Creating credentials for Cloud SQL

Before we are able to add secrets to our kubernetes deployment (pod) we will need to create a username, password, and database on our Cloud SQL instance.

For this purpose I find it easiest to go into the Google Cloud Shell and connect from there, navigate back to your [sql instances](https://console.cloud.google.com/sql/instances) in the cloud console.  Under your instance, under "Access Control" -> "Users" select "Change root password".  Set this to something very secure that you'll remember for the time being to configure other users.  

Next you'll need to figure out your IP address to allow access to your SQL instance.  From the cloud shell run:

```bash
curl -4 icanhazip.com
104.123.123.123
```

Add this IP address under the "Access Control" -> "Authorization" section of your Cloud SQL instance.  Now connect to your SQL instance from the cloud shell with:

```bash
mysql -h [sql instance IP] -u root -p'[mysql password]'
```

And create a user and database:

```mysql
mysql> CREATE DATABASE somedatabase;
Query OK, 1 row affected (0.01 sec)
mysql> GRANT ALL PRIVILEGES ON somedatabase.* TO 'someuser'@'%' IDENTIFIED BY 'UnENCRYPT3DPAssWORD';
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

Now close out of your session with ctrl+c, and verify the above worked:

```bash
mysql -h [sql instance IP] -u someuser -p'UnENCRYPT3DPAssWORD' somedatabase
```

Now delete the authorized network you added previously to give access to your cloud shell.

## Adding secrets to GKE pod

Now you will want to get all of the base64 versions of your secrets, from the linux command line it looks like:

```bash
echo "my-secret-password" | base64
bXktc2VjcmV0LXBhc3N3b3JkCg==
```

Use this to get all of your secrets in base64 format, next create a new file named secrets.yml, fill it out like so:

```
secrets.yml
apiVersion: v1
kind: Secret
metadata:
  name: secrets
type: Opaque
data:
  hostname: [base-64 encoded hostname]
  username: [base-64 encoded username]
  password: [base-64 encoded password]
  database: [base-64 encoded database]
```

Upload your secrets with

```bash
kubectl create -f ./secrets.yml
```

Next find your deployment name again with

```bash
kubectl get deployment
```

Modify your deployment

```bash
kubectl edit deployment [deployment-name]
```

Next you'll get a fairly large YML file, ignore everything up until the **second** spec section, it will be indented with 4 spaces. Modify it to look similar to the following:

```
<...>
    spec:
      containers:
      - image: gcr.io/[project-id]/[container]:[version]
        imagePullPolicy: Always
        name: [container]
        resources: {}
        terminationMessagePath: /dev/termination-log
        volumeMounts:
        - mountPath: /etc/secrets
          name: secrets
          readOnly: true
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: secrets
        secret:
          secretName: secrets
<...>
```

The important parts out of that are:

```
        volumeMounts:
        - mountPath: /etc/secrets
          name: secrets
          readOnly: true
```

```
      volumes:
      - name: secrets
        secret:
          secretName: secrets
```

After saving and exiting assuming it's successful your secrets are then available at `/etc/secrets/<secret-name>`.  For example, the database hostname secret is at `/etc/secrets/hostname`

```bash
cat /etc/secrets/hostname
104.123.123.123 
```

## Finishing up

Now you'll be able to read in your secrets and use them in your applications! There are more ways that you can use secrets [here](http://kubernetes.io/docs/user-guide/secrets/).  I hope this helps you get started with GKE and Cloud SQL.
