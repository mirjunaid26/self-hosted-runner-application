# HPC Engineers Toolkit

## Problem 2

**Strategy: Efficient and Versioned Archiving**

1. **Organize and Structure Data:**
   Maintain a well-structured directory hierarchy for the data on your "soma_fs" file system. Each level corresponds to a specific category (wafer, sections, FOVs) and aids in easy retrieval and updates.

2. **Incremental Updates:**
   Instead of rearchiving the entire dataset, implement a strategy that allows you to perform incremental updates. This ensures that only new or modified data are transferred and archived.

3. **Versioning:**
   Implement a versioning mechanism that keeps track of changes to the dataset. This way, you can access previous versions when needed.

4. **Data Integrity and Verification:**
   Implement checksums or hashing mechanisms to ensure data integrity during transfer and archival.

5. **Efficient Data Transfer:**
   Utilize a protocol that takes advantage of the 10G internet connection for efficient data transfer. Consider using protocols like rsync or SCP.

6. **Archive Metadata:**
   Maintain metadata that describes the archived datasets, including timestamps, version information, and descriptions.

**Bash Script: Data Preparation and Archival**

```bash
#!/bin/bash

# Paths and variables
SOURCE_DIR="/path/to/soma_fs"        # Source directory
TAPE_MOUNT="/path/to/tape/mount"    # Mount point of the tape backup
ARCHIVE_NAME="data_archive_$(date +'%Y%m%d').tar.gz"  # Archive name
LOG_FILE="archive_log_$(date +'%Y%m%d').log"          # Log file name

# Log function
log() {
    echo "$(date +'%Y-%m-%d %H:%M:%S') $1" >> "$LOG_FILE"
}

# Step 1: Prepare data for archiving
log "Archiving data..."
tar -czf "$ARCHIVE_NAME" -C "$SOURCE_DIR" .

# Step 2: Copy archived data to tape
log "Copying data to tape..."
cp "$ARCHIVE_NAME" "$TAPE_MOUNT"

# Step 3: Verify data integrity
log "Verifying data integrity..."
# Implement checksum verification here

# Step 4: Clean up
log "Cleaning up..."
rm "$ARCHIVE_NAME"

log "Archival process completed."
```

Replace `/path/to/soma_fs` and `/path/to/tape/mount` with your actual source and destination paths.

Consider incorporating error handling, logging, and other necessary security measures based on your organization's policies and requirements.


To automate the archiving process using GitHub Actions, you can set up a workflow that triggers on certain events, such as pushing to a specific branch, and then uses a self-hosted runner to execute the archiving script. Here's a high-level overview of the process:

1. **Set Up Self-Hosted Runner:**

   - Install and configure a self-hosted runner on a machine in your environment. This runner will be used to execute the archiving script.

2. **Create the Archiving Script:**

   - Create a version of the archiving script that you've outlined earlier, making sure it works correctly in your environment.

3. **Create a GitHub Actions Workflow:**

   - In your repository, create a new YAML file (e.g., `.github/workflows/archive.yml`) to define the GitHub Actions workflow.

   - Define the workflow to trigger on specific events, such as pushes to a certain branch, manual triggers, or on a schedule.

   - Specify the jobs and steps that need to be executed in the workflow. In this case, you'll need a job that runs the archiving script using the self-hosted runner.

4. **Configure the Workflow:**

   - Specify the runner you want to use for this workflow. You can do this by assigning a label to the runner during its configuration, and then referencing that label in the workflow YAML.

   - Set up any environment variables or secrets that the archiving script might need.

5. **Run the Workflow:**

   - Commit and push your changes to trigger the workflow based on the defined conditions.

   - GitHub Actions will automatically execute the workflow, which will run the archiving script on the self-hosted runner.

Here's an example YAML for the GitHub Actions workflow:

```yaml
name: Archive Data

on:
  push:
    branches:
      - main  # Change this to the desired branch

jobs:
  archive:
    runs-on: self-hosted  # Replace with the label of your self-hosted runner

    steps:
    - name: Checkout Repository
      uses: actions/checkout@v2

    - name: Run Archiving Script
      run: |
        chmod +x archiving_script.sh
        ./archiving_script.sh  # Replace with the actual script name

    - name: Upload Log to Artifacts
      uses: actions/upload-artifact@v2
      with:
        name: archive-log
        path: archive_log_*.log  # Replace with the actual log file name
```

In this example, replace placeholders like `archiving_script.sh`, `archive_log_*.log`, and others with actual values corresponding to your setup.


## Problem 3

### SLURM Power Saving Tools

# SLURM Power Saving
## Example Scripts

The files in this repository will give you examples of what is need to get SLURM's Power Saving feature working.

The number one piece of advice that I received was to increase the timeouts to 10 minutes and wait. Everything in here is on a timer and you need to wait 10 minutes or whatever for those timers to timeout. Wait. You will not get this done in a day. Wait.

