# Benchmarking Cloud Spanner with PerfKit Benchmarker

This is a hands-on lab/tutorial for generating benchmarks for Google Cloud
Spanner. See the [methodology section](#benchmark-methodology) for best practices and tips for benchmarking Spanner.

## Overview

### Introducing PerfKit Benchmarker

[PerfKit Benchmarker](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker)
is an open source framework with commonly accepted benchmarking tools that you
can use to measure and compare cloud providers. PKB automates setup and teardown
of resources, including Virtual Machines (VMs). Additionally, PKB installs and
runs the benchmark software tests and provides patterns for saving the test
output for future analysis and debugging.

Check out the
[PerfKit Benchmarker README](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/master/README.md)
for a detailed introduction.

### Spanner Tutorial Overview

PKB supports
[__custom configuration__](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/wiki/PerfkitBenchmarker-Configurations)
files in which you can set the machine type, number of machines, parameters for
loading/running, and many other options.

This tutorial uses the provided PKB configuration files in the [data](./data)
folder to run latency and throughput benchmarks for Spanner using the
[YCSB Workload files](https://github.com/brianfrankcooper/YCSB/wiki/Running-a-Workload)
in [perfkitbenchmarker/data/ycsb](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/).

__Note__: This tutorial has been created specifically to run the following YCSB
Workloads:

*   [workloadx](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloadx): Write-Only
*   [workloada](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloada): 50/50 Read/Write
*   [workloadb](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloadb): 95/5 Mostly Read
*   [workloadc](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloadc): Read-only

When you finish this tutorial, you will have the tools needed to create your own
combination of PKB Configurations and YCSB Workloads to run custom benchmarks.

### What you'll do

This lab demonstrates an end-to-end workflow for running benchmark tests and
uploading the result data to Google Cloud.

In this lab, you will:

*   Install PerfKit Benchmarker within a Google Compute Engine (GCE) instance
*   Create a BigQuery dataset for benchmark result data storage
*   Start a benchmark test for Spanner latency
*   Work with the test result data in BigQuery
*   Learn how to run your own benchmarks for throughput and latency

### Prerequisites

*   Basic familiarity with Linux command line
*   Basic familiarity with Google Cloud

## Set up

### What you'll need

To complete this lab, you'll need:

*   Access to a standard internet browser (Chrome browser recommended), where
    you can access the Cloud Console and the Cloud Shell
*   A Google Cloud project

### Sign in to Cloud Console

In your browser, open the [Cloud Console](https://console.cloud.google.com).

Select your project using the project selector dropdown at the top of page.

### Activate the Cloud Spanner API

PerfKit Benchmarker uses the Cloud Spanner and Cloud Spanner Admin APIs to
provision Spanner instances for benchmarking and to delete the instances when
benchmarks are complete.
[Click here](https://console.cloud.google.com/flows/enableapi?apiid=spanner.googleapis.com)
to enable the Spanner APIs for your project.

 > **Note:** You can ignore the prompts for generating credentials, given this tutorial uses GCE to run benchmarks.

### Activate the Cloud Shell

From the Cloud Console click the __Activate Cloud Shell__ icon on the top right
toolbar. You may need to click __Continue__ the first time.

It should only take a few moments to provision and connect to your Cloud Shell
environment.

This Cloud Shell virtual machine is loaded with all the development tools you'll need. It offers a persistent 5GB home directory, and runs on Google Cloud, greatly enhancing network performance and authentication. All of your work in this lab can be done within a browser on your Google Chromebook.

Once connected to the Cloud Shell, you can verify your setup.

1.  Check that you're already authenticated.

    ```
    gcloud auth list
    ```

    **Expected output**

    ```
     Credentialed accounts:
    ACTIVE  ACCOUNT
    *       <myaccount>@<mydomain>.com
    ```

    **Note:** `gcloud` is the powerful and unified command-line tool for Google
    Cloud. Full documentation is available from
    https://cloud.google.com/sdk/gcloud. It comes pre-installed on Cloud Shell.
    Notice `gcloud` supports tab-completion.

1.  Verify your project is known.

    ```
    gcloud config list project
    ```

    **Expected output**

    ```
    [core]
    project = <PROJECT_ID>
    ```

    If it is not, you can set it with this command:

    ```
    gcloud config set project <PROJECT_ID>
    ```

    **Expected output**

    ```
    Updated property [core/project].
    ```

### Task 1. Create a Virtual Machine in Google Compute Engine to execute Benchmarks

From the Cloud Shell, create a new Google Compute Engine (GCE) VM with the necessary scopes.

```sh
gcloud compute instances create pkb-host \
  --scopes="https://www.googleapis.com/auth/compute,https://www.googleapis.com/auth/spanner.admin,https://www.googleapis.com/auth/bigquery" \
  --machine-type=n1-standard-1 \
  --zone=us-west1-a
```

### Task 2. Install PerfKit Benchmarker within the GCE instance

SSH into the GCE VM created in Step 1 and install the required Github Repositories.

1.  SSH into your GCE VM
    ```sh
    gcloud compute ssh pkb-host --zone=us-west1-a
    ```
If you are unable to connect using the `gcloud compute ssh` command, you can use one of the other connection methods described in the [Google Compute Engine Documentation](https://cloud.google.com/compute/docs/instances/connecting-to-instance).

Run all following commands from within the GCE VM.

1.  Set up a virtualenv isolated Python environment within Cloud Shell.

    ```sh
    sudo apt-get install git python3-venv -y
    python3 -m venv $HOME/my_virtualenv
    source $HOME/my_virtualenv/bin/activate
    ```

1.  Ensure Google Cloud SDK tools like bq find the proper Python.

    ```sh
    export CLOUDSDK_PYTHON=$HOME/my_virtualenv/bin/python
    ```

1.  Clone the PerfKitBenchmarker GitHub Repository.

    ```sh
    cd $HOME
    git clone https://github.com/GoogleCloudPlatform/PerfKitBenchmarker.git
    ```

1.  Install PKB dependencies.

    ```sh
    cd $HOME/PerfKitBenchmarker/
    pip install --upgrade pip
    pip install -r requirements.txt
    ```

1.  Clone the Cloud Spanner Ecosystem Benchmarking GitHub Repository.

    ```sh
    cd $HOME && git clone https://github.com/cloudspannerecosystem/spanner-benchmarks-tutorial.git
    ```


### Task 3. Create a BigQuery dataset for benchmark result data storage

By default, PKB logs test output to the terminal and to result files under
`/tmp/perfkitbenchmarker/runs/`.

A recommended practice is to push your result data to
[BigQuery](https://cloud.google.com/bigquery/), a serverless, highly-scalable,
cost-effective data warehouse. You can then use BigQuery to review your test
results over time and create data visualizations.

Using the BigQuery command-line tool `bq`, initialize an empty __dataset__.

```sh
bq mk pkb_results
```

__Output (do not copy)__

```output
Dataset '[PROJECT-ID]:pkb_results' successfully created.
```

You can safely ignore any warnings about the `imp` module.

You can also create datasets using the
[BigQuery UI](https://cloud.google.com/bigquery/) in the Cloud Console. The
dataset can be named anything, but you need to use the dataset name in options
on the command-line when you run tests.

### Task 4. Run a benchmark test

Now that you've installed Perfkit Benchmarker into your GCE VM, you can run benchmark tests from within the VM.

__Note__: This tutorial has been created specifically to run the following YCSB
Workloads:

*   [workloadx](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloadx): Write-Only
*   [workloada](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloada): 50/50 Read/Write
*   [workloadb](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloadb): 95/5 Mostly Read
*   [workloadc](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloadc): Read-only

__Note__: This tutorial splits the benchmarks into two workload categories:

*   [throughput.yaml](./data/throughput_benchmarks): Maximum-throughput workloads
*   [latency.yaml](./data/latency_benchmarks): Latency-sensitive workloads

1.  Grab and review the PKB Configuration file for latency testing.

    ```sh
    cat $HOME/spanner-benchmarks-tutorial/data/latency_benchmarks/workloada.yaml
    ```

    __Output (do not copy)__

    ```output
    # Spanner latency benchmark PKB configuration (workloada)
    benchmarks:
    - cloud_spanner_ycsb:
        flags:
          # Spanner Provisioning
          cloud_spanner_config: regional-us-west1
          cloud_spanner_nodes: 3

          # GCE Provisioning: 5 In-Region VMs per Spanner Node
          ycsb_client_vms: 15
          machine_type: n1-standard-2
          gce_network_name: default
          zones: us-west1-a

          # Data: 100M 1kb rows (100GB total)
          ycsb_record_count: 100000000
          ycsb_field_count: 1
          ycsb_field_length: 1000
          
          # Use 25 threads/VM in Load phase
          ycsb_preload_threads: '25'

          # Sleep 1hr between Load and Run phases
          ycsb_sleep_after_load_in_sec: 3600

          # Execute workloada for 30minutes
          ycsb_workload_files: workloada
          ycsb_timelimit: 1800
          ycsb_operation_count: 100000000 # Opcount high so we hit timelimit
          ycsb_threads_per_client: '25'

          # Target 300 QPS/VM (1,500 QPS / Spanner Node)
          ycsb_run_parameters: target=300,requestdistribution=zipfian,dataintegrity=True

          # Custom YCSB tar to use Spanner Java Client v2.0.1
          ycsb_tar_url: https://storage.googleapis.com/externally_shared_files/ycsb-0.18.0-SNAPSHOT.tar.gz
          ycsb_version: 0.18.0-SNAPSHOT

          # Output the results as a histogram
          ycsb_measurement_type: hdrhistogram
    ```

1.  Grab and review [workloada](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloada), a
    YCSB workload with 50/50 read/write traffic. For more info on YCSB
    workloads and parameters, see
    [YCSB's "Running a Workload" docs](https://github.com/brianfrankcooper/YCSB/wiki/Running-a-Workload).


1.  Run the latency test using `workloada` and store the results in your
    BigQuery dataset.

    > __Note__: Each benchmark can take up to 3 hours to complete when
    > accounting for time to provision, run, and de-provision the required
    > resources.

    ```sh
    cd $HOME/PerfKitBenchmarker
    nohup ./pkb.py \
      --benchmark_config_file=$HOME/spanner-benchmarks-tutorial/data/latency_benchmarks/workloada.yaml \
      --bigquery_table=pkb_results.spanner_benchmarks \
      --file_log_level=info > nohup.out &
    ```

1.  Follow the output of the pkb command using tail:
    ```sh
    tail -f nohup.out
    ```

1.  Monitor the benchmark in action.

    [Monitor](https://cloud.google.com/spanner/docs/monitoring-cloud)
    your Spanner instance in Cloud Console as the benchmark runs.

    Verify you see no ERROR messages in the PKB command's output.

    __Example Output (do not copy)__

    ```output
    INFO     Verbose logging to: /tmp/perfkitbenchmarker/runs/dfff5602/pkb.log
    <...>
    INFO     Running: ssh ...
    <...>
    INFO     Cleaning up benchmark cloud_spanner_ycsb...
    <...>
    ----------------------------------------------------------------------
    Name                 UID                   Status     Failed Substatus
    ----------------------------------------------------------------------
    cloud_spanner_ycsb  cloud_spanner_ycsb   SUCCEEDED
    ----------------------------------------------------------------------
    ```

    If the benchmark fails, verify the provisioned GCE VMs and Spanner instance
    have been removed using the
    [GCE UI](https://console.cloud.google.com/compute/instances) and
    [Spanner UI](https://console.cloud.google.com/spanner/instances). If the
    provisioned resources have not been removed, remove them manually using the
    UI.

1.  Once the benchmark is complete, query `pkb_results.spanner_benchmarks` to
    view the test results in BigQuery.

    In Cloud Shell, run a `bq` command.

    ```sh
    bq query 'SELECT * FROM pkb_results.spanner_benchmarks'
    ```

    You can also see your data using the __Query editor__ in the
    [BigQuery UI](https://console.cloud.google.com/bigquery).

### Task 5. Run more benchmarks

You can use the .yaml files in the [data folder](./data) to run latency or throughput benchmarks on some of the 
common workloads provided by [ycsb](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/)
folder. You can also modify those workloads to run your own custom benchmarks.

> The following workloads represent common Spanner use-cases:
>
> *   [workloadx](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloadx): Write-Only
> *   [workloada](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloada): 50/50
>     Read/Write
> *   [workloadb](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloadb): 95/5 Mostly
>     Read
> *   [workloadc](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/07de8d5b6f59eb477c39e770ec83d32164ea9b0b/perfkitbenchmarker/data/ycsb/workloadc): Read-only

For example, you can run a write-only throughput benchmark by setting the
`--benchmark_config_file` argument to point at the
[throughput_benchmarks/workloadx.yaml](./data/throughput_benchmarks/workloadx.yaml) PKB
Configuration file.

Example: Write-only throughput benchmark command:

```sh
$HOME/PerfKitBenchmarker/pkb.py \
--benchmark_config_file=$HOME/spanner-benchmarks-tutorial/data/data/throughput_benchmarks/workloadx.yaml \
--bigquery_table=pkb_results.spanner_benchmarks
```

### Cleanup

When you are finished running benchmarks, you should delete the `pkb-host` GCE VM. You can delete these with the following command in
Cloud Shell:

```sh
gcloud compute instances delete pkb-host --zone=us-west1-a
```

You may also wish to remove the `pkb_results` dataset in BigQuery. You can
remove this dataset with the following command in Cloud Shell:

```sh
bq rm pkb_results
```

## Congratulations!

You have completed the Spanner PerfKit Benchmarker tutorial!

## Troubleshooting
### Connection timed out
Your test may fail with the following error:
`STDOUT: STDERR: ssh: connect to host xxx.xxx.xxx.xxx port 22: Connection timed out in CreateAndBootVm`
This is a transient error that sometimes occurs during a run. This error occurs at a higher frequency for larger instances / number of client VMs. You can rerun to get rid of this error. Alternatively, you can amend [this line in pkb](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker/blob/46fe3bf4dc71e7357ab75c491c6a0d47037c2b19/perfkitbenchmarker/linux_packages/ycsb.py#L1206) to become  
`vm.RemoteCommand('sudo rm -f ' + remote_path, ignore_failure=True)`
(add the ignore_failure=True flag)


## Benchmark Methodology

### Use-Cases

The provided benchmark configurations cover two separate workload requirement use-cases:
 - **Low Latency**: Benchmarks for latency-sensitive workloads
 - **Throughput**: Benchmarks for throughput-maximizing workloads

### CPU Utilization Targets

#### Latency

In the latency benchmark configurations, the YCSB [`target` runtime parameter](https://github.com/brianfrankcooper/YCSB/wiki/Running-a-Workload#step-4-choose-the-appropriate-runtime-parameters) is used to maintain Spanner High-Priority CPU below 65% Utilization.

#### Throughput

During throughput benchmarks, Spanner High-Priority CPU was maintained above the [published](https://cloud.google.com/spanner/docs/cpu-utilization#recommended-max) 65% High-Priority CPU Utilization guideline.

**CPU Utilization and Zonal Fault Tolerance**

A Regional Cloud Spanner instance is spread across three zones. In case of a zonal failure, Spanner will shift traffic from the downed zone to the remaining two zones. Any application running above the 65% CPU Utilization guidance should be prepared to modify instance provisioning and/or  application behavior to avoid overwhelming the remaining zones.
If your application consistently runs above the published 65% guidance, consider employing an [Autoscaler](https://github.com/cloudspannerecosystem/autoscaler) to dynamically provision the application’s Spanner resources based on CPU load.

### Configuring target QPS
This benchmark allows you to set any number as the target QPS, so what number should you use? This will depend on your actual workload. For example, if you are just curious how far you can push Cloud Spanner, you can set this target QPS to an arbitrarily high number. YCSB will then issue as many QPS as Cloud Spanner can handle. Do note that this will push CPU utilization to 100% and the latency for each request may be well beyond that typical applications can handle. Another way to run this benchmark is to gradually ramp up QPS until latency numbers are beyond what your application can tolerate. For example, starting with X qps per node, and gradually ramp up till Y QPS while P50 < 5ms, P90 < 20ms and P99 latencies at < 200ms.

### Provisioning

These run against [Regional](https://cloud.google.com/spanner/docs/instances#regional_configurations) Cloud Spanner instances (us-west1) using in-region [Google Compute Engine](https://cloud.google.com/compute) virtual machines. All benchmarks provision a three-node Spanner Instance with thirty [n1-standard-1](https://cloud.google.com/compute/docs/machine-types#n1_standard_machine_types) Compute Engine VMs. (10VMs per Spanner Node

### Sleeping between Load and Run

Cloud Spanner periodically rewrites your tables to remove deleted entries and to reorganize your data so that reads and writes are more efficient. In these benchmarks, a 60-minute sleep period is introduced between the YCSB Load and Run steps (using the `ycsb_sleep_after_load_in_sec` argument) to leave time for this process.

## Using Benchmark Tests and Results

It is recommended that these tests are executed with some frequency to account for product evolution and client library version updates.

### Alerting

Alerting thresholds for Cloud Spanner can be set using [Cloud Monitoring](https://cloud.google.com/spanner/docs/monitoring-cloud#create-alert).

Benchmark figures are not suitable for defining alert thresholds on real-world workloads. Alerting thresholds for any workload should be based on the specific requirements of that workload. Actual workloads vary in requirements and behavior. 

### Workload Variability

Performance for any database system (including Cloud Spanner) will vary based on the specifics of a given workload. The performance of real-world workloads may differ significantly from the performance of these simulated benchmark workloads due to differences in data shape, traffic pattern, client configuration, or other nuances.

### Provisioning

Keep a healthy margin when provisioning services based on performance testing of similar workloads.


## Learn More

*   Learn more about
    [best practices](https://cloud.google.com/spanner/docs/best-practice-list)
    for using Google Cloud Spanner.
*   Follow the
    [PKB repo](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker).
