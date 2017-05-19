# Application Onboarding Guide

## tl;dr I just want to run it

- copy `files/example-app-inventory-step-3.json` into the `vars` directory
- execute `run.sh`

## Introduction

This guide assumes:
- You are using [the CI-CD Starter](https://github.com/rht-labs/examples/tree/master/ci-cd-starter) or a version of it. It's possible to take a different approach, but we haven't documented that
- You are onboarding a Java app that builds into an uber jar (TODO - guide for web apps)
- Your app exposes a REST API on port `8080`
- Your app does not require external services (e.g. a database) to be available in order to boot up and operate
- You have installed the `oc` binary that corresponds to the OpenShift cluster you are using
- The necessary tools to build your app are installed in a local or remote workspace which you can access
- You have basic familiarity with OpenShift objects and how they are configured via templates
- You have basic familiarity with `Jenkinsfile`

As an example, we will use a Vert.x REST API, built with maven into an uber jar, and deployed to openshift with the [Java S2I image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html-single/red_hat_java_s2i_for_openshift/).


## Getting Started

 This guide will start you down the path of deploying applications to OpenShift using version controlled templates and ansible. With the getting starting material we've provided here, we think this is a simple, yet robust approach for both developers and operators.

 But that's a decision each team needs to make for itself. So before going further, and have a conversation with your team that this is the approach you want to take. It might help to take a look at [this example](files/example-app-inventory-step-3.json) of what an app looks like when managed with this approach. We refer to these documents as an "Application Inventory." In practical terms, it's just a `.json` file that provides a list of projects and the templates to apply to those projects in a way that our ansible automation can consume.

If the team decides to move forward with this approach, there are few more things you need to do :

- Decide as a team how many stages of your pipeline you want. We default to development (dev), test (test), and user acceptance test (uat).
- Decide as a team what you want the projects to be called for pipeline stage. This example uses acme-dev, acme-test, and acme-uat. It's unlike you'll prefix your projects with acme, we encourage you to follow this convention.
- Fork this repository into your own VCS. This will be a shared repository between dev and ops, so make sure you agree on it's location, naming, access rights etc.


## Verify the Application and Container Image Builds

In this section, we'll verify you app can be compiled, packaged and built into a container image to run on OpenShift.

- Clone your source code and build it. Ensure that:
  - the build passes
  - the build produces a target directory with a single jar
  - the jar created is a uber jar for your app. More than one jar in the target directory can cause [issues with S2I](https://issues.jboss.org/browse/OSFUSE-580?workflowName=jira&stepId=1). The [maven fabric plugin](https://maven.fabric8.io/) and the [vert.x maven plugin](https://vmp.fabric8.io/) can help you achieve this objective.
- Create a new openshift project to test S2I works correctly
  - `oc new-project build-test`
- Create a S2I build in the test project
  - `oc new-build --binary=true --name=test -i=redhat-openjdk18-openshift`
- Run the S2I build, using the output your local app build in step 1. If the uber-jar is not created in a `target` folder in the root directory, update the command below accordingly.
  - `oc start-build test --from-dir=target/ --follow`
- Create an application to consume the image you just built.
  - `oc new-app test`
- Expose the applications HTTP endpoint so you can verify it deployed correctly
  - `oc expose svc test`
- Navigate to your test project in the OpenShift Console. Inspect the URL for your app and ensure it produces the correct results. Check the log of the deployment and make sure it's error free.
- If everything worked correctly, delete the test project:
  - `oc delete project build-test`


## Initialize the Application Inventory

Take a look a [this skeleton Application Inventory](files/example-app-inventory-step-1.json). At this time, it's just a list of projects, with some information on how those projects should display themselves in the OpenShift Console. We're going to use this example to start creating your own Application Inventory.

- Create a new file in this repository - `vars/application-inventory.json`
- Copy the contents of [the example](files/example-app-inventory-step-1.json) into `vars/application-inventory.json`
- Update the contents to reflect the project names your team decided upon. This guide assumes you are building on [the CI-CD Starter](https://github.com/rht-labs/examples/tree/master/ci-cd-starter); and therefore, you should leave the `labs-ci-cd` project in place as it is configured by the `CI-CD Starter` to have Jenkins, Nexus & other tools integrated.
- Create the projects in OpenShift by executing `run.sh`. Verify the projects are created with the OpenShift console or using `oc`. This should begin a recurring feedback loop of updating the Application Inventory and then using the automation tooling to create the corresponding state in OpenShift.


## Add the Jenkinsfile to Your Application Source Code

OpenShift natively supports a [Pipeline Build Strategy](https://docs.openshift.com/container-platform/3.5/architecture/core_concepts/builds_and_image_streams.html#pipeline-build) which allows OpenShift to manage Jenkins Pipeline builds just like a normal S2I build. This section will show you how to create the `Jenkinsfile` for your app.

- Review [this example Jenkinsfile](builds/helloworld-java-rest-api/Jenkinsfile). TODO - explain what's happening here.
- Copy [this example Jenkinsfile](builds/helloworld-java-rest-api/Jenkinsfile) into a working copy of your application's source code. By default, you should place it in the root of your source code hierarchy, which hopefully the root directory of the repository. If the root of the source code is further nested in the repository, place the `Jenkinsfile` alongside the source, but remember you'll need to set a property for this later on.
- Now it's time to customize your `Jenkinsfile` for your app. Update the values of `env.DEV_PROJECT`, `env.TEST_PROJECT` and `env.UAT_PROJECT` to reflect the project names you used in the Application Inventory in the previous step
- Update `env.SOURCE_CONTEXT_DIR` to the root of your source code hierarchy. If source code hierarchy starts at the root directory of the repository (which again should usually be the case), set this value to `""`
- Update `env.UBER_JAR_CONTEXT_DIR` to the location where the application's uber-jar will be built. This value is relative to `env.SOURCE_CONTEXT_DIR`
- Update `env.MVN_COMMAND` with the command you need to build your app and deploy it an artifact repository. Be sure to include switches like `-D` or `-P` here.
- Commit your changes back to the source repository and push them (via PR if necessary) to the upstream repository that will be build by Jenkins later on.


## Add the S2I Build and Pipeline Build to the Application Inventory

With the projects in place, we'll now use the provided templates to provision the OpenShift objects that define the Jenkins Pipeline, as well as the S2I build that will create our container image. Again, we'll leverage [an example](files/example-app-inventory-step-2.json) to help get us started.

- Update your the `labs-ci-cd` project in `vars/application-inventory.json` with the `templates` section found on lines 9-19. Don't forget to add a comma at the end of your line 8.
- The `filename` property declares that the template is found in a file in your local repository, [files/java-app-build-template.json](files/java-app-build-template.json). Open this file and read through it. You'll notice it has two `BuildConfigs`, one for the Jenkins pipeline, and one for S2I build to create the container image. Also notice that the template uses several parameters, which you'll need to populate with values that correspond to your app.
- Update the value of `PIPELINE_SOURCE_REPOSITORY_URL` with the git repository url for your app. Note that this will be used by Jenkins to clone your source code, so the VCS needs to be visible to Jenkins on the network, and if using `ssh`, your Jenkins needs to be configured with the `ssh-key`.
- Update `PIPELINE_SOURCE_REPOSITORY_REF` with the git ref for your app. If using `master`, you can remove this line if you wish.
- If your `Jenkinsfile` is not in the root directory, update `PIPELINE_CONTEXT_DIR` with it's location. If the `Jenkinsfile` is in the root directory, you have the option of deleting this line.
- Update `NAME` with the value you want use for the name of all of your objects.
- Create the OpenShift objects by executing `run.sh`. This is a safe operation because ansible is idempotent, so it will leave your existing projects in place. Verify that the Pipeline is created and running using OpenShift console. When it gets to the `Build Image` step, verify that the S2I build is run.

The Jenkins Pipeline should pause at a step called `Deploy to Dev`. At this point, you can move on to the last section


## Add the Deployment Objects to the Application Inventory

With the project in place, the Jenkins Pipeline running, and the container image built, it is time to configure our `dev`, `test` & `uat` projects with the template that will create the OpenShift objects needs to deploy and test the container image. Again, let's first look at [an example](files/example-app-inventory-step-3.json). This template should look straightforward, apart from the `RoleBinding` objects. One of the objects enables the Jenkins `ServiceAccount` to tag the container image intp the `ImageStream` in the namespace, which is how promotions are handled. The other `RoleBinding` provides the proper access rights for applications using [the fabric8 kubernetes client](https://github.com/fabric8io/kubernetes-client).

- Update your `dev` project in `vars/application-inventory.json` with the `templates` section found on lines 23-31. Don't forget to add a comma at the end of your line 22.
- Update `APP_NAMESPACE` to reflect the name of the project the template is deployed in
- If you are using [the CI-CD Starter](https://github.com/rht-labs/examples/tree/master/ci-cd-starter), you do not need to update `PIPELINES_NAMESPACE`. If you are not, then update this value to the project where the application container image was built.
- Update `NAME` with the value you want use for the name of all of your objects.
- Repeat the above steps for the `test` & `uat` projects. Then move on
- Create the OpenShift objects by executing `run.sh`. This is a safe operation because the ansible automation uses `oc apply` under the covers to process our template, which delegates the responsibility of idempotence to OpenShift. Verify that the `DeploymentConfig` and related objects are created in the `dev`, `test` & `uat` projects.
- To promote the container image, navigate back to the Jenkins Pipeline and approve the deployments. After each deploy, verify your container image comes up properly and the log is free of errors.
- If all goes correctly, commit your changes and push them (via PR if needed) to the upstream repository

## Adding More Apps

Simply update your existing Application Inventory with new apps following this guide. You may need to create new templates or modify the existing ones. That perfectly OK and our design is intended to support you doing that.

## THE END