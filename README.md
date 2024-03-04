# HPC Engineers Toolkit

## Data Management in HPC: How to archive large data sets?

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

## GitHub Actions/Runners

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


### SLURM Power Saving

The files in this repository will give you examples of what is need to get SLURM's Power Saving feature working.

Wake-On-Lan Steps:
1. Configure the Hardware to do wake-on-lan. This may be done in Bios or a vendor specific command. Be sure to configure the network adapter you will use for sending (headnode) and receiving (compute node?) wake-on-lan packets.
2. Get the wake-on-lan utility on your head node. If you are on Centos 7, install net-tools. On SUSE flavors, it's netdiag. This gets you ether-wake, which is how we will wake the nodes when they are needed.
3. Create a file, /etc/ethers, which has all of your MAC addresses and what node they tie to. If you have multiple networks, you only need the mac addresses for the network you are going to use to carry wake-on-lan packets.

IPMI Steps:
1. Setup your IPMI/BMC hardware for access on the network you need the traffic to run over.

Final Steps:
1. Find a place to put your suspend, node_shutdown, resume, and node_startup scripts. Normally they are in /opt/system/slurm/etc. Please note how we use eth-tool to ensure the g bit is set on the network adapter before shutdown, so that wake-on-lan works. 
2. Ensure those .sh files are executable.
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

#### node_suspend.sh 

```
#!/bin/bash
# Example SuspendProgram
echo "`date` Suspend invoked $0 $*" >> /var/log/power_save.log
hosts=`scontrol show hostnames $1`
for host in $hosts
do
   echo "Suspending " $host >> /var/log/power_save.log
   sudo /opt/system/slurm/etc/node_shutdown.sh $host &>> /var/log/power_save.log
done
```

#### node_shutdown.sh

```
#!/bin/bash
# Let's first ensure it'll respond to the magic packet since it is more secure
# and how the ether-wake command works
ssh $1 "ethtool -s enp2s0 wol g"
# Then power down through shutdown...
ssh $1 "shutdown -h now"
# or IPMI (but note you'd need to retreive the IPMI address, not the interconnect)...
#ipmitool -H $1 -v -I lanplus -U username -P userpassword chassis power off
```

#### node_startup.sh

```
#!/bin/bash
# If using Wake-On-Lan on RHEL or variants:
ether-wake $1
# If using Wake-On-Lan on SUSE
#
# If using IPMI:
#ipmitool -u $username -p $password -I lanplus -h $1 chassis power up
```

#### node_resume.sh

```
#!/bin/bash
# Example ResumeProgram
echo "`date` Resume invoked $0 $*" >> /var/log/power_save.log
hosts=`scontrol show hostnames $1`
for host in $hosts
do
   echo "Starting up " $host >> /var/log/power_save.log
   sudo /opt/system/slurm/etc/node_startup.sh $host &>> /var/log/power_save.log
done
```


## Powe Management in HPC Clusters
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



## OPEN RESERACH PROBLEM 


Toward Building a Digital Twin of Job Scheduling and Power Management on an HPC System by Ohmura et al. 2023

The purpose of this work is to reduce the burden on system administrators by virtually reproducing job scheduling and power management of their target systems and thereby helping them properly configure the system parameters and policies.
