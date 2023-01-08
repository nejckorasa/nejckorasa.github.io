---
title: "How to schedule Spark jobs in Kubernetes with Airflow"
date: 2022-12-06
tags: ["Java", "S3", "AWS", "Spark", "Airflow"]
categories: Software Engineering
---

In summary, the steps are:
1) Install and configure Airflow on your system
2) Set up a connection to your Kubernetes cluster in Airflow, so that Airflow can access and manage the cluster.
3) Create a new Airflow DAG (Directed Acyclic Graph) to define the workflow for your Spark job. The DAG should include tasks for submitting the Spark job to the Kubernetes cluster, monitoring the job's progress, and handling any errors or failures.
4) In the DAG, specify the details of your Spark job, such as the main class, the job arguments, and the dependencies. You can also use Airflow's built-in operators and hooks to interact with the Kubernetes API and manage the Spark job.
5) Define the schedule for your DAG, using Airflow's schedule interval feature. This will determine how often the Spark job is run.
6) Start the Airflow scheduler to begin running your Spark job according to the defined schedule.
7) Monitor the status of your Spark job using the Airflow web interface, and make any necessary adjustments to the DAG or the cluster settings as needed.

### Can you show an example of such dag?

Sure, here is an example of a DAG that can be used to schedule a Spark job on a Kubernetes cluster:

```python
from airflow import DAG
from airflow.operators.spark_kubernetes_operator import SparkKubernetesOperator

default_args = {
    'start_date': '2022-12-06',
    'retries': 3,
    'retry_delay': 300
}

dag = DAG(
    'spark_kubernetes_example',
    default_args=default_args,
    schedule_interval='0 3 * * *'
)

spark_task = SparkKubernetesOperator(
    task_id='spark_task',
    name='spark-job',
    namespace='default',
    image='image:latest',
    arguments=['spark-submit', '--class', 'com.example.Main', '--deploy-mode', 'cluster', '--executor-memory', '2G', '--total-executor-cores', '2', '--executor-cores', '1', '--driver-memory', '512m', 'local:///opt/spark/examples/jars/spark-examples_2.11-2.4.7.jar'],
    in_cluster=True,
    is_delete_operator_pod=True,
    dag=dag
)
```
In this example, the `SparkKubernetesOperator` is used to submit a Spark job to the Kubernetes cluster. The `arguments` parameter specifies the details of the Spark job, including the main class, the executor and driver memory, and the dependencies. The `schedule_interval` parameter determines how often the job will be run, in this case every day at 3:00 AM. You can adjust these settings as needed for your specific use case.

### Can you show an example of a dockerfile for the image used?

Sure, here is an example of a Dockerfile that can be used to create an image for running a Spark job on a Kubernetes cluster:

```Docker
FROM ubuntu:20.04

# Install Java and other dependencies
RUN apt-get update && apt-get install -y openjdk-8-jdk wget

# Download and extract Spark
RUN wget http://mirrors.sonic.net/apache/spark/spark-2.4.7/spark-2.4.7-bin-hadoop2.7.tgz
RUN tar -xzf spark-2.4.7-bin-hadoop2.7.tgz

# Set environment variables
ENV SPARK_HOME /spark-2.4.7-bin-hadoop2.7
ENV PATH $PATH:$SPARK_HOME/bin

# Copy the application JAR file to the container
COPY app.jar /opt/spark/examples/jars/

# Set the cmd to run the Spark job
CMD ["spark-submit"]
```
This Dockerfile installs the necessary dependencies (Java and Spark), sets the environment variables, and copies the application JAR file to the container. The `CMD` is set to `spark-submit`, so that running the container automatically submits the Spark job. You can modify this Dockerfile as needed to include any additional dependencies or settings required for your specific use case.

### Can you show an example Spark Java application that can be used to produce the jar?

Sure, here is a simple example of a Spark Java application that can be used to produce the JAR file that is referenced in the Dockerfile above:

