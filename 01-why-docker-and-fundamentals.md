# Docker Fundamentals: Why Docker, What Is Docker, and Understanding Virtualization

# 1. Why Do We Need Docker?

Before understanding Docker, let's understand a common problem in software development and deployment.

Imagine a developer builds a Node.js application on their laptop.

The application works perfectly on their machine.

The developer then shares the application with the DevOps team to deploy it on a server.

But suddenly, the application fails.

Why?

Because the developer's environment and the server environment might be different.

For example:

| Developer's machine | Production server |
|---|---|
| Node.js 20 | Node.js 18 |
| Ubuntu 24.04 | Ubuntu 22.04 |
| Library version 5 | Library version 4 |
| Environment variable configured | Environment variable missing |

The application might depend on a particular version of Node.js, an operating system library, or a specific configuration.

When these dependencies differ, the application may behave differently or fail.

This is commonly known as:

**"It works on my machine!" problem.**

## 1.1 Problems Before Docker

Before containerization became widely adopted, teams commonly used traditional deployment approaches.

Some of the major challenges were:

### 1. Environment inconsistency

Applications worked in development but failed in testing or production because environments had different configurations and dependencies.

### 2. Dependency conflicts

Imagine two applications:

- Application A requires Node.js 18.
- Application B requires Node.js 20.

Installing and maintaining different runtime versions directly on the same server can become complicated.

### 3. Difficult deployment

Developers and operations teams had to manually configure:

- Runtime versions
- System libraries
- Dependencies
- Environment variables
- Application configurations

This made deployment more error-prone.

### 4. Resource consumption

One traditional solution was to create separate virtual machines for different applications.

However, every VM generally requires its own guest operating system, consuming additional CPU, RAM, and storage.

### 5. Scalability and portability

Moving applications between development, testing, staging, and production environments could require repeated configuration work.

Teams needed a more consistent way to package and run applications.

---

# 2. What Is Docker?

**Docker is a platform that allows developers and operations teams to build, package, distribute, and run applications inside containers.**

Docker packages an application along with its required dependencies and configuration into a container image.

That image can then be used to create containers in different supported environments.

### In simple words

Docker helps ensure that an application has a consistent packaged environment when moving between development, testing, and production.

For example:

A developer builds a Node.js application and creates a Docker image containing:

- Application source code
- Node.js runtime
- Required dependencies
- Application configuration and startup instructions

The image can be transferred to another machine and used to run the application without manually installing all those application dependencies on the host.

### Example workflow

```text
Developer writes application
          |
          v
      Dockerfile
          |
          v
    Build Docker Image
          |
          v
     Push to Registry
          |
          v
   Pull Image on Server
          |
          v
    Run Container
          |
          v
    Application Running
```

### Important terminology

| Term | Meaning |
|---|---|
| Dockerfile | Instructions for building an image |
| Docker Image | Packaged, read-only template used to create containers |
| Docker Container | Running or created instance of an image |
| Docker Engine | Platform components responsible for building and running containers |
| Docker Registry | Service used to store and distribute container images |
| Docker Hub | A popular public Docker image registry |

---

# 3. Before Understanding Containers: What Is Virtualization?

To understand why containers are lightweight, we first need to understand virtualization.

**Virtualization is the process of creating virtual versions of computing resources, such as servers, operating systems, storage, and networks.**

Instead of requiring one physical server for every application, virtualization allows multiple virtual machines to run on the same physical hardware.

### Example

Imagine you have one physical server:

- 16 CPU cores
- 64 GB RAM
- 1 TB storage

You want to run three different applications.

Without virtualization, you might need three separate physical servers.

With virtualization, you can create three virtual machines on the same physical server.

```text
          Physical Server
        16 CPU | 64 GB RAM
                 |
             Hypervisor
                 |
       ---------------------
       |         |         |
      VM 1      VM 2      VM 3
       |         |         |
     Linux    Windows    Linux
       |         |         |
      App A     App B     App C
```

Each VM behaves like an independent computer.

It has virtualized hardware resources and its own guest operating system.

---

# 4. What Is a Hypervisor?

**A hypervisor is software or firmware that creates and manages virtual machines by allocating and controlling access to physical hardware resources.**

In simple words:

A hypervisor allows multiple virtual machines to run on a single physical server.

It manages resources such as:

- CPU
- Memory
- Storage
- Network interfaces

### Example

Suppose you have one physical server running VMware ESXi.

You create three VMs:

- VM 1: Ubuntu
- VM 2: Windows Server
- VM 3: Ubuntu

The hypervisor manages the virtual hardware presented to each VM and allocates the underlying physical resources.

### Types of Hypervisors

There are two major types of hypervisors.

## 4.1 Type 1 Hypervisor (Bare-Metal)

