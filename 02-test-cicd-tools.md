# Test the CI/CD environment

In this part we are going to test tools deployed in the previous part. 

## Clone and configure sample application

Clone the sample application

```
cd $HOME
git clone https://github.com/redhat-gpte-devopsautomation/openshift-tasks.git
cd $HOME/openshift-tasks
```

Set up `nexus_settings.xml` for local builds, making sure that `<url>` points to your specific Nexus URL.

```
cat > nexus_settings.xml <<EOF
<?xml version="1.0"?>
<settings>
  <mirrors>
    <mirror>
      <id>Nexus</id>
      <name>Nexus Public Mirror</name>
      <url>http://$(oc get route nexus -n ${GUID}-nexus --template='{{ .spec.host }}')/repository/maven-all-public/</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>
  <servers>
  <server>
    <id>nexus</id>
    <username>admin</username>
    <password>app_deploy</password>
  </server>
</servers>
</settings>
EOF
```

Also change the `nexus_openshift_settings.xml` to point to the internal Nexus URL

```
cat > nexus_openshift_settings.xml <<EOF
<?xml version="1.0"?>
<settings>
  <mirrors>
    <mirror>
      <id>Nexus</id>
      <name>Nexus Public Mirror</name>
      <url>http://nexus.$GUID-nexus.svc.cluster.local:8081/repository/maven-all-public/</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>
  <servers>
    <server>
      <id>nexus</id>
      <username>admin</username>
      <password>app_deploy</password>
    </server>
  </servers>
</settings>
EOF
```

Commit and push the updated settings files to Gogs:

```
git commit -m "Updated Nexus Settings" nexus_settings.xml nexus_openshift_settings.xml
```

### Push the sample app into Gogs


* Open your Gogs Server in a web browser.
* Log in to your Gogs server with your new user account.
* Create an organization named **CICDLabs**.
* Under the **CICDLabs** organization, create a repository called **openshift-tasks**.
  * Do not make this a Private repository.
* Configure your new repository as remote repository in the cloned repository and push it.
  * Make sure to replace **<gogs_user>** and **<gogs_password>** with your credentials.
  
  ```
  cd $HOME/openshift-tasks

  git remote add gogs http://<gogs_user>:<gogs_password>@$(oc get route gogs -n ${GUID}-gogs --template='{{ .spec.host }}')/CICDLabs/openshift-tasks.git
  
  git push -u gogs master
  ```


## Test the local build

Make sure you can build the openshift-tasks application:

```
cd $HOME/openshift-tasks
mvn clean install -DskipTests=true -s ./nexus_settings.xml
```

Run the unit tests:

```
mvn test -s ./nexus_settings.xml
```

Run the Maven deploy tests:

```
mvn -s ./nexus_settings.xml deploy -DskipTests=true \
-DaltDeploymentRepository=nexus::default::http://$(oc get route nexus -n ${GUID}-nexus --template='{{ .spec.host }}')/repository/releases
```

Run the code analysis tests (you may ignore any errors complaining about a missing node)

```
mvn sonar:sonar -s ./nexus_settings.xml -Dsonar.host.url=http://$(oc get route sonarqube -n ${GUID}-sonarqube --template='{{ .spec.host }}')
```

Run the Nexus Docker registry tests by copying the `python` image into the Nexus Docker registry 

```
export REGISTRY=default-route-openshift-image-registry.apps.$(oc whoami --show-server | cut -d. -f2- | cut -d: -f1)


podman login -u $(oc whoami) -p $(oc whoami -t) ${REGISTRY}
skopeo inspect docker://${REGISTRY}/openshift/python
podman logout ${REGISTRY}


skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds=openshift:$(oc whoami -t) --dest-creds=admin:app_deploy docker://${REGISTRY}/openshift/python docker://$(oc get route nexus-registry -n ${GUID}-nexus --template='{{ .spec.host }}')/openshift/python
```