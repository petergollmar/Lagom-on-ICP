# Instructions: Reactive Platform Kubenetes & Helm minilab

In our Getting started with Reactive Platform blog post we showed how to deploy a sample application that makes use of Akka, Lagom and the Play Framework: https://developer.ibm.com/articles/get-started-with-lightbends-reactive-platform/


In a previous blog, we showed you how to install the Lagom sample Chirper app to IBM Cloud Private, from the ICP catalog. This blog will show you how to build Chirper from source, and deploy it to ICP using a Helm chart. It will then take you through modifying the app, and deploying a new version of one of the microservices.

Login using the id 'skytap' / password A1rb0rn3

Create a namespace in IBM Cloud Private for the Chirper sample application
--------------------------------------------------------------------------

From a Firefox window, click the IBM Cloud Private (ICP) link on the bookmarks toolbar. This takes you to: https://mycluster.icp:8443

Login using admin/admin

Use the menu icon in the top left corner to select Manage -> Namespaces

Create a namespace called reactive

Building and deploying the sample using Docker and Helm
-------------------------------------------------------
We will build the sample from source using sbt, push the generated Docker images to ICP, and then deploy the sample using a Helm chart. We will use a tweaked version of the existing Helm chart that was written to deploy the Chirper sample from publicly available images on Dockerhub. Only some very minor modifications will be need. 

Start a terminal window.

A clone of the git repo at https://github.com/lagom/lagom-java-sbt-chirper-example is located in the ~/reactive-minilab/lagom-java-sbt-chirper-example directory:
'code'    cd ~/reactive-minilab/lagom-java-sbt-chirper-example

Using the sbt build tool build the project, which includes the modules inside.
'code'     sbt clean docker:publishLocal

Once the build has finished, you will have 5 new images in your local Docker registry, front-end, chirp-impl, friend-impl, activity-stream-impl and load-test-impl, all tagged with 1.0.0

Now push push the Docker images to the ICP registry. First tag 4 of the images with the remote registry name (you don't need to push the load-test-impl image):

'code'     docker tag front-end:1.0.0 mycluster.icp:8500/reactive/reactive-sample-front-end:1.0.0

Repeat this for chirp-impl, friend-impl and activity-stream-impl:

'code'     docker tag activity-stream-impl:1.0.0 mycluster.icp:8500/reactive/reactive-sample-activity-stream-impl:1.0.0
'code'     docker tag friend-impl:1.0.0 mycluster.icp:8500/reactive/reactive-sample-friend-impl:1.0.0
'code'     docker tag chirp-impl:1.0.0 mycluster.icp:8500/reactive/reactive-sample-chirp-impl:1.0.0

This assumes you are using the 'reactive' namespace created above. Also note that the image names all need to have 'reactive-sample-...' prepended to them. This is to match the values used in the helm chart (see below).

Now push the images to ICP. First, login to the Docker registry hosted in ICP:
    docker login mycluster.icp:8500

Now push the Docker images to ICP:
    docker push mycluster.icp:8500/reactive/reactive-sample-front-end:1.0.0
    docker push mycluster.icp:8500/reactive/reactive-sample-chirp-impl:1.0.0
    docker push mycluster.icp:8500/reactive/reactive-sample-friend-impl:1.0.0
    docker push mycluster.icp:8500/reactive/reactive-sample-activity-stream-impl:1.0.0

Deploy the complete application using Helm
------------------------------------------
A clone of the git repo at https://github.com/IBM/charts.git is located in the ~/reactive-minilab/charts directory:
     cd ~/reactive-minilab/charts/stable/ibm-reactive-platform-lagom-sample

Now open up the vscode editor:
    code .

The values.yaml file contains application parameters to assist deployment: the Docker image repository name to use, the versions (tags) of the Docker images in the application, the 'pull policy' to determine whether to redeploy the Docker images even if they are already present at the specified version, and the front end hostname to use to access the application.

We're going to override these values so we can use the Container Registry in IBM Cloud Private. Copy the values.yaml file, calling the copied file: overrides.yaml.
In the overrides.yaml file, replace the repository value with: mycluster.icp:8500/reactive
Specify a pullPolicy of Always and replace all the imageTags values with 1.0.0
Specify a hostname of chirper.10.0.0.1.nip.io

The sample is now ready to deploy using helm. First, log into ICP:
    cloudctl login -a https://mycluster.icp:8443 --skip-ssl-validation

    Note: the skip-ssl-validation flag is necessary in this case as the ICP instance has a self-signed certificate.

Follow the prompts and provide user/password as admin/admin and select the 'reactive' namespace you created earlier.

Now run helm:
    helm install --name reactive-chirper -f overrides.yaml --namespace reactive . --tls

The deployment will take a few minutes to fully start up. You can check the progress of the deployments with:
    kubectl get deployments --selector=release=reactive-chirper

When at least 1 pod is available for each of the 4 deployments, you can access the the sample using a web browser, and going to the hostname specified when you installed the Helm chart
https://chirper.10.0.0.1.nip.io
You can look at our previous blog post for more information on how to use the sample.

The sample is now successfully deployed!

Making a change and deploying it as a new version
-------------------------------------------------

We will now update and redeploy the sample, a process which is simplified by the use of the Helm chart. In order to keep this lab fairly small, the change made to the application is just to change the name on the title page. The redeployment process would stay the same for larger changes.

Edit and rebuild the sample: from the directory ~/reactive-minilab/lagom-java-sbt-chirper-example run another vscode window by executing: code .
Edit the file:

front-end/src/main/resources/assets/main.jsx

Search for and modify the string:
    <Link to="/" id="logo">Chirper</Link>
For example, change 'Chirper' to 'My Chirper'

Modify the file: build.sbt so that line 85 becomes:
    version := buildVersion101,

    which is a version string defined earlier in the build.sbt file.

Now rebuild the front-end module
    sbt front-end/clean front-end/docker:publishLocal

Re-tag and re-push the docker images. As only the front-end has been modified, only one Docker image needs to be tagged/pushed.
    docker tag front-end:1.0.1 mycluster.icp:8500/reactive/reactive-sample-front-end:1.0.1

Note that the version on the tag has been bumped here too. This ensures that Helm picks up the change. Now that the updated image has been tagged, push it up to your ICP cluster:

    docker push mycluster.icp:8500/reactive/reactive-sample-front-end:1.0.1

Now use helm to upgrade the release in ICP
Within vscode, edit overrides.yaml file to reference the updated tag. For just the frontend image tag, change '1.0.0' to '1.0.1'

Run the Helm upgrade command
    helm upgrade reactive-chirper . --namespace reactive --tls

Kubernetes will delete the old front end pod, and replace it with a new pod running the updated Docker images. This will take a moment or two. You can check on the status of the rollout with:
    watch -n2 kubectl rollout status deployment/frontend-deploy-reactive-chirper

When this reports the rollout as succesful, you will be able to see the updated front end in your browser.

Where Next
Learn more about Lagom framework https://www.lagomframework.com/