A Type 1 hypervisor runs directly on the physical hardware.

It does not require a general-purpose host operating system underneath it.

Architecture:

```text
       Physical Hardware
               |
       Type 1 Hypervisor
               |
      -------------------
      |        |        |
     VM 1     VM 2     VM 3
      |        |        |
     OS       OS       OS
      |        |        |
    App A    App B    App C
```

Examples:

- VMware ESXi
- Microsoft Hyper-V (when deployed as a Type 1 hypervisor)
- Xen

### Characteristics

- Runs directly on physical hardware.
- Manages multiple virtual machines.
- Commonly used in enterprise data centers.
- Provides hardware-level virtualization.

## 4.2 Type 2 Hypervisor (Hosted)

A Type 2 hypervisor runs as an application on an existing host operating system.

Architecture:

```text
       Physical Hardware
               |
         Host OS
               |
      Type 2 Hypervisor
               |
      -------------------
      |        |        |
     VM 1     VM 2     VM 3
      |        |        |
     OS       OS       OS
      |        |        |
    App A    App B    App C
```

Examples:

- Oracle VirtualBox
- VMware Workstation
- VMware Fusion

### Characteristics

- Runs on top of an existing OS.
- Uses the host OS to access hardware resources.
- Commonly used for development, testing, and learning.

---

# 5. What Is a Virtual Machine (VM)?

**A virtual machine is a software-based computer that runs its own guest operating system using virtualized hardware resources.**

A VM behaves like an independent computer.

It has:

- Virtual CPU
- Virtual RAM
- Virtual disk
- Virtual network interface
- Guest operating system
- Its own operating system kernel

### Example

Suppose you have a Windows laptop.

You install VirtualBox and create an Ubuntu VM.

Your architecture looks like this:

```text
          Physical Laptop
                 |
             Windows OS
                 |
          VirtualBox
                 |
          Ubuntu VM
                 |
            Linux OS
                 |
           Linux Kernel
                 |
          Ubuntu Application
```

Even though your physical machine runs Windows, you can run Ubuntu inside a VM.

The Ubuntu VM has its own Linux kernel.

### Important

A virtual machine has its own guest operating system and kernel.

This is one of the biggest differences between traditional VMs and Linux containers.

---

# 6. VM vs Hypervisor vs Container

These three terms are related, but they do not mean the same thing.

| Term | What it is | Main responsibility |
|---|---|---|
| Hypervisor | Software or firmware | Creates and manages VMs |
| Virtual Machine | Virtual computer | Runs its own guest OS |
| Container | Isolated process environment | Runs applications using a shared host kernel |

### Understand the relationship

- Physical hardware provides the computing resources.
- Hypervisor manages virtual machines.
- Virtual machines run guest operating systems.
- Containers use operating system features to isolate application processes.

### Architecture comparison

**Traditional virtualization:**

```text
        Physical Hardware
                |
            Hypervisor
                |
       ---------------------
       |         |         |
      VM 1      VM 2      VM 3
       |         |         |
    Guest OS  Guest OS  Guest OS
       |         |         |
     Kernel    Kernel    Kernel
       |         |         |
      App       App       App
```

**Containerization:**

```text
        Physical Hardware
                |
             Host OS
                |
          Host Kernel
                |
          Docker Engine
                |
       ---------------------
       |         |         |
   Container A Container B Container C
       |         |         |
    Node.js     Java      Nginx
```

Containers share the host kernel instead of running separate guest operating systems.

---

# 7. Why Are Containers Lightweight?

Now we can understand why containers are generally considered lightweight.

Imagine you want to run three applications.

### Approach 1: Using Virtual Machines

You create three virtual machines.

Each VM requires:

- Its own guest OS
- Its own kernel
- Background operating system services
- Application dependencies
- Application processes

This creates additional resource overhead.

### Approach 2: Using Containers

You run three containers on one Linux host.

Each container has:

- Application
- Required dependencies
- Isolated execution environment

But all containers share the host's Linux kernel.

They do not need to boot a separate guest operating system.

### Comparison

| Feature | VM | Container |
|---|---|---|
| Guest OS | Required | Not required |
| Separate kernel | Yes | No |
| Resource overhead | Generally higher | Generally lower |
| Startup time | Generally slower | Generally faster |
| Isolation | Hardware virtualization | OS-level isolation |
| Portability | VM image | Container image |
| Best suited for | Full OS isolation and mixed OS workloads | Lightweight application deployment |

**Important:** Containers still consume resources. Docker Engine, the runtime, networking, storage, and isolation mechanisms also have overhead. Lightweight does not mean zero overhead.

---

# 8. How Does Docker Solve the Original Problem?

Let's return to our original Node.js example.

The developer has:

- Node.js application
- Node.js runtime
- Required npm packages
- Application configuration

