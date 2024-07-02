# Docker Networking Basics

Docker provides several networking options to manage how containers communicate with each other, the host system, and the outside world. The most common Docker networking modes are:

## 1) Bridge Network (default):
- When you run a container without specifying a network, Docker creates a virtual bridge (a docker0 interface) on the host and connects the container to it.
- Containers on the same bridge network can communicate with each other using their IP addresses.
- Port mapping (e.g., -p 80:80) is used to expose container ports to the host.


![docker000](https://github.com/MdShafiqulSaymon/Portfolio/assets/68004638/fa0a622d-6ef8-4167-8da2-da8d50c93f51)


### Let's Dive in Real Example
**I'm using EC2 instance which I show my main branch README.md file**

```
   docker run -d -p 80:80 --name nginx-container nginx
```
This command for 
```
    -d runs the container in detached mode.
    -p 80:80 maps port 80 of the EC2 instance to port 80 of the Nginx container.
    --name nginx-container names the container nginx-container.
    nginx specifies the Nginx image.
```
Output Like:

![image](https://github.com/MdShafiqulSaymon/Portfolio/assets/68004638/a41ecd18-b6cb-4e27-91a2-5c5418a3b80e)

**Let's See the Container IP assign by docker0(default Bridge)**
use command to show details of an container

```
   docker inspect <container_id>
   docker inspect 87cf06605ee7
```

![DD](https://github.com/MdShafiqulSaymon/Portfolio/assets/68004638/8e34911a-1df0-45ee-a9c7-c17e16e246d4)

**Hit The URL of EC2 Instance IP(13.231.217.86)    http://13.231.217.86**

![image](https://github.com/MdShafiqulSaymon/Portfolio/assets/68004638/2ba77e35-cfe2-471b-8862-a9a6b6150a7d)




## 2) Host Network:
- In the host network mode, the container shares the network namespace with the host. This means the container uses the host's IP address and ports directly.
- There is no need for port mapping; the container can use any port on the host directly.

![HOST](https://github.com/MdShafiqulSaymon/Portfolio/assets/68004638/410a4364-5b2b-4760-9ee2-988b27e3abfa)

In host network mode, containers use the host’s network directly. There’s no need to map ports because the container shares the host’s network stack. However, this mode can cause port conflicts with the host.

```
    docker run --network host nginx
```
This makes Nginx available on port 80 of the host directly.


## 3) Custom User-Defined Bridge Networks:
A custom user-defined bridge network is a bridge network that you create explicitly, allowing for better control over container networking. Containers connected to the same custom bridge network can communicate with each other using their names, simplifying network management and enhancing security.

**Benefits of Custom Bridge Networks**
- Improved Container Name Resolution:
  - Containers on the same custom network can resolve each other by name. Docker sets up an internal DNS service to handle this.
- Network Isolation:
  - Containers on a custom network are isolated from containers on other networks, enhancing security.
- Better Control and Flexibility:
  - You can configure custom networks with specific options, such as subnet ranges, IP address management, and more.
- Simplified Inter-Container Communication:
  - Using container names instead of IP addresses makes configuration easier and more readable.
- Scoped Network Policies:
  - You can define rules and access controls specific to the network.



![custom](https://github.com/MdShafiqulSaymon/Portfolio/assets/68004638/dfdd8af3-142f-43e5-a7d5-6ce8c2103cc7)




