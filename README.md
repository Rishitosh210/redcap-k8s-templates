# Redcap on k8s #

This project contains a Dockerfile to containerize Redcap running on a LAMP server. It also contains configuration files to deploy this container on a k8s cluster.

## Secrets ##
**Please do not store any passwords, credentials, or secret files in this repository**. All secret files are stored in the PICI LastPass under the path GCP-Credentials4Development\Redcap. You can find secrets and passwords for existing deployments in this shared folder. When creating a new deployment please create a new shared folder in the same path and add all secrets to that folder.

## Building the Docker Image ##

First build and [push the Docker image to GCR](https://cloud.google.com/container-registry/docs/pushing-and-pulling). Replace <PROJECT_ID> with the ID of the project you wish to deploy to. Run these commands from the base directory.

You will need to download the .zip file for Redcap version you want to install and place it in the code folder. Make sure to check the Dockerfile and replace the name of the Redcap zip with that of the zip you are using.

The Docker image is configured to use the America/Los_Angeles time zone. If you wish to use a different time zone you will need to change the TZ environment variable in the Dockerfile and also the date.timezone variable in the php.ini configuration file.

```shell
$ docker build -t gcr.io/<PROJECT_ID>/redcap:8.6.0 .
$ docker push gcr.io/<PROJECT_ID>/redcap:8.6.0
```

## Setting up the Database ##

[Create a MySQL CloudSQL instance](https://cloud.google.com/sql/docs/mysql/create-instance), login, and create a user and database for Redcap. If in production it is strongly advised that you enabled replication on the CloudSQL instance.

Change the name of the database and the user in the following SQL script to reflect the environment you are deploying to. Connect to your database and execute them. If you are deploying to multiple enviornments with the same CloudSQL instance then you will need to have different database names and usernames for each environment.

```sql
CREATE DATABASE redcap_db;
CREATE USER 'redcap_user'@'%' IDENTIFIED BY 'redcap_pass';
GRANT ALL PRIVILEGES ON redcap_db.* TO 'redcap_user'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

## Initializing k8s ##

### Creating the cluster ###

Create a k8s cluster, get the credentials, and create secrets. **If you are deploying multiple environments to the same cluster you only need to complete this step once.** Each deployment will have its own namespace in the same cluster.

```shell
$ gcloud container clusters create redcap --num-nodes 3 --scopes "https://www.googleapis.com/auth/projecthosting,compute-rw,storage-rw" --zone us-west1-a
$ gcloud container clusters get-credentials redcap
```

### Creating a namespace and adding secrets ###

If you are deploying multiple environments in the same cluster create a namespace and use this namespace for any commands where a namespace is specified. You will also need to edit all of the .yaml files in the redcap folder so that the namespace field under metadata reflects the namespace they are being deployed to.

```shell
$ kubectl create namespace redcap
```

Create the CloudSQL secrets. [Create a service account for the CloudSQL database and download the .json credentials](https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine). Use this .json file for credentials.json. Use the username, password, and database name that you created above for the cloudsql-db-credentials.

```shell
$ kubectl --namespace=redcap create secret generic cloudsql-instance-credentials --from-file=credentials.json=./secrets/downloaded-credentials.json
$ kubectl --namespace=redcap create secret generic cloudsql-db-credentials --from-literal=username=redcap_user --from-literal=password='redcap_pass' --from-literal=database=redcap_db
```

Create the secret containing Redcap's salt string. This just needs to be a random string 16 characters in length.

```shell
$ kubectl --namespace=redcap create secret generic redcap-server-secrets --from-literal=salt='<SALT_HERE>''
```

Create a secret for our mail forwarding configuration. We are using [SSMTP](https://wiki.debian.org/sSMTP) to relay mail from the Kubernetes pods to a mail relay. Create an ssmtp configuration file with the following format. [You can use this guide to find SMTP services that work to send mail from GKE](https://cloud.google.com/compute/docs/tutorials/sending-mail/).

```
root=redcap-email@yourdomain.org
mailhub=smtp-relay.gmail.com:587
AuthUser=redcap-email@yourdomain.org
AuthPass=<PASSWORD-HERE>
UseTLS=YES
UseSTARTTLS=YES
hostname=yourdomain.org
FromLineOverride=YES
```

And then create a secret with the configuration file.

```shell
$ kubectl --namespace=redcap create secret generic ssmtp-credentials --from-file=ssmtp.conf=./secrets/ssmtp-sendgrid.conf
```
## Create a Copy of k8s Configuration Files ##

Now that the cluster is created we will want to make a copy of all of our k8s configuration files so that we can keep track of changes specific to our new environment. Make a copy of the 'templates' folder in k8s and name it after the tier/environment you are deploying to (i.e staging, production)

## Shared Filesystem ##

Currently we are using GCS for a shared filesystem.

### Using GCS ###

[Create two new buckets](https://cloud.google.com/storage/docs/creating-buckets) in the GCS project where you are deploying redcap. One for temp data and one for user uploads.

Replace the names `redcap` and `redcap-temp` with the name of the buckets you just created in the files redcap-deployment.yaml and redcap-cron.yaml inside the Redcap folder in the folder containing your k8s configuration files. You will be replacing the bucket name in the following entry.

```yaml
lifecycle:
  postStart:
    exec:
      command:
        - "sh"
        - "-c"
        - >
          gcsfuse -o nonempty -o allow_other --dir-mode 777 --file-mode 777 redcap /mnt/redcap-bucket;
          gcsfuse -o nonempty -o allow_other --dir-mode 777 --file-mode 777 redcap-temp /var/www/site/temp;
```

### Using NFS ###

**Skip this section unless you wish to use NFS instead of GCS**. If you do, you may wish to look at using [Filestore](https://cloud.google.com/filestore/) instead of running your own NFS in your cluster.

If you decide to switch to NFS you should to remove the securityContext and lifecycle sections in the redcap-deployment.yaml and redcap-cron.yaml configuration files.

The following will guide you through provisioning an NFS server in the cluster backed by persistent storage. It is based on the contents of [this repo](https://github.com/kubernetes/examples/tree/master/staging/volumes/nfs).

Run the following commands. You can increase the size if the persistent disk if you desire. If we decide to use Filestore then skip these steps.

```shell
$ gcloud compute disks create redcap-nfs --zone us-west1-a --size 200GB
$ kubectl --namespace=redcap apply -f ./k8s/templates/nfs/nfs-server-pv.yaml
$ kubectl --namespace=redcap apply -f ./k8s/templates/nfs/nfs-server-pvc.yaml
$ kubectl --namespace=redcap apply -f ./k8s/templates/nfs/nfs-server-rc.yaml
$ kubectl --namespace=redcap apply -f ./k8s/templates/nfs/nfs-server-service.yaml
```

Persistent Volumes don't currently support using service names instead of IP addresses for NFS servers. Get the IP address for the NFS Server Service, and replace the IP address in the file nfs-pv.yaml with it. If we are using Filestore then use the NAS server's IP address instead of the IP for the NFS Server Service.

```shell
$ kubectl --namespace=redcap describe services nfs-server
```

```shell
$ kubectl --namespace=redcap apply -f ./k8s/templates/nfs/nfs-pv.yaml
$ kubectl --namespace=redcap apply -f ./k8s/templates/nfs/nfs-pvc.yaml
```

Add the following entry to the volumeMounts key under containers in redcap-cron.yaml and redcap-deployment.yaml in the redcap folder.

```yaml
 - name: nfs
   mountPath: /mnt/nfs
   readOnly: false
```

Add the following entry to the volume key under spec in redcap-cron.yaml and redcap-deployment.yaml in the redcap folder.

```
- name: nfs
  persistentVolumeClaim:
    claimName: nfs
```

You will also need to replace the entry for command with the following under the containers section in redcap-cron.yaml and redcap-deployment.yaml in the redcap folder. Replace `/usr/sbin/apache2ctl -D FOREGROUND` with `/usr/bin/php /var/www/site/cron.php` for the cron job.

```yaml
command: ["/bin/sh", "-c"]
args: ["chown -R www-data:www-data /mnt/nfs &&\
  /usr/sbin/apache2ctl -D FOREGROUND"]
```
## Deploying Redcap ##

Now that we have the cluster and shared filesystem setup, we can actually deploy the Redcap image we crated.

First, deploy the CloudSQL proxy. You will need to edit cloudsql-deployment.yaml and replace the project id, region, and instance name in the command string. You can find this on the instance details page for the CloudSQl instance under the Instance Connection Name field. It should be of the format project-id:region-name:instance-name.

```shell
$ kubectl --namespace=redcap apply -f ./k8s/templates/cloudsql/cloudsql-deployment.yaml
$ kubectl --namespace=redcap apply -f ./k8s/templates/cloudsql/cloudsql-service.yaml
```

Next we will deploy Redcap. Replace the image tag in the k8s configuration files redcap-deployment.yaml and redcap-cron.yaml with the tag we used (i.e. gcr.io/<PROJECT_ID>/redcap:8.6.0). Then create the k8s deployments and services.

```shell
$ kubectl --namespace=redcap apply -f ./k8s/templates/redcap/redcap-deployment.yaml
$ kubectl --namespace=redcap apply -f ./k8s/templates/redcap/redcap-service.yaml
$ kubectl --namespace=redcap apply -f ./k8s/templates/redcap/redcap-cron.yaml
```

You can view the status of these workloads and services from the GKE page in the Cloud Console of the project you are deploying to.

## Automated TLS Certificates ##

Loosely following [this guide](https://blog.n1analytics.com/free-automated-tls-certificates-on-k8s/) for automated TLS certificates.

[Setup Helm](https://docs.helm.sh/using_helm/). If you use a Mac and have Homebrew installed you can do this by running the following command.

```shell
$ brew install kubernetes-helm
```

Install Tiller on the cluster

```shell
$ kubectl create serviceaccount --namespace kube-system tiller
$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
$ helm init --service-account tiller
```

Replace the field controller.service.loadBalancerIP in nginx-ingress.yaml with an **unused/unbound** [reserved regional external IP from GCP](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address).

```shell
gcloud compute addresses create redcap --region us-west-1a
```

Install the nginx-ingress chart with the custom static IP. If you are installing multiple ingresses in the same culster you must name them differently.

```shell
$ helm install --namespace redcap --name nginx-ingress stable/nginx-ingress --values ./k8s/templates/helm/nginx-ingress.yaml
```

We can use the following command to check when our static IP has been assigned to the load balancer.

```shell
$ kubectl --namespace=redcap get services -o wide nginx-ingress-controller

NAME                       TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                      AGE       SELECTOR
nginx-ingress-controller   LoadBalancer   10.3.244.86   <STATIC-IP>   80:30624/TCP,443:31639/TCP   1h        app=nginx-ingress,component=controller,release=nginx-ingress
```

Once this is done, create the Redcap Ingress to be exposed by the load balancer. You will need to change the namespace, host, and the values in the tls field to match the domain name and namespace. You will not be able to access Redcap until certificates have been created.

```shell
$ kubectl --namespace=redcap apply -f ./k8s/templates/redcap/redcap-ingress.yaml
```

Point DNS to the IP address for the load balancer. Verify that it's working with either of the following:

```shell
$ dig redcap.yourdomain.org
```

```shell
$ host -a redcap.yourdomain.org
```

Install the cert-manager chart onto the cluster. You only need one cert-manager instance per cluster

```shell
$ git clone https://github.com/jetstack/cert-manager
$ cd cert-manager
$ git checkout v0.3.0
$ helm install --name cert-manager --namespace kube-system contrib/charts/cert-manager
```

Set up the Issuer and Certificate. Change the email value in the issuer to an email address where registration notifications should be sent. You will also need to change the values in redcap-staging-cert to match your domain name.

```shell
$ kubectl --namespace=redcap apply -f ./k8s/templates/acme/acme-staging-issuer.yaml
$ kubectl --namespace=redcap apply -f ./k8s/templates/redcap/redcap-staging-cert.yaml
$ kubectl --namespace=redcap get certificates
```

At this point we can check the status of the certificate with this command. It may fail the first time and take a minute or so for the domain to be validated and the certificate to be issued. You will need to change the name of the certificate you are describing to reflect the changes you made to redcap-staging-cert.

```shell
$ kubectl --namespace=redcap describe certificate redcap-yourdomain-org-tls
```

We can watch the cert-manager logs to watch progress or see if anything has gone wrong.

```shell
$ kubectl logs deployment/cert-manager-cert-manager cert-manager --namespace kube-system -f
```

Once we have successfully verified that certificate issuing is working with a stating issuer we will create a production issuer. You will need to change the email value in the issuer to an email address where registration notifications should be sent.

```shell
$ kubectl --namespace=redcap apply -f ./k8s/templates/acme/acme-issuer.yaml
```

Once this is created, we will delete the staging certificate and tls secret, and replace it with a production one. You will need to change the values in the production redcap-cert to match your domain name.

```shell
$ kubectl --namespace=redcap delete certificate redcap-yourdomain-org-tls
$ kubectl --namespace=redcap delete secret redcap-yourdomain-org-tls
$ kubectl --namespace=redcap apply -f ./k8s/templates/redcap/redcap-cert.yaml
```

Check the status of the certificate again with this command. It may fail the first time and take a minute or so for the domain to be validated and the certificate to be issued. You will need to change the name of the certificate you are describing to reflect the changes you made to redcap-cert.

```shell
$ kubectl --namespace=redcap describe certificate redcap-yourdomain-org-tls
```

At this point Redcap should have a valid certificate and be properly serving traffic over SSL!

In the future, you may want to use the cert-manager Ingress Shim. This would allow you to tag your Ingress with an certmanager.k8s.io/issuer annotation instead of explicitly creating a Certificate.

## Installing Redcap ##

At this point we will navigate to the path /install.php on our Redcap installation. From here follow instructions to setup the database for Redcap. Note: You should run the database creation commands through a client, as they may not run to completion through the GCloud UI.

## Post-installation Tasks ##

The primary user of Redcap will need to create administrator accounts.

Once this is done they will need to navigate to Control Center and then to File Upload Settings. From here they should set Storage Location to `Local` and set the path to `/mnt/redcap-bucket`.

They should then navigate to the Control Center again and then to the Configuration Check to make sure that the webserver has been deployed and configured correctly.


## License ##

This repository is made available and distributed under the GPLv3 license.