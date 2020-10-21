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