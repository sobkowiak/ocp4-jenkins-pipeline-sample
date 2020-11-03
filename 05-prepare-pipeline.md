# Prepare sample pipeline

In this part, you explore the practice of continuous integration and continuous deployment (CI/CD). You build a pipeline that delivers the **openshift-tasks** application into a production environment. The build process integrates Gogs, Nexus, SonarQube, and S2I builds. You configure the pipeline to be triggered when a new version of the application is pushed to Gogs. Finally, you integrate the pipeline with the OpenShift web console.

## Deployment to `dev` project

### Set up dev project in OpenShift

Create the **GUID-tasks-dev** OpenShift project (replacing **GUID** with your assigned GUID) to hold the development version of the **openshift-tasks** application:
- Set up the permissions for Jenkins to be able to manipulate objects in the `${GUID}-tasks-dev` project.
- Create a binary build using the `jboss-eap72-openshift:1.0` image stream.
- Create a new deployment configuration pointing to `tasks:0.0-0`.
- Turn off automatic building and deployment.
- Expose the deployment configuration as a service (on port 8080) and the service as a route.
- Create a placeholder ConfigMap (to be updated by the pipeline).
- Attach the ConfigMap to the deployment configuration.

```
# Set up Dev Project
oc new-project ${GUID}-tasks-dev --display-name "Tasks Development"
oc policy add-role-to-user edit system:serviceaccount:${GUID}-jenkins:jenkins -n ${GUID}-tasks-dev

# Set up Dev Application
oc new-build --binary=true --name="tasks" jboss-eap72-openshift:1.0 -n ${GUID}-tasks-dev
oc patch bc/tasks -n ${GUID}-tasks-dev -p '{"spec":{"resources":{"limits":{"cpu": "2", "memory": "2Gi"}, "requests":{"cpu": "1", "memory": "1Gi"}}}}'

oc new-app ${GUID}-tasks-dev/tasks:0.0-0 --name=tasks --allow-missing-imagestream-tags=true -n ${GUID}-tasks-dev
oc set triggers dc/tasks --remove-all -n ${GUID}-tasks-dev
oc expose dc tasks --port 8080 -n ${GUID}-tasks-dev
oc expose svc tasks -n ${GUID}-tasks-dev
oc set probe dc/tasks -n ${GUID}-tasks-dev --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8080/

oc create configmap tasks-config --from-literal="application-users.properties=Placeholder" --from-literal="application-roles.properties=Placeholder" -n ${GUID}-tasks-dev
oc set volume dc/tasks --add --name=jboss-config --mount-path=/opt/eap/standalone/configuration/application-users.properties --sub-path=application-users.properties --configmap-name=tasks-config -n ${GUID}-tasks-dev
oc set volume dc/tasks --add --name=jboss-config1 --mount-path=/opt/eap/standalone/configuration/application-roles.properties --sub-path=application-roles.properties --configmap-name=tasks-config -n ${GUID}-tasks-dev
```

### Create pipeline job

In your `openshift-tasks` directory, create a Jenkinsfile file and copy the content of the [sample](jenkins/jobs/openshift-tasks/Jenkinsfile.dev) pipeline script. Make sure to replace the parameter **prefix** with your GUID. Add the Jenkinsfile file to the repository, then commit and push it into the repository.

In Jenkins, create a New Item:
- Type: Pipeline
- Name: Tasks
- Definition: Pipeline script from SCM
- Add the Git repository.  Your Jenkins job may need credentials to retrieve the `Jenkinsfile` from the repository.
- Start the pipeline

## Deployment to `prod` project (optional)

### Set up prod project in OpenShift (optional)

Create the **GUID-tasks-prod** OpenShift project (replacing **GUID** with your assigned GUID) to hold the production versions of the **openshift-tasks** application:
- Set up the permissions for Jenkins to be able to manipulate objects in the `${GUID}-tasks-prod` project.
- Create two new deployment configurations: `tasks-green` and `tasks-blue` and point both to `tasks:0.0`.
- Turn off automatic building and deployment for both deployment configurations.
- Add placeholder ConfigMaps.
- Associate the ConfigMaps with deployment configurations.
- Expose the deployment configurations as a service (on port 8080) and the blue service as a route.