Also, I owe a huge thank you to Brian Haymore at the University of Utah for his help in getting all of this working.

Examples of the below steps are given under the SERVER_ROOT directory.

Wake-On-Lan Steps:
1. Configure the Hardware to do wake-on-lan. This may be done in Bios or a vendor specific command. Be sure to configure the network adapter you will use for sending (headnode?) and receiving (compute nodes?) wake-on-lan packets.
2. Get the wake-on-lan utility on your head node. If you are on Centos 7 like me, install net-tools. On SUSE flavors, it's netdiag. This gets you ether-wake, which is how we will wake the nodes when they are needed.
3. Create a file, /etc/ethers, which has all of your MAC addresses and what node they tie to. If you have multiple networks, you only need the mac addresses for the network you are going to use to carry wake-on-lan packets.

IPMI Steps:
1. Setup your IPMI/BMC hardware for access on the network you need the traffic to run over.

Final Steps:
1. Find a place to put your suspend, node_shutdown, resume, and node_startup scripts. Mine are in /opt/system/slurm/etc. Please note how I am using eth-tool to ensure the g bit is set on the network adapter before shutdown, so that wake-on-lan works. 
2. Ensure those .sh files are executable. Github seems to mess this up.
3. Edit slurm.conf. Be sure to change your SuspendProgram and ResumeProgram locations to where you put your scripts.
4. Run "scontrol reconfigure" to make the changes permanent.
5. Wait. Remember. Wait.
6. See if your idle nodes become idle~ in sinfo.
7. Run a job on those to bring them back.
8. See if they return to idle~ when they are idle once more.

It may be useful to remember the sinfo codes:  
\*  The node is presently not responding and will not be allocated any new work. If the node remains non-responsive, it will be placed in the DOWN state (except in the case of COMPLETING, DRAINED, DRAINING, FAIL, FAILING nodes).  
\~  The node is presently in powered off.  
\#  The node is presently being powered up or configured.  
\!  The node is pending power down.  
\%  The node is presently being powered down.  
\$  The node is currently in a reservation with a flag value of "maintenance".  
\@  The node is pending reboot.  
\^  The node reboot was issued.  
\-  The node is planned by the backfill scheduler for a higher priority job.  

#### slurm.conf script
```
#!/bin/bash
# PowerSaving Segment
# How long shall we wait for a node to be idle before shutting it down?
SuspendTime=300 # 5 minutes
# How many nodes will your utility let you stop at once?
SuspendRate=40
# How many nodes will your utility let you start at once?
ResumeRate=40
# Where is the script to shut the nodes down?
SuspendProgram=/opt/system/slurm/etc/suspend.sh
# Where is the script to start the nodes up?
ResumeProgram=/opt/system/slurm/etc/resume.sh
# Time how long does it take for your slowest node to shutdown? Add 60 to that and put the answer below.
SuspendTimeout=300
# Time how long it takes for your slowest node to start up? Add 60 to that and put the answer below.
ResumeTimeout=240
# If you want to exclude certain nodes or partitions, enable the below and fill them in.
#SuspendExcNodes=compute-3
#SuspendExcParts=main
# How long to wait between a node being up, and when a job runs. This can take awhile, FYI.
BatchStartTimeout=360 # Default is 10.
# Time for messages to go from the management node and a compute node AND back. This can take awhile.
MessageTimeout=100 # Default is 10.

# I kept the partition and node information below the above.
...

# At the very bottom I changed/added the below. The first is to prevent the nodes from being 'down' when they suspend.
# Which requires some scontrol commands to return them to service.
SlurmctldParameters=enable_configless,idle_on_node_suspend
# This one returns a node to service when it registers itself as up.
ReturnToService=2
```





#### slurm.conf file
```
#!/bin/bash
# Excerpt of slurm.conf
SelectType=select/cons_res
SelectTypeParameters=CR_CORE_Memory

SuspendProgram=/usr/sbin/slurm_suspend
ResumeProgram=/usr/sbin/slurm_suspend
SuspendTime=600
SuspendExcNodes=tux[0-127]
TreeWidth=128

NodeName=DEFAULT    Sockets=1 CoresPerSocket=4 ThreadsPerCore=2
NodeName=tux[0-127] Weight=1 Feature=local State=UNKNOWN
NodeName=ec[0-127]  Weight=8 Feature=cloud State=CLOUD
PartitionName=debug MaxTime=1:00:00 Nodes=tux[0-32] Default=yes
PartitionName=batch MaxTime=8:00:00 Nodes=tux[0-127],ec[0-127] Default=no
```

