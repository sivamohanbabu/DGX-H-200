This Kubernetes manifest deploys an accelerated PyTorch environment with dual running services (JupyterLab and a lightweight HTTP server) and exposes Jupyter externally via a NodePort Service.

1. Pod Configuration (gdx-app-pod)
Metadata & Selectors:

Name: gdx-app-pod

Label: app: gdx-app (used by the Service to route network traffic to this Pod).

Container Specifications (gdx-container):

Base Image: nvcr.io/nvidia/pytorch:26.10-py3 from NVIDIA NGC, providing a pre-configured CUDA, cuDNN, and PyTorch runtime.

Image Pull Policy: IfNotPresent avoids redownloading the image if it already exists on the worker node.

Startup Execution:

Launches jupyter lab in the background (&) bound to all interfaces on port 8888, with browser opening disabled and root execution permitted.

Concurrently starts Python's built-in HTTP server (python3 -m http.server 8080) in the foreground to serve local directory files or act as an auxiliary API on port 8080.

Container Ports:

jupyter-port (8888/TCP)

api-port (8080/TCP)

Resource Management:

Matches requests and limits exactly (Guaranteed Quality of Service / QoS):

CPU: 4 cores

Memory: 18 GiB

GPU: 1 NVIDIA GPU ([nvidia.com/gpu](https://nvidia.com/gpu): "1"), requiring the NVIDIA Container Toolkit / GPU Operator installed on the cluster.

Security & Capabilities:

Grants SYS_PTRACE, enabling profiling and debugging tools (such as gdb, cuda-gdb, nsys, or py-spy) to inspect processes inside the container.

Shared Memory Mount (/dev/shm):

Mounts a memory-backed emptyDir volume (medium: Memory) with a limit of 18Gi to /dev/shm.

Why it matters: Multi-process data loading in PyTorch (torch.utils.data.DataLoader with num_workers > 0) relies heavily on shared memory. Standard container runtimes default /dev/shm to 64 MiB, which quickly causes bus errors (SIGBUS) during deep learning training without this mount.

2. Service Configuration (gdx-app-service)
Type: NodePort

Exposes the service across each cluster node's IP at a dynamically allocated port (typically within the range 30000–32767).

Selector: Targets pods carrying the app: gdx-app label.

Port Mapping:

Service Port: 8888

Target Port: 8888 (routes to jupyter-port on the container).

Note: Port 8080 (the Python HTTP server) is declared on the Pod but is not exposed by this Service definition.