```
# Set up Production Project
oc new-project ${GUID}-tasks-prod --display-name "${GUID} Tasks Production"
oc policy add-role-to-group system:image-puller system:serviceaccounts:${GUID}-tasks-prod -n ${GUID}-tasks-dev
oc policy add-role-to-user edit system:serviceaccount:${GUID}-jenkins:jenkins -n ${GUID}-tasks-prod

# Create Blue Application
oc new-app ${GUID}-tasks-dev/tasks:0.0 --name=tasks-blue --allow-missing-imagestream-tags=true -n ${GUID}-tasks-prod
oc set triggers dc/tasks-blue --remove-all -n ${GUID}-tasks-prod
oc expose dc tasks-blue --port 8080 -n ${GUID}-tasks-prod
oc set probe dc tasks-blue -n ${GUID}-tasks-prod --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8080/
oc create configmap tasks-blue-config --from-literal="application-users.properties=Placeholder" --from-literal="application-roles.properties=Placeholder" -n ${GUID}-tasks-prod
oc set volume dc/tasks-blue --add --name=jboss-config --mount-path=/opt/eap/standalone/configuration/application-users.properties --sub-path=application-users.properties --configmap-name=tasks-blue-config -n ${GUID}-tasks-prod
oc set volume dc/tasks-blue --add --name=jboss-config1 --mount-path=/opt/eap/standalone/configuration/application-roles.properties --sub-path=application-roles.properties --configmap-name=tasks-blue-config -n ${GUID}-tasks-prod

# Create Green Application
oc new-app ${GUID}-tasks-dev/tasks:0.0 --name=tasks-green --allow-missing-imagestream-tags=true -n ${GUID}-tasks-prod
oc set triggers dc/tasks-green --remove-all -n ${GUID}-tasks-prod
oc expose dc tasks-green --port 8080 -n ${GUID}-tasks-prod
oc set probe dc tasks-green -n ${GUID}-tasks-prod --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8080/
oc create configmap tasks-green-config --from-literal="application-users.properties=Placeholder" --from-literal="application-roles.properties=Placeholder" -n ${GUID}-tasks-prod
oc set volume dc/tasks-green --add --name=jboss-config --mount-path=/opt/eap/standalone/configuration/application-users.properties --sub-path=application-users.properties --configmap-name=tasks-green-config -n ${GUID}-tasks-prod
oc set volume dc/tasks-green --add --name=jboss-config1 --mount-path=/opt/eap/standalone/configuration/application-roles.properties --sub-path=application-roles.properties --configmap-name=tasks-green-config -n ${GUID}-tasks-prod

# Expose Blue service as route to make blue application active
oc expose svc/tasks-blue --name tasks -n ${GUID}-tasks-prod
```

### Update pipeline job

In your `openshift-tasks` directory, update the Jenkinsfile file and copy the content of the [sample](jenkins/jobs/openshift-tasks/Jenkinsfile.bg) pipeline script. Make sure to replace the parameter **prefix** with your GUID. Add the Jenkinsfile file to the repository, then commit and push it into the repository. Start the pipeline.

## Set up job trigger

### Set Up Web Deployment Hook

To automate the build whenever new content is pushed to the **openshift-tasks** repository, set up a Git hook in Gogs. This way, a Jenkins build is triggered whenever code is pushed to the **openshift-tasks** source repository on Gogs.
- Find the authorization token in Jenkins:
  - Open a browser and log in to Jenkins.
  - In the top right corner, click the down arrow next to your username and select **Configure**.
  - Under **API Token** click **Add Token**. Use **gogs** for the name of the token and then copy the displayed token and store it somewhere (You will not be able to retrieve this token again if you don’t copy it now).
  - Note that your full User ID for Jenkins is your cluster username (e.g. **user1**) followed by **-admin-edit-view**. So for this example it would be **user1-admin-edit-view**. You can find it out by clicking on your username in the top-right corner of Jenkins page.
