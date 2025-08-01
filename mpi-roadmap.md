# MPI Roadmap: A Strategy for MPI Ranking with OpenMPI and Slurm

**Author(s):** Kelvin Woo
**Date:** 2025-07-01

## **1. Title and Context**

- **Title**: MPI Roadmap: A Strategy for MPI Ranking with OpenMPI and Slurm
- **Author(s)**: [Your Name/Team]
- **Date**: {current_date}
- **Context / Purpose**: This document outlines a strategy for standardizing how Message Passing Interface (MPI) processes are ranked and mapped to resources when using OpenMPI within a Slurm-managed HPC environment. The goal is to ensure performance, predictability, and ease of use for all users running MPI applications.
- **Background**: For readers new to this area, MPI is a standard for parallel programming on distributed memory systems. OpenMPI is a popular open-source implementation of MPI. Slurm is a widely used workload manager for Linux clusters. The interaction between these components, specifically how MPI processes (ranks) are assigned to nodes and cores, is critical for application performance.

## **2. Executive Summary**

*A concise summary of the entire document. This section should be written last.*

This document proposes a clear path forward for our MPI strategy. We analyze the current challenges and opportunities related to process ranking and placement. After evaluating several options—ranging from relying purely on Slurm's integration to using specific OpenMPI configurations—we recommend a hybrid approach that balances performance with user simplicity. The proposed solution is expected to improve application performance by an estimated 15-20% for communication-bound workloads and reduce user support tickets related to MPI configuration by 30%.

## **3. Problem Statement / Opportunity**

Currently, there is no single, documented standard for running OpenMPI jobs under Slurm in our environment. This leads to several problems:

- **Inconsistent Performance**: Users are left to discover their own optimal settings, leading to highly variable and often suboptimal application performance.
- **Configuration Complexity**: The number of options available through both Slurm (`srun`) and OpenMPI (`mpirun`) for process placement is confusing for many users.
- **Lack of Reproducibility**: Different users running the same code on the same resources may get different performance results due to underlying configuration differences.
- **Support Overhead**: A significant portion of support time is dedicated to troubleshooting MPI performance and configuration issues.

### **Cloud-Specific Challenges**

Unlike traditional HPC environments with stable, predictable hardware configurations, our cloud environment introduces additional complexity:

- **Dynamic Resource Allocation**: Compute and network resources are provisioned dynamically, requiring topology-aware MPI configuration that adapts to the current hardware layout.
- **Rapid Technology Evolution**: OpenMPI, EFA (Elastic Fabric Adapter), and related technologies evolve quickly, necessitating a structured release process to leverage the latest performance improvements and bug fixes.
- **Scale-Dependent Issues**: Application errors such as memory leaks and performance degradation often only manifest at scale, typically months after feature delivery, requiring comprehensive testing across all deployment scales.

The opportunity is to define a "golden path"—a recommended and well-supported configuration—that delivers excellent performance for the majority of use cases out-of-the-box while addressing the unique challenges of cloud-based MPI deployments. This will simplify the user experience, improve resource utilization, and establish a baseline for performance tuning.

## **4. Goals and Tenets**

### **Goals**
- **Performance**: Achieve near-optimal performance for common MPI communication patterns.
- **Simplicity**: Provide a straightforward, easy-to-use configuration for users.
- **Consistency**: Ensure that MPI jobs run predictably across the cluster.
- **Flexibility**: Allow expert users to override defaults for specialized use cases.
- **Topology Awareness**: Automatically adapt MPI configuration to the current hardware topology for optimal resource utilization.
- **Technology Currency**: Maintain up-to-date versions of MPI components to leverage latest performance improvements and security patches.
- **Scale Testing**: Ensure comprehensive testing across all deployment scales to catch issues early in the development cycle.

### **Tenets**
*(As per our discussion, we will incorporate our broader engineering tenets later.)*

## **5. Detailed Analysis and Options**

Here we explore the primary options for integrating OpenMPI with Slurm for process ranking and placement, with special consideration for cloud environments.

### **Cloud Environment Considerations**

Before diving into the specific options, it's important to understand how cloud environments differ from traditional HPC:

- **Dynamic Topology**: Unlike static HPC clusters, cloud resources are provisioned dynamically, requiring MPI to adapt to the current hardware layout.
- **Network Variability**: Cloud networks can have varying latency and bandwidth characteristics, necessitating topology-aware process placement.
- **Resource Elasticity**: The ability to scale up and down requires MPI configurations that can handle varying numbers of nodes and cores.
- **Technology Velocity**: Cloud providers and MPI implementations evolve rapidly, requiring a structured approach to version management.

### **Option 1: Native Slurm Integration (`srun`)**

