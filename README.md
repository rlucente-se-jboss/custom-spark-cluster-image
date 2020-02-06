# How To Customize the Spark Cluster Image
Here are the steps necessary to deploy a spark cluster using a
customized Spark container image.  The image can contain additional
jar files.

## Create a working directory
Create a working directory to build a customized spark container
image that contains the desired jar files.  For this example, we'll
create an empty file called `mothra.jar`.

    mkdir example
    cd example
    touch mothra.jar

You can add other jar files here.

## Create a Dockerfile
Create a text file named `Dockerfile` to build a custom spark cluster
image and make sure it contains the following content:

    FROM quay.io/jkremser/openshift-spark:2.3-latest
    COPY mothra.jar /opt/spark/examples/jars
    COPY mothra.jar /opt/spark/jars

You can add any number of jar files to the base image.

## Build the custom spark image on OpenShift
Login in as a developer to OpenShift and then build the custom image
using your `Dockerfile`.  In the `oc login` command below, make
sure that the `-u` and `-p` options have the correct username and
password, respectively, for an unprivileged user on your OpenShift
cluster.  In this example, the user is `developer`.

    oc login -u developer -p developer https://api.crc.testing:6443
    oc new-project spark-example
    oc new-build . --name custom-spark --strategy=docker
    oc start-build custom-spark --from-file=Dockerfile
    
## Clone the spark operator project
Clone the spark operator github project that allows you to configure
which container image is used to launch a spark cluster.

    git clone https://github.com/rlucente-se-jboss/spark-operator.git
    cd spark-operator

## Determine the spark image identity
Use the following command to get the repository of the custom spark
image we just created with the additional jars.

    oc get -o template is/custom-spark --template={{.status.dockerImageRepository}} && echo

Copy the result to the clipboard.  On my cluster, the image repository was `image-registry.openshift-image-registry.svc:5000/spark-example/custom-spark`.

## Edit the spark operator manifest to use the custom spark image
Edit the file `manifest/operator.yaml` and change the value for the
`DEFAULT_SPARK_CLUSTER_IMAGE` environment variable to the clipboard
content.  On my system, the relevant lines look like:

    - name: DEFAULT_SPARK_CLUSTER_IMAGE
      value: "image-registry.openshift-image-registry.svc:5000/spark-example/custom-spark:latest"

## Deploy the spark operator
As a cluster administrator, deploy the spark operator into the same
project.  In the `oc login` command below, make sure that the `-u`
and `-p` options have the correct username and password, respectively,
for a cluster administrator on your OpenShift cluster.

    oc logout
    oc login -u kubeadmin -p BdTY8-Vra64-gD3cI-dAPS6 https://api.crc.testing:6443
    oc project spark-example
    oc apply -f manifest/operator.yaml    

## Deploy your spark cluster using the custom spark image
Edit the file `examples/cluster.yaml` and set the number of spark
cluster workers and masters that you desire.  Deploy the cluster
using the following command:

    oc apply -f examples/cluster.yaml

Wait for the spark cluster pods to start.  You can check the status
to see if they're ready using the command:

    oc get pods

Once the pods are running for the spark cluster, you can expose the
spark management user interface using the command:

    oc expose svc/my-spark-cluster-ui

You can get the URL to management UI via the command:

    oc get route

