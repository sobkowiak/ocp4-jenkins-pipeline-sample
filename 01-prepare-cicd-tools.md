# Prepare CI/CD environment

## Check External Route for Registry

By default the external route for image registry is not created in an OpenShift 4 cluster. We need it in some of our exercises so we have created it beforehand in our lab environment. Check if you can access it by inspecting one of the existing images.

```
# Set your REGISTRY variable to the proper route export 
REGISTRY=default-route-openshift-image-registry.apps.$(oc whoami --show-server | cut -d. -f2- | cut -d: -f1) 
echo ${REGISTRY}

# Log into the registry using Podman, use your username and the token as the password
podman login -u $(oc whoami) -p $(oc whoami -t) ${REGISTRY}
```

After that try to run the `skopeo` command to confirm you can access the registry.

```
skopeo inspect docker://${REGISTRY}/openshift/python
```

## Pre-configuration

Check the `GUID` variable

```
echo $GUID
```

If empty, define it using unique value (e.g. using the GUID received from OPENTLC)

```
export GUID=57bc
```

You can also add this to the `.bashrc` file

```
echo "export GUID=57bc" >> ~/.bashrc
```

## Set up Gogs 

[Gogs](https://gogs.io/) is a self-hosted Git service. Use this step if you don't want to use any already existing git hosting (e.g. Github).

### Set Up Project and Database and Install Gogs

Set up the project, database, and Gogs pod:

```
oc new-project $GUID-gogs --display-name "Shared Gogs"
oc new-app postgresql-persistent --param POSTGRESQL_DATABASE=gogs --param POSTGRESQL_USER=gogs --param POSTGRESQL_PASSWORD=gogs --param VOLUME_CAPACITY=4Gi -lapp=postgresql_gogs
```
Wait until the database pod is running. Set up Gogs pod:

```
oc new-app wkulhanek/gogs:11.86 -lapp=gogs --as-deployment-config
oc rollout pause dc gogs
```

Change the deployment strategy from Rolling to Recreate 

```
oc patch dc gogs --patch='{ "spec": { "strategy": { "type": "Recreate" }}}'
```

Create a persistent volume claim of 4G and connect it to /data

```
oc set volume dc/gogs --add --overwrite --name=gogs-volume-1 --type persistentVolumeClaim --claim-size=4G --claim-name=gogs-data
```

Expose the service as a route:

```
oc expose svc gogs
```

Finally, resume deployment of the Gogs deployment to roll out all changes at once.

```
oc rollout resume dc gogs
```

Display the route

```
oc get route gogs
```
     
Use the value for `$GOGSROUTE`. Navigate to `$GOGSROUTE`. Set up Gogs with these values:
- Database Type: PostgreSQL
- Host: postgresql:5432
- User: gogs
- Password: gogs
- Database Name: gogs
- Run User: gogs
- Application URL: `http://$GOGSROUTE`

![Gogs setup](images/gogs-setup.jpg)

### Make Gogs Pod Resilient to Restarts

The Gogs installation wizard writes the configuration into a `/opt/gogs/custom/conf/app.ini` file in the container. When the pod is restarted, the pod is recreated from the container image and any files written to the container from previous invocations are no longer present. Therefore you must make this file permanent. The easiest way to do this is to extract the file from the container and create a ConfigMap holding that file. This ConfigMap can then be mounted in the correct location, which makes this configuration permanent and resilient to pod restarts.

Examine the generated `app.ini` file:


```
oc exec $(oc get pod | grep "^gogs" | grep Running | awk '{print $1}') -- cat /opt/gogs/custom/conf/app.ini | more
```

Copy the `app.ini` file to your local home directory:


```
oc cp $(oc get pod | grep "^gogs" | grep Running | awk '{print $1}'):opt/gogs/custom/conf/app.ini $HOME/app.ini
```

Create the ConfigMap with the `app.ini` file and mount it as a volume into the pod:


```
oc create configmap gogs --from-file=$HOME/app.ini
oc set volume dc/gogs --add --overwrite --name=config-volume -m /opt/gogs/custom/conf/ -t configmap --configmap-name=gogs
rm -f app.ini
```

Wait until the redeployment finishes.

### Register a new user

* Open your Gogs Server in a web browser.
* Register a new user account for yourself.
* Log in to your Gogs server with your new user account.


## Set up Nexus

### Deploy Nexus

Create a new project

```
oc new-project ${GUID}-nexus --display-name "Shared Nexus"
```

Deploy the Nexus container image and create a route to the Nexus service.

```
oc new-app sonatype/nexus3:3.25.1 --name=nexus --as-deployment-config
oc expose svc nexus
oc rollout pause dc nexus
```

Change the deployment strategy and set requests and limits for memory.

```
oc patch dc nexus --patch='{ "spec": { "strategy": { "type": "Recreate" }}}'
oc set resources dc nexus --limits=memory=2Gi,cpu=2 --requests=memory=1Gi,cpu=500m
```

Create a persistent volume claim (PVC) and mount it at /nexus-data.

```
oc set volume dc/nexus --add --overwrite --name=nexus-volume-1 --mount-path=/nexus-data/ --type persistentVolumeClaim --claim-name=nexus-pvc --claim-size=10Gi
```

Set up liveness and readiness probes for Nexus.

```
oc set probe dc/nexus --liveness --failure-threshold 3 --initial-delay-seconds 60 -- echo ok
oc set probe dc/nexus --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8081/
```

Finally, resume deployment of the Nexus deployment configuration to roll out all changes at once.

```
oc rollout resume dc nexus
```

### Coonfigure Nexus

Once Nexus is deployed, set up your Nexus repository using the provided script. Use the Nexus default user ID (**admin**). The default password for **admin** user is stored in the Nexus pod in the `/nexus-data/admin.password` file. Use the oc rsh command to get into the Nexus pod and retrieve the password.

```
export NEXUS_PASSWORD=$(oc rsh $(oc get pod | grep "^nexus" | grep Running | awk '{print $1}') cat /nexus-data/admin.password)
echo $NEXUS_PASSWORD
```


Due to changes in version `3.21.2` which disabled Groovy scripting by default, you must enable the ability to use a script to allow creation of proxies, repositories, etc. in Nexus. You can enable this by manually adding `nexus.scripts.allowCreation=true` to the `/nexus-data/etc/nexus.properties` file in the Nexus Pod. However, doing this manually should not be a method you follow. With a **DeploymentConfig**, you can use a **deployment-hook** to do this for you.

```
oc set deployment-hook dc/nexus --mid --volumes=nexus-volume-1 \
-- /bin/sh -c "echo nexus.scripts.allowCreation=true >./nexus-data/etc/nexus.properties"

oc rollout latest dc/nexus

watch oc get po
```

With the script option enabled and your Nexus Pod **Ready** and **Running**, you can now proceed with executing the setup script.

```
cd $HOME

curl -o setup_nexus3.sh -s https://raw.githubusercontent.com/redhat-gpte-devopsautomation/ocp_advanced_development_resources/master/nexus/setup_nexus3.sh

chmod +x setup_nexus3.sh

./setup_nexus3.sh admin $NEXUS_PASSWORD http://$(oc get route nexus --template='{{ .spec.host }}')

rm setup_nexus3.sh
```

This script creates:
- A few Maven proxy repositories to cache Red Hat and JBoss dependencies.
- A **maven-all-public** group repository that contains the proxy repositories for all of the required artifacts.
- An NPM proxy repository to cache Node.JS build artifacts.
- A private Docker registry.
- A **releases** repository for the WAR files that are produced by your pipeline.
- A Docker registry in Nexus listening on port 5000. OpenShift does not know about this additional endpoint, so you need to create an additional route that exposes the Nexus Docker registry for your use.

  ```
  oc expose dc nexus --port=5000 --name=nexus-registry
  oc create route edge nexus-registry --service=nexus-registry --port=5000
  ```


### Set up Nexus

Confirm your routes

```
oc get routes
```

- Using the **nexus** route, **admin** as the user id and the Nexus Password you retrieved earlier (using `$NEXUS_PASSWORD` variable) log into Nexus in a web browser. 
- You will see the "Welcome" wizard that prompts you to select a new password for admin.
- Use `app_deploy` as the new password.
- When prompted to Configure Anonymous Access select the Checkbox to allow anonymous access and click Next.
- Click Finish to exit the wizard.
- You may double check that all the repositories (docker, maven-all-public, redhat-ga) are listed under repositories

## Set up SonarQube

Create a new project 

```
oc new-project ${GUID}-sonarqube --display-name "Shared Sonarqube"
```

Deploy a persistent PostgreSQL database.

```
oc new-app --template=postgresql-persistent --param POSTGRESQL_USER=sonar --param POSTGRESQL_PASSWORD=sonar --param POSTGRESQL_DATABASE=sonar --param VOLUME_CAPACITY=4Gi --labels=app=sonarqube_db
```

Make sure that your database is fully up before moving to the next step.

Deploy the SonarQube

```
oc new-app --docker-image=quay.io/gpte-devops-automation/sonarqube:7.9.1 --env=SONARQUBE_JDBC_USERNAME=sonar --env=SONARQUBE_JDBC_PASSWORD=sonar --env=SONARQUBE_JDBC_URL=jdbc:postgresql://postgresql/sonar --labels=app=sonarqube --as-deployment-config
oc rollout pause dc sonarqube
oc expose service sonarqube
```

Create a PVC (at least 5Gi in size) and mount it at `/opt/sonarqube/data`.

```
oc set volume dc/sonarqube --add --overwrite --name=sonarqube-volume-1 --mount-path=/opt/sonarqube/data/ --type persistentVolumeClaim --claim-name=sonarqube-pvc --claim-size=5Gi
```

Set the deployment strategy and resources. SonarQube is a heavy application. The following parameters are suggested:
* Memory request: 2Gi
* Memory limit: 3Gi
* CPU request: 1 CPU
* CPU limit: 2 CPUs

```
oc set resources dc/sonarqube --limits=memory=3Gi,cpu=2 --requests=memory=2Gi,cpu=1
oc patch dc sonarqube --patch='{ "spec": { "strategy": { "type": "Recreate" }}}'
```

Add liveness and readiness probes

```
oc set probe dc/sonarqube --liveness --failure-threshold 3 --initial-delay-seconds 40 --get-url=http://:9000/about
oc set probe dc/sonarqube --readiness --failure-threshold 3 --initial-delay-seconds 20 --get-url=http://:9000/about
```

Because SonarQube uses Elasticsearch and Elasticsearch has some rather heavy requirements add the label `tuned.openshift.io/elasticsearch: "true"` to the SonarQube Pods. This ensures that the OpenShift 4 Tuned operator configures the node that the Sonarqube pod lands on correctly. 

```
oc patch dc/sonarqube --type=merge -p '{"spec": {"template": {"metadata": {"labels": {"tuned.openshift.io/elasticsearch": "true"}}}}}'
```

 Resume deployment 
 
 ```
 oc rollout resume dc sonarqube
 ```
 
Once SonarQube has fully started, open your web browser and log in via the exposed route. The default user ID is **admin** and password is **admin**.