Instead of asking the DevOps team to manually recreate the environment, the developer provides a Dockerfile.

Example:

```dockerfile
FROM node:20

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

EXPOSE 3000

CMD ["node", "index.js"]
```

Build the image:

```bash
docker build -t my-node-app:1.0 .
```

Run the container:

```bash
docker run -d -p 3000:3000 my-node-app:1.0
```

Now the DevOps team can use the same image in supported environments.

The team does not need to manually install Node.js and the npm dependencies on every deployment server.

### What Docker actually improves

1. **Consistency:** The same image carries the application runtime and dependencies.
2. **Portability:** Images can be distributed between machines with compatible container support.
3. **Deployment:** Applications can be started from images using a repeatable process.
4. **Isolation:** Containers isolate application processes and resource views.
5. **Efficiency:** Multiple containers can share the host kernel.

### Important clarification

Docker does not automatically solve every environment-related problem.

You still need to manage:

- Environment-specific variables
- Secrets
- External services
- Persistent storage
- Network configuration
- CPU and memory requirements
- Security and permissions

The container image makes the application environment more consistent, but external dependencies and infrastructure still need proper configuration.

---

# 9. Where Does Docker Fit in DevOps?

Docker plays an important role in modern CI/CD pipelines.

A typical workflow looks like this:

```text
       Developer
           |
           v
      Git Repository
           |
           v
       CI Pipeline
           |
           v
      Build Application
           |
           v
      Build Docker Image
           |
           v
       Run Tests
           |
           v
     Push Image to Registry
           |
           v
       CD Pipeline
           |
           v
    Deploy Container
           |
           v
   Development / Testing /
       Production
```

### Example DevOps tools

| Stage | Example tools |
|---|---|
| Source control | Git, Azure Repos, GitHub |
| CI/CD | Azure DevOps, Jenkins, GitHub Actions |
| Build | Maven, npm, dotnet |
| Containerization | Docker |
| Image registry | Docker Hub, Azure Container Registry, Amazon ECR, Nexus Repository |
| Orchestration | Kubernetes |
| Monitoring | Prometheus, Grafana |

Docker is not a replacement for build tools like Maven, npm, or dotnet.

They solve different problems.

For example:

- Maven builds a Java application.
- Docker packages the application and its runtime environment into an image.
- Kubernetes can orchestrate containers across multiple machines.

---

# 10. Introduction to Docker Architecture

Now that we understand why Docker exists, let's briefly introduce its architecture.

Docker uses multiple components to build and run containers.

The major components include:

1. Docker CLI
2. Docker Daemon (`dockerd`)
3. Docker Engine API
4. containerd
5. runc
6. Linux kernel

A simplified architecture:

```text
       User / Developer
               |
               v
          Docker CLI
               |
               v
       Docker Engine API
               |
               v
        Docker Daemon
            dockerd
               |
               v
           containerd
               |
               v
             runc
               |
               v
         Linux Kernel
               |
               v
          Container
```

We will explore these components and their responsibilities in the next chapter.

---

# 11. Final Revision

| Concept | Definition |
|---|---|
| Docker | Platform for building, distributing, and running containers |
| Virtualization | Creating virtual versions of computing resources |
| Hypervisor | Software or firmware that creates and manages VMs |
| Virtual Machine | Software-based computer with its own guest OS |
| Type 1 Hypervisor | Runs directly on physical hardware |
| Type 2 Hypervisor | Runs on top of an existing host OS |
| Container | Isolated environment for running processes |
| Container Image | Read-only template used to create containers |
| Containerization | OS-level isolation and packaging of applications |
| Overhead | Additional resources required to operate something |

## 12. Interview Questions

1. Why was Docker introduced?
2. What problem does Docker solve?
3. What is Docker?
4. What is virtualization?
5. What is a hypervisor?
6. What is the difference between Type 1 and Type 2 hypervisors?
7. What is a virtual machine?
8. What is the difference between a VM and a container?
9. Why are containers lightweight?
10. Do containers have their own kernels?
11. Can Docker eliminate all environment-related issues?
12. What is the difference between Docker and Kubernetes?
13. Is Docker a replacement for Maven, npm, or dotnet?
14. How does Docker fit into a CI/CD pipeline?

---

## Final Takeaway

Docker addresses the challenges of application packaging, deployment consistency, portability, and isolation.

Traditional virtual machines use a hypervisor to virtualize hardware and run separate guest operating systems.

Containers use operating system-level isolation to run applications while sharing the host kernel.

This makes containers generally more lightweight and faster to start than traditional VMs.

**The learning journey:**

Why Docker → Virtualization → Hypervisor → Virtual Machines → Containers → Docker Architecture → Linux Namespaces → cgroups → containerd → runc → Docker Workflow.
