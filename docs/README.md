# Jenkins Install #

## Deployment information ##

    $ export PROJECT_ID='endpoints-jenkins'
    $ export ZONE='us-central1-f'
    # Jenkins does not need much scopes as Slaves do most of the work, maybe consider reducing this list.
    $ export JENKINS_SCOPES='https://www.googleapis.com/auth/appengine.admin,https://www.googleapis.com/auth/appengine.apis,https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/compute,https://www.googleapis.com/auth/devstorage.full_control,https://www.googleapis.com/auth/gerritcodereview,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/projecthosting,https://www.googleapis.com/auth/servicecontrol'
    $ export JENKINS_INSTANCE='jenkins'
    $ export JENKINS_PORT=8080


## Creating a Jenkins VM Instance ##
Let start by creating a Vm instance in GCE

    $ gcloud compute \
    --project "${PROJECT_ID}" \
    instances create "${JENKINS_INSTANCE}" \
    --zone "${ZONE}" \
    --machine-type "custom-4-8192" \
    --network "default" \
    --maintenance-policy "MIGRATE" \
    --scopes default="${JENKINS_SCOPES}" \
    --tags "jenkins" \
    --image-family=debian-8 \
    --image-project=debian-cloud \
    --boot-disk-size "200" \
    --no-boot-disk-auto-delete \
    --boot-disk-type "pd-standard" \
    --boot-disk-device-name "${JENKINS_INSTANCE}"

Save the external ip:

    $ export JENKINS_IP=<>

ssh to the vm and install the right dependencies:

    $ sudo apt-get update
    $ sudo apt-get install git

Currently we need to install docker on the Jenkins Master to do so, please follow [docker
instructions](https://docs.docker.com/engine/installation/linux/debian/#/debian-jessie-80-64-bit)

Next let's install Jenkins. Follow the [Installation and Upgrade instructions](
https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins+on+Ubuntu).

Jenkins starts on port 8080, You should not be able to access Jenkins unless you create a
firewall rule for it. To do so:

    $ gcloud compute --project "${PROJECT_ID}" firewall-rules create "jenkins-to-remove" \
    --allow tcp:8080 \
    --network "default" \
    --target-tags "jenkins"

Direct your browser to http://${JENKINS_IP}:8080 and follow the instructions from the screen.
Opt for the "Select plugins to install option" and follow to the next section.

## Jenkins Plugins ##

