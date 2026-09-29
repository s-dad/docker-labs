# CLOUD-460: Containers and Orchestration

A hands-on lab portfolio documenting what I learned across 12 weeks of
container and orchestration work  from installing Docker for the first time
to deploying a multi-tier WordPress application on Kubernetes.

## What I Learned

Working through this course gave me practical experience with the full
container lifecycle. I started with the basics of Docker: pulling images,
running containers interactively, building custom images, and pushing to
Docker Hub. From there I moved into private registries, progressively
securing them with basic authentication and then SSL/TLS certificates.

Storage and networking became clearer through hands-on comparison — bind
mounts vs volumes, bridge vs host vs none networks, and how port mapping
controls what is and isn't exposed outside a container.

Docker Compose showed me how to think about multi-container applications
as a single system rather than isolated pieces. Defining a WordPress and
MySQL stack in one YAML file and bringing it up with a single command made
the value of declarative configuration obvious.

Docker Swarm introduced me to clustering and orchestration at a basic level 
initializing a manager, joining workers, and understanding how the swarm
coordinates across nodes.

Kubernetes was the most challenging and most rewarding part of the course.
I worked through pods, deployments, services, scaling, rolling updates,
secrets, persistent volumes, and multi-tier application deployment. Running
WordPress on Kubernetes with persistent storage and secrets management tied
everything together.

## Labs

| Lab | Topic |
|-----|-------|
| [Week 01](./week-01/) | Install Docker Desktop for Windows & Linux |
| [Week 02](./week-02/) | Windows Docker Images |
| [Week 03](./week-03/) | Build Docker Interactive Image & Linux Docker Images |
| [Week 04](./week-04/) | Build a Private Docker Registry & Image Tagging |
| [Week 05](./week-05/) | Docker Private Registry with Authentication & Storage |
| [Week 06](./week-06/) | Docker Private Registry with SSL & Docker Networking |
| [Week 07](./week-07/) | Docker Compose — WordPress and MySQL |
| [Week 08](./week-08/) | Docker Swarm |
| [Week 09](./week-09/) | Kubernetes with Minikube |
| [Week 10](./week-10/) | Kubernetes Networking and Storage |
| [Week 11](./week-11/) | WordPress on Kubernetes |

## Environment
- Windows 11 Home + Docker Desktop
- Ubuntu 20.04 LTS via VirtualBox 7
- Minikube + kubectl