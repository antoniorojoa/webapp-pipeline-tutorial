# Web App Pipeline Tutorial

This is a tutoral for deploying your Modern Web Applications using Openshift pipelines.  It is the next part in the Modern Web Application blog series located [here](https://developers.redhat.com/blog/2018/10/04/modern-web-apps-openshift-part-1/).  This tutorial assumes there is some knowledge of the blog series, especially the part on Chained Builds


## What are Pipelines

This post isn't going to go to far into what pipelines are, so if you would like more information on Pipelines, check out the [official tutorial here](https://github.com/openshift/pipelines-tutorial)

So the short answer:


> OpenShift Pipelines are a cloud-native, continuous integration and delivery (CI/CD) solution for building pipelines using Tekton. Tekton is a flexible, Kubernetes-native, open-source CI/CD framework that enables automating deployments across multiple platforms (Kubernetes, serverless, VMs, etc) by abstracting away the underlying details.



## Setup

To effectively follow this, there is some initial setup.

First, you will need an Openshift 4 cluster.  I've been using Code Ready Containers(crc) to setup this environment.  Follow [this link](https://cloud.redhat.com/openshift/install/crc/installer-provisioned) to follow the setup instructions.

Once your cluster is up and running, you will also need to install the Pipeline Operator, which is really only a couple clicks.  Follow this guide to [install the Operator](https://github.com/openshift/pipelines-tutorial/blob/master/install-operator.md)

You will also need the Tekton CLI(tkn), which you can get from [here](https://github.com/tektoncd/cli#installing-tkn)

## Getting Started

The application that we are going to eventually deploy, will be this [basic React example](https://github.com/nodeshift-starters/react-pipeline-example), which was created by running the "create-react-app" cli tool.

The application repo also has a "k8s" directory, which has some kubernetes/openshift yamls, which can be used to deploy the application.

### Creating Some Tasks

So, first thing we are going to do, is to create a new project namespace on our Openshift cluster for this example.  We are going to call it **webapp-pipeline**

```
$ oc new-project webapp-pipeline
```

Before we get started with setting up the Pipeline, lets first create a couple **Tasks** that will help us out with the applying and deploying those yamls from the applications k8s directory.

To add them, run:

```
$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/tasks/update_deployment_task.yaml
$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/tasks/apply_manifests_task.yaml
```

You can then see the added tasks by running:

```
tkn task ls
```

_note: these Tasks are "local" to the namespace that we created earlier_

### Creating Some ClusterTasks

Next, we are going to add a couple **ClusterTasks** which can be used from any namespace.

The first will be responsible for checking out our Application git repo and using the **ubi8-s2i-web-app** image to "build" the application and store the resulting image in Openshifts internal docker registry.

This ClusterTask is called **s2i-web-app** and can be created by running the following command:

```
$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/clustertasks/s2i-web-app-task.yaml
```

We also need to add the ClusterTask responsible for taking the "built" web application files from the s2i-web-app task and feed them into an NGINX image.  This ClusterTask is basically the "Chained Build" part of the aforementioned blog posts.

This ClusterTask is called **webapp-build-runtime** and can be created by running the follow command:

```
$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/clustertasks/webapp-build-runtime-task.yaml
```

Once both of those have been created, you can see them by running `tkn clustertask ls`

### Setting up Resources

Before we create the Pipeline, we will create some resources that we will pass into the Pipeline.  This helps abstract away some of the more specific bits.

The first Resource we need, is the Git Repo where our application is.  This can look something like this:

```
# This resource is the location of the git repo with the web application source
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: web-application-repo
spec:
  type: git
  params:
    - name: url
      value: https://github.com/nodeshift-starters/react-pipeline-example
    - name: revision
      value: master
```

This **PipelineResource** is of the **git** type and we can see in the params section, the url targets a specific repo, and we are also specifiying the master branch(this is optional, but i'm including it for completeness)

The next resource we need is an Image(docker image) that we will store the result of the s2i-web-app task.  This might look something like this:

```
# This resource is the result of running "npm run build",  the resulting built files will be located in /opt/app-root/output
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: built-web-application-image
spec:
  type: image
  params:
    - name: url
      value: image-registry.openshift-image-registry.svc:5000/webapp-pipeline/built-web-application:latest
```

This **PipelineResource** is of the **image** type and the value of the url is pointing to the internal Openshift Image Registry, specifically the one in the **webapp-pipeline** namespace.  If you are using a different namespace, then change that value accordingly.

The last resource, will also be an image type, and this will be the resulting NGINX image that we will eventually deploy.

```
# This resource is the image that will be just the static html, css, js files being run with nginx
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: runtime-web-application-image
spec:
  type: image
  params:
    - name: url
      value: image-registry.openshift-image-registry.svc:5000/webapp-pipeline/runtime-web-application:latest
```

Again, this resource will store the image inside the internal Openshift registry in the webapp-pipeline namespace.

To create all these resources at once, you can run this create command:

```
$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/resources/resource.yaml
```

You can see the created resources using:

```
$ tkn resource ls
```

### Putting the Pipeline Together

Now that we have all the pieces, lets put them together in our Pipeline.  You can create the pipeline by running the following command:

```
$ oc create -f https://raw.githubusercontent.com/nodeshift/webapp-pipeline-tutorial/master/pipelines/build-and-deploy-react.yaml
```

Before we run it, lets take a look at the pieces.

First, then name:

```
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: build-and-deploy-web-applications
```

Then, in the **spec** section, we see specify the resources that we created earler:

```
spec:
  resources:
    - name: web-application-repo
      type: git
    - name: built-web-application-image
      type: image
    - name: runtime-web-application-image
      type: image
```

Then we create some tasks for our pipeline to run.  The first task we want to run is the s2i-web-app ClusterTask we created earlier.

```
tasks:
    - name: build-web-application
      taskRef:
        name: s2i-web-app
        kind: ClusterTask
```

This task takes an input, which is the git resource, and an output, which is the built-web-application-image resource.  We also pass in a parameter to tell our ClusterTask we don't need to verify TLS becuase we are using self-signed certs.

```
resources:
        inputs:
          - name: source
            resource: web-application-repo
        outputs:
          - name: image
            resource: built-web-application-image
      params:
        - name: TLSVERIFY
          value: "false"
```


The next task, has a similar setup, but this time calls the **webapp-build-runtime** ClusterTask we created earlier.

```
name: build-runtime-image
    taskRef:
      name: webapp-build-runtime
      kind: ClusterTask
```

Similar to our previous task, we are passing in a resource,  but this time it is the **built-web-application-image**(this was the output of our previous task) and again we are specifying an image as the output.  This task should run after the previous task, so we add the **runAfter** field

```
resources:
        inputs:
          - name: image
            resource: built-web-application-image
        outputs:
          - name: image
            resource: runtime-web-application-image
        params:
        - name: TLSVERIFY
          value: "false"
      runAfter:
        - build-web-application
```
The next task is responsible for applying the Service, Route and Deployment yaml files that live in the "k8s" directory of the web application.

And the last task will update the deployment, with the newly created image.


### Finally Run the Pipeline

To run the pipeline, run the following command:

```
$ tkn pipeline start build-and-deploy-web-applications
```

The CLI will turn interactive, and you will need to choose the appropriate resources at each prompt.

Choose **web-application-repo** for the git resource

**built-web-application-image** for the first image resource  and **runtime-web-application-image** for the second image resource

You can check the status of the pipeline by running:

```
$ tkn pipeline logs -f
```
To get the exposed route, you can do this little command:

```
$ oc get route react-pipeline-example --template='http://{{.spec.host}}'
```