The basic list of plugins is good. You need to add the following:
[Google Authenticated Source plugin](https://wiki.jenkins-ci.org/display/JENKINS/Google+Source+Plugin)
[Google Cloud Backup Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Google+Cloud+Backup+Plugin)
[Google Container Registry Auth Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Google+Container+Registry+Auth+Plugin)
[Google OAuth Credentials plugin](https://wiki.jenkins-ci.org/display/JENKINS/Google+OAuth+Plugin)
[Gradle Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Gradle+Plugin)
[Kubernetes plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin)
[Pre SCM BuildStep Plugin](https://wiki.jenkins-ci.org/display/JENKINS/pre-scm-buildstep)
[Reverse Proxy Auth Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Reverse+Proxy+Auth+Plugin)

Only a few plugins require configuration.

### Reverse Proxy Auth Plugin ###

#### Setup AppEngine Reverse Proxy ####

We need to limit authentication to our Jenkins. To do so we'll create a reverse
proxy using AppEngine and nginx. All we need to do checkout some code and deploy
to AppEngine.

    $ cd appengine-proxy
    $ sh deploy.sh -a "${JENKINS_INSTANCE}" -b "${JENKINS_PORT}" -c "${PROJECT_ID}"


#### Configuring Jenkins Authentication ####
Go to https://${PROJECT_ID}.appspot.com/configureSecurity/ and enable
HTTP Header by reverse proxy. Fill in the following fields:

* Header User Name: X-AppEngine-User-Email.
* Header Groups Name: X-Forwarded-Groups.
* Header Groups Delimiter Name: |.

Go to the advanced settings and set:

* User search filter: uid{0}.
* Cache Update Interval: 15.


### Cloud Backup ###
In Pantheon, https://pantheon.corp.google.com/project/${PROJECT_ID}/storage/browser
There should be one created: jenkinsbackups, if not create a new one.

Go to https://${PROJECT_ID}.appspot.com/configure. In Peristent Master
Backup Plugin, set the following:

* Enable backup: Check
* Enable auto-restore: Check
* Storage Mechanism: Google Cloud Storage
* Bucket: jenkinsbackups (or the name of the bucket you created.)


### Kubernetes Plugin Setup ###
#### Creating Jenkins Cluster ####

Jenkins runs all its slaves in a Kubernetes Cluster. The cluster name is
hardcoded in the Jenkinsfile.

    $ export CLUSTER_NAME='jenkins-cluster-tmp'

    $ gcloud container \
    --project "${PROJECT_ID}" \
    clusters create "${CLUSTER_NAME}" \
    --zone "${ZONE}" \
    --machine-type "n1-highmem-32" \
    --scopes\
    "https://www.googleapis.com/auth/appengine.admin",\
    "https://www.googleapis.com/auth/appengine.apis",\
    "https://www.googleapis.com/auth/bigquery",\
    "https://www.googleapis.com/auth/cloud-platform",\
    "https://www.googleapis.com/auth/compute",\
    "https://www.googleapis.com/auth/devstorage.full_control",\
    "https://www.googleapis.com/auth/devstorage.read_only",\
    "https://www.googleapis.com/auth/gerritcodereview",\
    "https://www.googleapis.com/auth/logging.write",\
    "https://www.googleapis.com/auth/projecthosting",\
    "https://www.googleapis.com/auth/service.management",\
    "https://www.googleapis.com/auth/servicecontrol" \
    --num-nodes "1" \
    --network "default" \
    --enable-cloud-logging \
    --no-enable-cloud-monitoring

#### Adding jenkins cluster to kubectl config ####

You should be running this on Jenkins machine after cluster creation.

    # To ssh to jenkins
    $ gcloud container --project "${project_id}" --zone "${zone}" \
    compute ssh jenkins

    # On Jenkins VM, sudo as jenkins
    $ sudo su - jenkins

    # Update kubectl config.
    # You might want to run this command as well on your desktop.
    $ gcloud container --project "${project_id}" --zone "${zone}" \
    clusters get-credentials "${CLUSTER_NAME}"

    # You can run this command from jenkins or from your desktop
    $ kubectl create namespace jenkins-slaves

#### Updating Jenkins to point to the cluster ####

First we need to find out where the kubernetes service is running

    $ kubectl  describe service kubernetes
    Name:                   kubernetes
    Namespace:              default
    Labels:                 component=apiserver,provider=kubernetes
    Selector:               <none>
    Type:                   ClusterIP
    IP:                     10.39.240.1
    Port:                   https   443/TCP
    Endpoints:              104.154.116.242:443
    Session Affinity:       None
    No events.

In this case the Endpoint is 104.154.116.242, so jenkins needs to
connect to https://104.154.116.242.

Point your browser to to go/esp-j/configure and find the 'Kubernetes'
section at the end of the page. The only things that needs to be updated
is 'Kubernetes URL' which should point to the URL above.
This tutorial is here to help install Jenkins to GCE from scratch in case our
current install needs to be redone.


## Maintenance ##

### Keeping Jenkins instance up to date ###

Log in via SSH and run the following commands

    $ sudo aptitude update
    $ sudo aptitude full-upgrade
    $ sudo apt-get update
    $ sudo apt-get dist-upgrade


### Keeping Jenkins Plugins up to date ###

Go to the [Plugin Manager](https://endpoints-jenkins.appspot.com/pluginManager),
select all plugins and check the restart checbox.
