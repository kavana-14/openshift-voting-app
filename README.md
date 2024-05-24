# Deploying microservices in Openshift

This project contains reworked voting application from [Docker Official Samples](https://github.com/dockersamples/example-voting-app), to be able to deploy them easily in Openshift.

## Application architecture

![architecture](docs/img/architecture.png)

---

## Requirements

- An Openshift cluster up and running
- [oc CLI](https://medium.com/r/?url=https%3A%2F%2Fmirror.openshift.com%2Fpub%2Fopenshift-v4%2Fclients%2Foc%2Flatest%2F)
- [git](https://medium.com/r/?url=https%3A%2F%2Fgit-scm.com%2Fdownloads)

>I will use Openshift dedicated for this example, which is a minimal OpenShift 4 cluster run on Red Hat developer sandbox. 
This could be an Openshift (version 4).
At the moment, I use oc CLI 4.14.0.
Using git repositories allows us to keep a track of what has been done, to version all YAML files, as we can do for source code.
Using and understanding the oc CLI will make it possible to automate the creation and deployment of applications.

---

## Deploy voting-app

Before starting, if you don't know the basics of Openshift or Kubernetes, let me redirect you to the official documentation of the objects that will be used, stored and versionned:
- [Pod](https://docs.openshift.com/container-platform/4.6/rest_api/workloads_apis/pod-core-v1.html)
- [DeploymentConfig](https://docs.openshift.com/container-platform/4.6/rest_api/workloads_apis/deploymentconfig-apps-openshift-io-v1.html)
- [Service](https://docs.openshift.com/container-platform/4.6/rest_api/network_apis/service-core-v1.html)
- [Route](https://docs.openshift.com/container-platform/4.6/rest_api/network_apis/route-route-openshift-io-v1.html)
- [ImageStream](https://docs.openshift.com/container-platform/4.6/rest_api/image_apis/imagestream-image-openshift-io-v1.html)
- [BuildConfig](https://docs.openshift.com/container-platform/4.6/rest_api/workloads_apis/buildconfig-build-openshift-io-v1.html)

The last things you should know:
- Deployments may take a few minutes, the cluster needs to pull images, to build images from Dockerfile or from Source to Image.
- When your deployments are performed, you can access the result and vote application through the Openshift routes and you should see something like this:
![app](docs/img/vote-result.png)

---

## Getting started

Now, let's create an Openshift project and deploy redis and postgreSQL databases.
Connect to your Openshift cluster by using the command `oc login --token=<token> --server=<SERVER_URL>` with your credentials and replace <token> and <SERVER_URL> before run any command with the oc client.

```bash
# Create voting-app project
$ oc new-project voting-app

# Create non persistent PostgreSQL database
$ oc new-app postgresql:latest --name=db \ 
-e POSTGRESQL_DATABASE=postgres \ 
-e POSTGRESQL_USER=postgres \ 
-e POSTGRESQL_PASSWORD=postgres \ 
-e POSTGRESQL_PORT=5432 
 

# Create non persistent Redis
$ oc new-app redis:latest --name=redis \
-e REDIS_PASSWORD=redis \
-e REDIS_PORT=6379 
```

By default, there are some templates already present on your cluster.
But if you have errors and **postgresql-ephemeral** or **redis-ephemeral** are not available on your cluster, you can use templates located in openshift-specifications/templates:
```bash
# Create databases from template files
$ oc process -f openshift-specifications/templates/postgresql-ephemeral-template.yaml \
    -p DATABASE_SERVICE_NAME=db \
    -p POSTGRESQL_USER=postgres \
    -p POSTGRESQL_PASSWORD=postgres \
    -p POSTGRESQL_DATABASE=postgres | oc apply -f - -n voting-app
$ oc process -f openshift-specifications/templates/redis-ephemeral-template.yaml \
    -p REDIS_PASSWORD=redis | oc apply -f - -n voting-app
```

What we will deploy:
![deployments](docs/img/deployments.png)

Login to your openshift registry to push the image to the registry. 
```bash
$ oc registry login 

$  sudo cp /run/user/1000/containers/auth.json ~/.docker/config.json 

#From the docker, login to your openshift registry. oc registry login displays the path to registry, use that path for login
$ docker login <openshift-registry-path>
```
###  With Dockerfile
In this step, we are going to deploy the same application but with some changes: use of Dockerfile in your Git repository. It's Openshift that will take care of building our images. 
Let's update Openshift objects to try out this method and trigger container image building.
This method allows you to update your Dockerfile and source code at the same time in your SCM, you keep control over everything and it's all versioned.

```bash
#Deploy Polling Application
$ docker build -t openshift-registry-path/project-name/vote . 
 
$ docker push openshift-registry-path/project-name/vote:latest 

$ oc new-app openshift-registry-path/project-name/vote:latest --name=vote \ 
-e REDIS_HOST=redis \ 
-e REDIS_PASSWORD=redis \ 
-e REDIS_PORT=6379 

#Deploy Worker Application 
$ docker build -t openshift-registry-path/project-name/worker . 
 
$ docker push openshift-registry-path/project-name/worker:latest 
 
oc new-app openshift-registry-path/project-name/worker:latest --name=worker \ 
-e REDIS_HOST=redis \ 
-e REDIS_PASSWORD=redis \ 
-e REDIS_PORT=6379 \ 
-e POSTGRESQL_HOST=db \ 
-e POSTGRESQL_PORT=5432 \ 
-e POSTGRESQL_DATABASE=postgres \ 
-e POSTGRESQL_USER=postgres \ 
-e POSTGRESQL_PASSWORD=postgres 

#Deploy Result Application
$ docker build -t openshift-registry-path/project-name/result .

$ docker push openshift-registry-path/project-name /result:latest 

$ oc new-app openshift-registry-path/project-name/result:latest --name=result \ 
-e POSTGRESQL_HOST=db \ 
-e POSTGRESQL_PORT=5432 \ 
-e POSTGRESQL_DATABASE=postgres \ 
-e POSTGRESQL_USER=postgres \ 
-e POSTGRESQL_PASSWORD=postgres

#Expose Services 
$ oc port-forward poll 8080:80 
$ oc port-forward result 8080:80 

$ oc expose service/vote 
$ oc expose service/result 
```

Once the builds are completed, you can check vote and result application are running.

## Conclusion
There are several ways to migrate applications, and there is not a best way to do that, it depends on how you want to manage your application lifecycle.
Do you already have tools that build images and store them, and you don't want to migrate everything to Openshift? Then deploy your applications using your container images.
Do you want to deploy applications from their source code? If you want to take care of a Dockerfile, choose to build your application with it. If not, Source to Image allows you to deploy already secured container images in your cluster.
There are other solutions you can implement, a combination of Dockerfile and S2I for example, or launching CI/CD pipelines with other tools and so on.
