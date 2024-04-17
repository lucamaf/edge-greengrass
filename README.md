# What is Greengrass

IoT AWS Greengrass expands AWS to edge devices so they may process data locally while continuing to use the cloud for analytics, administration, and storage.

Greengrass creates a safe link to the cloud by enabling local device communications via the MQTT protocol. Devices can then connect to the AWS Greengrass Core via the AWS IoT.

IoT AWS works by authorizing the device information to be filtered, Greengrass may be configured to let just the necessary information be broadcast back into the cloud.

# Red Hat Device Edge

 
Red Hat Device Edge (**RHDE**) is for organizations that need small-form-factor edge devices and support for bare metal, virtualized, or containerized applications.  
Red Hat Device Edge helps address many of the emerging questions around large-scale edge computing at the device edge by incorporating Kubernetes features in a new, smaller, lighter-weight footprint (based on Microshift community project) with an edge-optimized Linux operating system built from the world’s leading enterprise Linux platform in RHEL.  

## Use case

It might be that you want to keep control of the IoT applications and connections to the IoT devices while taking advantage of a stable and secure Edge platform.  
In this case integrating Greengrass and RHDE is fundamental. What are the steps needed?

### Build Greengrass container image

You can use as a reference this [documentation](https://docs.aws.amazon.com/greengrass/v2/developerguide/build-greengrass-dockerfile.html).  
Download the latest version from [Github](https://github.com/aws-greengrass/aws-greengrass-docker).  
You can then build the container image with:  

`$ podman build -t "x86_64/aws-iot-greengrass:2.5.3" -f Dockerfile`

If everything goes well you should see the image created in the local registry  

```
$ podman images
REPOSITORY                                                       TAG               IMAGE ID      CREATED        SIZE
localhost/x86_64/aws-iot-greengrass                              2.5.3             43d265d13c0d  6 days ago     769 MB
```

### Run Greengrass container on Podman

To run the container and connect it to AWS you would need an AWS account with permissions to provision the AWS IoT and IAM resources for a Greengrass core device.  
You can use long-term or temporary credentials for IAM.  

1. First of all create a folder where the container can access the credentials  
   
   `$ mkdir ./aws`
2. Create a credentials file  
   
    `$ vi ./aws/credentials`  

3. Add your own credentials  
    ```
    [default]
    aws_access_key_id = EXAMPLE
    aws_secret_access_key = EXAMPLEKEY
    aws_session_token = AQoEXAMPLEH4aoAH0gNCAPy...truncated...zrkuWJOgQs8IZZaIv2BXIa2R4Olgk
    ```
4. Create the environment file  
   
    `$ vi .env`  

5. Set the environment variables that will be passed to the Greengrass Core software installer inside the container to customize device naming, device group, aws region placement and iam role  
    ```
    GGC_ROOT_PATH=/greengrass/v2
    AWS_REGION=eu-central-1
    PROVISION=true
    THING_NAME=gg-microshift
    THING_GROUP_NAME=rh1
    TES_ROLE_NAME=GreengrassV2TokenExchangeRole
    TES_ROLE_ALIAS_NAME=GreengrassCoreTokenExchangeRoleAlias
    COMPONENT_DEFAULT_USER=ggc_user:ggc_group
    ```
    You can find more info [here](https://docs.aws.amazon.com/greengrass/v2/developerguide/run-greengrass-docker-automatic-provisioning.html)  

6. Execute the container on Podman  
   
    `$ podman run --rm --init -it --name aws-iot-greengrass -v /<home-greengrass-dir>:/root/.aws/:Z --env-file .env -p 8883 localhost/x86_64/aws-iot-greengrass:2.5.3`  

    As you can see I'm using:
    * `-it` launching the container in interactive mode to follow the installation  
    * `--rm` removing the container as soon as it exits
    * `-v` mounting a volume to the container with the credentials
    * `--env-file` passing the env variables to customize installation
    * `-p` exposing the port dedicated to MQTT traffic
7. If everything goes well you should see something like this in the container logs  
    ```
    Running Greengrass with the following options: -Droot=/greengrass/v2 -Dlog.store=FILE -Dlog.level= -jar /opt/greengrassv2/lib/Greengrass.jar --provision true --deploy-dev-tools false --aws-region us-east-1 --start false  
    Installing Greengrass for the first time...  
    ...  
    ...  
    Launching Nucleus...  
    Launched Nucleus successfully.  
    ```

8. Once installed correctly you should see something like this in your   
    [AWS Console](https://github.com/lucamaf/edge-greengrass/blob/main/aws-console.png?raw=true)    

### Run Greengrass container on RHDE

You can also deploy the greengrass container over K8S and use it with RHDE.  
To simplify creating a deployment artifact you can use one of the newest features of Podman.  
You can generate a K8S *Deployment* from a container!  

1. To generate the corresponding YAML simply run:  

    `$ podman generate kube --type deployment -s aws-iot-greengrass`  

    In this case the container is named *aws-iot-greengrass* and I've added `-s` to also generate an associated Service.  

2. You can then push the image to a public or a private registry with:  

    `$ podman push  localhost/x86_64/aws-iot-greengrass:2.5.3 docker://quay.io/YOUR-USERNAMEME/gg:2.5.3 --creds username:password`  

3. Given that the container needs to use a local volume (to access a directory with credentials) I could leverage the *ConfigMap* object and store the credentials there:  

    `$ oc create configmap gg-config --from-file=.aws/`  

    > remember the credentials file I created before

4. I can then modify the generated deployment yaml and include the configmap directory reference like shown in the [file](https://github.com/lucamaf/edge-greengrass/blob/main/gg-deployment-configmap.yml)
5. And eventually run greengrass on Microshift inside RHDE by creating the workload:  

    `$ oc create -f gg-deployment-configmap.yml`  

    and voilà the containes is running on Microshift:  

    ```
    $ oc get all
    NAME                                                 READY   STATUS    RESTARTS   AGE
    pod/aws-iot-greengrass-deployment-7cdf4fc48d-f9gcm   1/1     Running   0          7s

    NAME                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
    service/aws-iot-greengrass   NodePort    10.43.146.89   <none>        8883:32688/TCP   7s
    service/kubernetes           ClusterIP   10.43.0.1      <none>        443/TCP          183d

    NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/aws-iot-greengrass-deployment   1/1     1            1           7s

    NAME                                                       DESIRED   CURRENT   READY   AGE
    replicaset.apps/aws-iot-greengrass-deployment-7cdf4fc48d   1         1         1       7s
    ```