```java
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

public class Main {

    public static void main(String[] args) {
        // Create the SparkConf and JavaSparkContext
        SparkConf conf = new SparkConf().setAppName("Example Spark App");
        JavaSparkContext sc = new JavaSparkContext(conf);

        // Load the input data
        JavaRDD<String> input = sc.textFile("input.txt");

        // Perform some transformations on the data
        JavaRDD<String> transformed = input.map(s -> s.toUpperCase());

        // Save the output data
        transformed.saveAsTextFile("output.txt");

        // Stop the SparkContext
        sc.stop();
    }
}
```

This application simply reads a text file, performs a simple transformation on the data (converting the text to upper case), and then saves the output to another text file. You can modify this code as needed to perform more complex transformations or to use different input and output sources. Once the code is ready, you can use a tool like Maven to build a JAR file that can be included in the Docker image.

### Can you change it so that it reads and writes to AWS S3?

Sure, here is an example of a Spark Java application that reads from and writes to Amazon S3:

```java
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

public class Main {

    public static void main(String[] args) {
        // Create the SparkConf and JavaSparkContext
        SparkConf conf = new SparkConf().setAppName("Example Spark App");
        JavaSparkContext sc = new JavaSparkContext(conf);

        // Set the S3 access keys
        conf.set("fs.s3a.access.key", "<access-key>");
        conf.set("fs.s3a.secret.key", "<secret-key>");

        // Load the input data from S3
        JavaRDD<String> input = sc.textFile("s3a://<bucket>/input.txt");

        // Perform some transformations on the data
        JavaRDD<String> transformed = input.map(s -> s.toUpperCase());

        // Save the output data to S3
        transformed.saveAsTextFile("s3a://<bucket>/output.txt");

        // Stop the SparkContext
        sc.stop();
    }
}
```

In this example, the `SparkConf` object is used to set the S3 access keys, which are required for accessing the data in S3. The input and output paths are specified using the `s3a://` scheme, which indicates that the data is stored in S3. You will need to replace the placeholder values (`<access-key>`, `<secret-key>`, and `<bucket>`) with your own S3 access keys and bucket name.

Note that this example assumes that the necessary dependencies (e.g. the AWS S3 client library) are already included in the project. You may need to add these dependencies to your `pom.xml` file or equivalent build configuration, in order to build and run the application successfully.


### Can you change Spark mode from `cluster` to `client` and what effect does this have?

Yes, it is possible to change the Spark mode from `cluster` to `client` by using the `--deploy-mode` option when submitting the Spark job. The deploy mode determines how the driver and executors are launched in the cluster, and can have a significant impact on the behavior of the Spark application.

Here is an example of how you can change the deploy mode to `client` when submitting a Spark job:

```python
from airflow import DAG
from airflow.operators.spark_kubernetes_operator import SparkKubernetesOperator

default_args = {
    'start_date': '2022-12-06',
    'retries': 3,
    'retry_delay': 300
}

dag = DAG(
    'spark_kubernetes_example',
    default_args=default_args,
    schedule_interval='0 3 * * *'
)

spark_task = SparkKubernetesOperator(
    task_id='spark_task',
    name='spark-job',
    namespace='default',
    image='image:latest',
    arguments=['spark-submit', '--class', 'com.example.Main', '--deploy-mode', 'client', '--executor-memory', '2G', '--total-executor-cores', '2', '--executor-cores', '1', '--driver-memory', '512m', 'local:///opt/spark/examples/jars/spark-examples_2.11-2.4.7.jar'],
    in_cluster=True,
    is_delete_operator_pod=True,
    dag=dag
)
```

In this example, the arguments parameter is used to specify the `--deploy-mode` option as `client`. This will cause the driver to be launched on the same node as the Airflow task, rather than on a separate node in the cluster.

The main difference between the cluster and client modes is that in the client mode, the driver runs on the client machine (i.e. the Airflow worker), whereas in the cluster mode, the driver runs on a separate node in the cluster. This can affect the performance of the application, as well as the resource utilization on the client machine. You should choose the appropriate deploy mode based on your specific use case and requirements.