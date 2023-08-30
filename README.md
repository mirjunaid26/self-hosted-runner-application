### HPC Engineers Toolkit

# Problem 2
Given the complexity of archiving large microscope datasets and your specific requirements, here's a suggested strategy and an outline of a Bash script to prepare and archive the data to tape. This approach focuses on creating an organized, versioned, and efficient archiving process:

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

Remember to customize the script by replacing `/path/to/soma_fs` and `/path/to/tape/mount` with your actual source and destination paths. Also, implement the checksum verification mechanism based on your needs.

This script is a basic outline and should be thoroughly tested in a controlled environment before using it for critical archiving operations. Additionally, consider incorporating error handling, logging, and other necessary security measures based on your organization's policies and requirements.


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

Make sure to thoroughly test your workflow and archiving script to ensure they work as expected. Also, consider setting up additional steps, such as sending notifications on completion or handling potential failures.


# Problem 3

Creating a complete and effective Bash script for configuring an HPC cluster's power-saving measures using SLURM involves multiple complex steps that require thorough understanding of both SLURM and the cluster's hardware. However, I can provide you with a simplified example of a Bash script that demonstrates the concept of adjusting SLURM settings for power savings during low utilization periods.

Please note that this script is a starting point and may need to be customized according to your cluster's setup, SLURM configuration, and specific power-saving strategies.

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

It's important to note that real-world implementation requires a more comprehensive approach, possibly involving database integration, real-time monitoring, and more sophisticated power-saving strategies. Additionally, these scripts need to be developed and tested carefully in a controlled environment before applying them to a production cluster.

# SOLUTIONs

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

It's important to note that the specific tools used will depend on the hardware and software environment of your HPC cluster. Additionally, tools may need to be integrated and customized to fit the cluster's unique characteristics and requirements.


Custom scripts for adaptive power management in HPC clusters can be designed to automate various tasks that optimize power consumption based on the cluster's utilization and workload patterns. Here are some examples of custom scripts you could develop:

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

When creating custom scripts, ensure that they are thoroughly tested in a controlled environment before deploying them to a production cluster. Also, consider implementing error handling, logging, and security measures to ensure the scripts function reliably and securely. Custom scripts should be periodically reviewed and updated to adapt to changing cluster requirements and technologies.


# PAID SOLUTIONS

WORKLOAD-AWARE POWER MANAGEMENT
Moab® HPC Suite’s workload awareness enables it to provide a unique and innovative solution to power management. It is estimated that over the life of an HPC system, accrued energy costs are equivalent to the cost of the hardware itself. With HPC systems expanding rapidly, energy control is increasingly necessary to reducing costs, meeting power targets, and minimizing carbon footprint. Through idle system power reclamation and per-application power optimization, Moab offers the tools necessary for organizations to identify, create, and implement the optimal power savings solutions to meet their power management objectives.

IDLE SYSTEM POWER RECLAMATION
Through Moab’s Intelligent Power Management, HPC users can lower the power state of idle nodes in order to reclaim unnecessary energy usage. HPC systems inevitably experience some measure of node idleness, for example, at the start and end of the life of a cluster, during evenings, weekends, and holidays, or any other time when job submissions slow down or halt. Moab® identifies nodes that are not currently executing workloads and takes them offline by lowering their power state to either standby, suspend, hibernate, or off. This will result in significant reductions in excess power usage, especially with large systems.

To maintain fast response times, Moab® utilizes a Green Pool Buffer Policy which helps to mitigate the delay inherent in restarting these offline nodes. The Green Pool is a small number of nodes permitted to remain in an idle state, thereby ensuring that there are always online nodes readily available for new jobs. The Green Pool then powers down or powers up nodes to maintain quick access to idle nodes, but leverages Moab’s predictive power to ensure that the system is always prepared to perform optimally, while still saving energy appropriately.

Eliminate excess power consumption
Decrease carbon footprint
Maintain system responsiveness

# OPEN RESERACH PROBLEM 


Toward Building a Digital Twin of Job Scheduling and Power Management on an HPC System by Ohmura et al. 2023

Abstract: The purpose of this work is to reduce the burden on system administrators by virtually reproducing job scheduling and power management of their target systems and thereby helping them properly configure the system parameters and policies. Specifically, this paper focuses on a real computing system, named Supercomputer AOBA, as an example to discuss the importance of accurately reproducing the behaviors of job scheduling in the simulation. Since AOBA uses some special power saving features that are not supported by any existing job scheduling simulators, we have first implemented a component for a job scheduling simulator to support the special features, and thus to build a“Digital Twin" of AOBA’s job scheduler. By using the Digital Twin with actual operation data, a system administrator can check if the system is efficiently used in terms of computational performance and power efficiency. This paper shows a use case of exploring appropriate scheduling and power saving parameters. In the use case, we found that there are more appropriate parameter configurations, which can reduce the job waiting time by 70% at most and the energy consumption by 1.2% at most when the system is busy. By exploiting such a Digital Twin, therefore, it is demonstrated the feasibility that a system administrator can properly adjust various parameters without disturbing the system operation.