- Create the web hook in Gogs:
  - Open a browser, navigate to the Gogs server, log in, and go to the **CICDLabs/openshift-tasks** repository.
  - Click **Settings**, and then click **Git Hooks**.
  - Click the pencil icon next to **post-receive**.
  - Copy and paste this script into the **Hook Content** field, replacing **<userid>** and **<apiToken>** with your full Jenkins user ID and API token, **<jenkinsService>** with the name of your Jenkins service, and <jenkinsProject> with the name of the OpenShift project your Jenkins service is located in:
        
    ```       
    #!/bin/bash

    while read oldrev newrev refname
    do
    branch=$(git rev-parse --symbolic --abbrev-ref $refname)
    if [[ "$branch" == "master" ]]; then
        curl -k -X POST --user <userid>:<apiToken> http://<jenkinsService>.<jenkinsProject>.svc.cluster.local/job/Tasks/build
    fi
    done
    ```
        
     - This script signals the Jenkins **Tasks** build job each time a commit is made to the master branch.

- Click **Update Hook**. 

### Trigger new build

Committing a new version of the application source code triggers a new build as a result of the Git hook you configured. It is a good practice to increment the version number each time you make changes to your application. You can increment the version number manually or automatically.

- Increment the version number:
  - Change the openshift-tasks source code to make sure you are seeing an updated version of your application after the pipeline completes.
  - Look near line 45 of the code—where the title of the page is rendered—in the $HOME/openshift-tasks/src/main/webapp/index.jsp file.
  - Increment the version number of the project each time code is pushed using Maven—for example, updating the version number (VERSION=1.0) with the next minor or major number (VERSION=1.1):
    ```
    cd $HOME/openshift-tasks
    export VERSION=1.1
    mvn versions:set -f pom.xml -s nexus_settings.xml -DgenerateBackupPoms=false -DnewVersion=${VERSION}
    git add pom.xml src/main/webapp/index.jsp
    git commit -m "Increased version to ${VERSION}"
    git push gogs master
    ```
- Validate that this push triggered a new build in Jenkins.
- After a few minutes, when prompted by the pipeline, approve the switch to the new version.
- Click Proceed.
- Once the pipeline finishes successfully, verify that you can see the updated application.


## Build configuration with Pipeline

Instead of defining the pipeline in Jenkins, you can create an OpenShift build configuration with a Pipeline build strategy. This build configuration must be in the same project as the Jenkins pod (unless you configure the **master-config.yaml** to point to another Jenkins instance).
- To integrate the pipeline with the OpenShift web console, create a build configuration that points to the Jenkinsfile file in your **openshift-tasks** repository.
- You need to create a YAML/JSON file containing your build configuration definition and then create the build configuration from that file.
  - Remember that your **openshift-tasks** repository is private, which means OpenShift needs credentials to access the **Jenkinsfile** file.
  - Create the tasks-pipeline.yaml file:
```
echo "apiVersion: v1
items:
- kind: "BuildConfig"
  apiVersion: "v1"
  metadata:
    name: "tasks-pipeline"
  spec:
    source:
      type: "Git"
      git:
        uri: "http://gogs.${GUID}-gogs.svc.cluster.local:3000/CICDLabs/openshift-tasks"
    strategy:
      type: "JenkinsPipeline"
      jenkinsPipelineStrategy:
        jenkinsfilePath: Jenkinsfile
kind: List" | oc apply -f - -n ${GUID}-jenkins
```
- In the OpenShift web console, switch to your Jenkins project and navigate to **Builds → Build Configs**.
- In the top-right menu use **Actions → Start Build**
- Click **View Log** to view the pipeline progression and follow along in Jenkins.