### SuspendProgram and ResumeProgram
```
#!/bin/bash
# Example SuspendProgram
echo "`date` Suspend invoked $0 $*" >>/var/log/power_save.log
hosts=`scontrol show hostnames $1`
for host in $hosts
do
   sudo node_shutdown $host
done

#!/bin/bash
# Example ResumeProgram
echo "`date` Resume invoked $0 $*" >>/var/log/power_save.log
hosts=`scontrol show hostnames $1`
for host in $hosts
do
   sudo node_startup $host
done
```
Bash script that demonstrates the concept of adjusting SLURM settings for power savings during low utilization periods. This is a starting point and may need to be customized according to your cluster's setup, SLURM configuration, and specific power-saving strategies.

```bash
#!/bin/bash

# Set SLURM configurations for power savings during low utilization

# Idle node detection and standby mode
IDLE_THRESHOLD=10  # Example: Set a utilization threshold for idle nodes (percentage)
STANDBY_POWER_STATE="standby"  # Example: Power-saving state when nodes are idle

# Energy-efficient queue settings
EFFICIENT_QUEUE="energy_efficient"  # Example: Queue name for energy-efficient jobs
EFFICIENT_PRIORITY=100  # Example: Priority for the energy-efficient queue

# Dynamic rescheduling during moderate utilization
CONSOLIDATION_SCRIPT="/path/to/consolidation_script.sh"  # Example: Path to the consolidation script

# Predictive scheduling settings
PREDICTIVE_SCRIPT="/path/to/predictive_script.sh"  # Example: Path to the predictive script

# Set SLURM configurations
scontrol update NodeName=ALL State=${STANDBY_POWER_STATE}
scontrol create PartitionName=$EFFICIENT_QUEUE Priority=$EFFICIENT_PRIORITY

# Run dynamic rescheduling script during moderate utilization
if [ -f "$CONSOLIDATION_SCRIPT" ]; then
    $CONSOLIDATION_SCRIPT
fi

# Run predictive scheduling script
if [ -f "$PREDICTIVE_SCRIPT" ]; then
    $PREDICTIVE_SCRIPT
fi

echo "SLURM power-saving configurations updated."
```

Please replace placeholders (`/path/to/...`) with actual paths to your scripts and adjust the settings based on your cluster's configuration. The provided script demonstrates some basic concepts, including updating node states, creating an energy-efficient queue, and running custom scripts for dynamic rescheduling and predictive scheduling.


## SOLUTIONs

Adaptive power management in HPC clusters involves using various tools and techniques to optimize power consumption while maintaining performance. Here are some tools commonly used for adaptive power management in HPC clusters:

1. **SLURM (Simple Linux Utility for Resource Management):**
   SLURM is a widely used job scheduler and resource manager in HPC clusters. It offers features that can be leveraged for power management, such as managing power states of nodes, energy-efficient job scheduling, and prioritizing energy-efficient queues during low utilization periods.

2. **PowerTOP:**
   PowerTOP is a Linux utility that provides real-time power usage information for processes and devices. It helps identify power-hungry applications and processes, allowing administrators to make informed decisions about resource allocation and power management.

3. **Intel Node Manager (NM) and Dynamic Power Node Manager (DPNM):**
   Intel NM and DPNM are tools that allow administrators to monitor and control the power consumption of Intel-based servers. They provide features for managing power caps, monitoring thermal conditions, and optimizing power efficiency.

4. **IBM PowerAI:**
   IBM PowerAI offers a suite of tools and technologies for optimizing AI and HPC workloads on IBM Power Systems. It includes power management capabilities that dynamically adjust power usage based on workload requirements.

5. **HPC Power Management Libraries:**
   Some HPC vendors offer power management libraries and APIs that enable fine-grained control over power usage, allowing administrators to adjust frequencies, voltages, and power states of hardware components.

6. **Ganglia:**
   Ganglia is a scalable distributed monitoring system that provides real-time views of cluster performance and resource usage. It can help identify underutilized nodes and aid in making decisions for power management.

7. **Advanced Configuration and Power Interface (ACPI):**
   ACPI is a standard for managing power and configuration of hardware devices. It's built into most modern systems and can be used to control power states and optimize energy consumption.

8. **Adaptive Voltage and Frequency Scaling (AVFS) Tools:**
   Some processors support AVFS, which dynamically adjusts voltage and frequency levels based on workload demands. Tools exist to control AVFS and take advantage of energy-efficient scaling.

9. **Energy Management Tools from Hardware Vendors:**
   Many hardware vendors provide their own energy management tools designed to work with their specific hardware, allowing administrators to configure and optimize power usage.

10. **Custom Scripts and Automation:**
    Administrators often create custom scripts and automation to monitor resource usage, job scheduling, and power consumption, and then make adjustments based on predefined policies.




### OTHER SOLUTIONS

1. **Dynamic Node Scaling Script:**
   - Monitor cluster utilization using SLURM job queue data or monitoring tools.
   - Automatically adjust the number of active nodes based on the current workload.
   - Use SLURM commands to scale down nodes during low utilization periods and scale up during high utilization.