- **Description**: Rely entirely on Slurm's Process Management Interface (PMIx) to launch and manage OpenMPI processes. Users would use `srun` directly to launch their MPI executables.
- **Pros**:
    - Simplest user experience (`srun ./my_mpi_app`).
    - Tight integration with Slurm for accounting and resource management.
    - No need to generate hostfiles or manually specify process counts.
- **Cons**:
    - Historically, PMIx versions shipped with operating systems can have bugs or performance limitations compared to OpenMPI's native launcher.
    - Less control over fine-grained OpenMPI tuning parameters.
- **Risks**: We may be dependent on the system's PMIx version, which could be slow to update.

### **Option 2: OpenMPI's `mpirun` with Slurm Awareness**

- **Description**: Use `mpirun` as the primary launcher. OpenMPI has built-in support for detecting it is running inside a Slurm allocation.
- **Pros**:
    - Full access to all OpenMPI features and performance tuning capabilities.
    - Uses OpenMPI's highly-optimized process launch mechanism (ORTE/PRTE).
- **Cons**:
    - Can be slightly more complex for users if they need to pass `mpirun` options.
    - Potential for conflict if Slurm and OpenMPI settings are not aligned.
- **Risks**: Misconfiguration can lead to processes being placed incorrectly, negating performance gains.

### **Option 3: Hybrid `mpirun` with Slurm-generated Hostlists**

- **Description**: A more manual approach where a script first queries Slurm for the allocated nodes (`scontrol show hostnames`) to generate a hostfile, and then passes this to `mpirun`.
- **Pros**:
    - Explicit and transparent control over where processes are placed.
- **Cons**:
    - Most complex for the user; requires scripting.
    - Prone to errors in script logic.
    - Bypasses some of the useful integration features.
- **Risks**: This approach is brittle and not scalable from a user support perspective.

### **Option 4: Cloud-Optimized Topology-Aware MPI**

- **Description**: An enhanced version of Option 2 that incorporates cloud-specific optimizations including topology awareness, EFA integration, and dynamic resource adaptation.
- **Pros**:
    - Automatically adapts to current hardware topology for optimal performance.
    - Leverages latest cloud networking technologies (EFA, etc.).
    - Provides consistent performance across varying cloud configurations.
    - Integrates with cluster orchestration for resource optimization.
- **Cons**:
    - Requires more complex initial setup and testing.
    - May need ongoing maintenance as cloud technologies evolve.
- **Risks**: Dependency on cloud-specific features may limit portability to other environments.

## **6. Recommendation**

We recommend **Option 2: OpenMPI's `mpirun` with Slurm Awareness** as our primary, documented strategy, enhanced with cloud-specific optimizations.

- **Why**: This option provides the best balance of performance, by leveraging OpenMPI's native capabilities, and simplicity, as OpenMPI can automatically detect the Slurm environment. We can provide a simple wrapper or module file that sets sensible defaults, giving 90% of users a great experience while allowing the remaining 10% of expert users the flexibility they need.

### **Enhanced Implementation Strategy**

#### **1. Topology-Aware MPI Configuration**
- **Cluster Orchestration Integration**: Develop a system that optimizes compute and network resources during cluster orchestration, then generates topology-aware MPI commands.
- **Dynamic Hostfile Generation**: Create automated scripts that query the current cluster topology and generate optimized hostfiles for MPI process placement.
- **EFA Integration**: Ensure proper configuration of Elastic Fabric Adapter for optimal network performance in cloud environments.

#### **2. Technology Release Process**
- **Version Management**: Establish a structured process for evaluating and deploying new versions of OpenMPI, EFA drivers, and related components.
- **Testing Pipeline**: Implement automated testing for new MPI component versions across different hardware configurations.
- **Rollback Strategy**: Maintain the ability to quickly rollback to previous stable versions if issues are discovered.

#### **3. Comprehensive Scale Testing**
- **Multi-Scale Validation**: Implement testing protocols that validate application behavior across all deployment scales (small, medium, large, and maximum).
- **Memory and Performance Monitoring**: Deploy comprehensive monitoring and logging systems to detect memory leaks, performance degradation, and other issues at scale.
- **Core Dump Analysis**: Establish automated core dump collection and analysis to quickly identify root causes of failures.

### **Implementation Plan**:
1. **Q3**: Develop topology-aware MPI configuration system and enhanced environment modules.
2. **Q3**: Establish technology release process and automated testing pipeline.
3. **Q4**: Implement comprehensive scale testing framework with monitoring and logging.
4. **Q4**: Host user training sessions for the new cloud-optimized approach.
5. **Q1 Next Year**: Deprecate older MPI modules and guides, establish ongoing monitoring and optimization processes.

## **7. Appendices**

- [Link to performance benchmark results for each option]
- [Link to sample job scripts]
- [Link to OpenMPI and Slurm documentation]
- [Link to topology-aware MPI configuration examples]
- [Link to EFA configuration best practices]
- [Link to scale testing framework documentation]
- [Link to technology release process guidelines] 