2. **Energy-Efficient Job Scheduling Script:**
   - Analyze job queue data to identify energy-efficient job scheduling opportunities.
   - Prioritize or queue low-priority jobs during off-peak hours to consolidate tasks and reduce the number of active nodes.

3. **Predictive Scheduling Script:**
   - Analyze historical job scheduling patterns to predict upcoming low-usage periods.
   - Automatically adjust the power modes of nodes or schedule jobs accordingly.

4. **Thermal Management Script:**
   - Monitor the temperature of nodes using hardware sensors.
   - Dynamically adjust node power states to prevent overheating while minimizing performance impact.

5. **Resource Consolidation Script:**
   - Identify nodes with low utilization and consolidate active jobs onto fewer nodes.
   - Power down underutilized nodes to save energy.

6. **Job Checkpointing and Suspension Script:**
   - Automatically checkpoint long-running jobs and suspend them during low-usage periods.
   - Resume jobs when higher utilization is expected.

7. **Adaptive Frequency Scaling Script:**
   - Monitor CPU load and workload intensity.
   - Dynamically adjust CPU frequencies to match workload demands, optimizing performance-per-watt.

8. **Energy Reporting and Logging Script:**
   - Collect and log energy consumption data for individual nodes or the entire cluster.
   - Generate reports and analytics to track energy-saving improvements over time.

9. **User Notification Script:**
   - Notify users about scheduled power-saving periods and provide options for scheduling energy-efficient jobs.

10. **Hybrid Energy and Performance Optimization Script:**
    - Develop a script that optimizes a trade-off between performance and energy savings based on user-defined policies.

11. **Cluster Cooling Optimization Script:**
    - Integrate with cooling management systems to adjust cooling resources based on workload and temperature conditions.

12. **Emergency Power Management Script:**
    - Detect power grid stress or supply issues and automatically initiate power-saving measures to prevent overload or system failures.



## PAID SOLUTIONS

WORKLOAD-AWARE POWER MANAGEMENT
Moab® HPC Suite’s workload awareness enables it to provide a unique and innovative solution to power management. It is estimated that over the life of an HPC system, accrued energy costs are equivalent to the cost of the hardware itself. With HPC systems expanding rapidly, energy control is increasingly necessary to reducing costs, meeting power targets, and minimizing carbon footprint. Through idle system power reclamation and per-application power optimization, Moab offers the tools necessary for organizations to identify, create, and implement the optimal power savings solutions to meet their power management objectives.

IDLE SYSTEM POWER RECLAMATION
Through Moab’s Intelligent Power Management, HPC users can lower the power state of idle nodes in order to reclaim unnecessary energy usage. HPC systems inevitably experience some measure of node idleness, for example, at the start and end of the life of a cluster, during evenings, weekends, and holidays, or any other time when job submissions slow down or halt. Moab® identifies nodes that are not currently executing workloads and takes them offline by lowering their power state to either standby, suspend, hibernate, or off. This will result in significant reductions in excess power usage, especially with large systems.

To maintain fast response times, Moab® utilizes a Green Pool Buffer Policy which helps to mitigate the delay inherent in restarting these offline nodes. The Green Pool is a small number of nodes permitted to remain in an idle state, thereby ensuring that there are always online nodes readily available for new jobs. The Green Pool then powers down or powers up nodes to maintain quick access to idle nodes, but leverages Moab’s predictive power to ensure that the system is always prepared to perform optimally, while still saving energy appropriately.

Eliminate excess power consumption
Decrease carbon footprint
Maintain system responsiveness

## OPEN RESERACH PROBLEM 


Toward Building a Digital Twin of Job Scheduling and Power Management on an HPC System by Ohmura et al. 2023

Abstract: The purpose of this work is to reduce the burden on system administrators by virtually reproducing job scheduling and power management of their target systems and thereby helping them properly configure the system parameters and policies. Specifically, this paper focuses on a real computing system, named Supercomputer AOBA, as an example to discuss the importance of accurately reproducing the behaviors of job scheduling in the simulation. Since AOBA uses some special power saving features that are not supported by any existing job scheduling simulators, we have first implemented a component for a job scheduling simulator to support the special features, and thus to build a“Digital Twin" of AOBA’s job scheduler. By using the Digital Twin with actual operation data, a system administrator can check if the system is efficiently used in terms of computational performance and power efficiency. This paper shows a use case of exploring appropriate scheduling and power saving parameters. In the use case, we found that there are more appropriate parameter configurations, which can reduce the job waiting time by 70% at most and the energy consumption by 1.2% at most when the system is busy. By exploiting such a Digital Twin, therefore, it is demonstrated the feasibility that a system administrator can properly adjust various parameters without disturbing the system operation.