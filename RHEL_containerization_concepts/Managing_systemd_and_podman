In this document, You will get to know about 2 main concepts of RHCSA Examination. I am calling it as RHEL containerization concepts.
1. Containers as Services
2. Services inside Container

Containers as Services:
-----------------------

What It Means?
“Container as a Service” means treating containers like system services that can automatically start, stop, and restart just like httpd, sshd, or nginx services on your RHEL system.

Every time the server reboots, you can make your container managed by systemd (Linux service manager). For that, you need enable your container service.

How It Works?

1. check the podman installed in RHEL system
    podman -v

    sudo dnf install podman -y

2. Run a container using the available default images
    mkdir -p /opt/var/lib/mariadb # Volume
    podman run -d --name mariadb -v /opt/var/lib/mariadb/:/var/lib/mysql:Z -e MYSQL_ROOT_PASSWORD=Password1 docker.io/mariadb # -e= environmental variable for the image

    Testing:
        podman ps
        podman exec -it mariadb /bin/bash
        mariadb -u root -p 

    or

    podman run -d --name webserver -p 8080:80 nginx

    Testing:
        podman ps
        curl http://localhost:8080

3. Generate a systemd service file automatically

    step a. Generate in current directory using --files, then move it to /etc/systemd/system/.
    step b. Generate directly into systemd path using redirection (>), followed by daemon-reload.

    step a
        podman generate systemd --name mariadb --files --new
        mv container-mariadb.service /etc/systemd/system/

    step b
        podman generate systemd --name mariadb
        !! > /etc/systemd/system/container-mariadb.service
        or
        podman generate systemd --name mariadb > /etc/systemd/system/container-mariadb.service

        or

        podman generate systemd --name webserver > /etc/systemd/system/container-webserver.service
    
4. Enable and start the container as a service    
    systemctl daemon-reload
    systemctl enable --now container-mariadb

    or

    systemctl daemon-reload
    systemctl enable --now container-webserver

    Testing:
        sudo reboot
        sudo systemctl status container-mariadb

        or

        sudo systemctl status container-webserver

Additional Information:
    If you are a non-root user, than you don't need root privileges to do. you just need create service file below mentioned path

    step 3:
        podman generate systemd --name mariadb > /home/username/.config/systemd/user/container-mariadb.service

    User Sessions and enable-linger
    -------------------------------
    When a non-root user runs a container as a service, systemd normally stops all user services when that user logs out. To make the user’s containers continue running even after logout, use:

        sudo loginctl enable-linger username

        Note: “Enable-linger” keeps the user’s systemd session active in the background.

        sudo systemctl --user <username> status container-mariadb


My Understanding about “Containers as Services”
----------------------------------------------

I am comparing this concept with other containerization technologies to understand it more deeply.

**Docker** is an open-source tool that allows us to build images and run containers.
However, Docker alone cannot manage the complete **container lifecycle** — such as automatically starting containers on boot, restarting them after failure, or managing dependencies between services.

To overcome these limitations, another open-source platform called **Kubernetes** was developed. Kubernetes is a **container orchestration system** that manages container lifecycles at scale, using features like replica sets, scaling, and self-healing.

But in many situations — especially on **single-node systems** or **small environments** — using Kubernetes can be unnecessarily complex.

That’s where **Podman** comes in. Podman integrates directly with **systemd**, the Linux service manager, to handle container lifecycle management in a simple and native way. It doesn’t need any YAML manifest files or external orchestrators.

Using the command `podman generate systemd`, systemd can automatically create a **service configuration file** for the container. This service file ensures that:

* Containers start automatically when the system reboots.
* Containers are automatically restarted if they crash or fail (through systemd’s restart policy).

Because of this tight integration, managing container lifecycle becomes very easy with system services.

This approach is especially useful for **single-server applications**, **standalone microservices**, or **monitoring agents** (for example, running Grafana as a system service).
It provides lifecycle management that is reliable, persistent, and simple — without the complexity of full Kubernetes orchestration.

---
    
Services inside Container
-------------------------

What It Means?

Normally, a container runs one main process — for example, a web server, database, or application.
However, during testing, development, or when dealing with legacy applications, you might want to run multiple background services inside the same container.

For example:
A web server and a database server running together inside a single container.
This approach is called “Services inside a container.”

Why do we need **setsebool -P container_manage_cgroup true**?

When you run systemd-based containers (containers that start multiple services through systemd), you need to allow systemd inside the container to manage its own control groups (cgroups).


**Understanding cgroups (Control Groups)**

cgroups are a Linux kernel feature that allows the system to:

- Limit, monitor, and isolate CPU, memory, and I/O usage per process or container.

Normally, systemd on the host OS controls all cgroups — it manages how resources are distributed among services and processes.


**The Challenge**

When you run multiple services inside a container (for example, httpd, mariadb, atd, etc.), the systemd inside the container also needs to create and manage its own cgroups.

But by default, SELinux on RHEL blocks this behavior for security reasons — to prevent containers from controlling host-level cgroups and affecting other processes on the system.

This causes errors like:
    Failed to mount cgroup at /sys/fs/cgroup/systemd: Operation not permitted


**The Fix**

To allow systemd inside a container to safely manage its own cgroups, run:

    sudo setsebool -P container_manage_cgroup true


Explanation:

    setsebool → modifies an SELinux boolean (a switch for specific permissions)

    container_manage_cgroup → allows containers to manage control groups

    -P → makes the change persistent across reboots

This setting is required for systemd-based containers to function properly.


How It Works?

Changes an SELinux boolean to allow containers to manage host-level cgroups
    sudo setsebool -P container_manage_cgroup true

Create a separate directory for the project
    mkdir test && cd test
        Create a Dockerfile
            FROM docker.io/fedora
            RUN dnf install -y systemd at httpd && dnf clean all
            RUN systemctl enable httpd atd
            EXPOSE 80
            CMD ["/usr/sbin/init"]

Create Image
    podman image build -t web .

Check the Image and run the container
    podman image ls
    podman container run -d --name webby -p 80:80 web

Test the web application access and container details
    curl localhost
    podman container top webby

    sudo podman exec -it webby /bin/bash


My understanding services inside container
------------------------------------------

Legacy applications (monolithic systems) - Containerize old apps without refactoring them (Migration of Legacy workloads)

Older enterprise apps (like ERP, CRM, or custom in-house systems) may expect multiple services to run on the same OS — e.g., application server + database + background daemon.

Containerizing such apps is difficult using microservices.
So, during migration, teams often:

Put the whole legacy stack into one container.

Use systemd inside the container to manage all dependent services.




My Understanding — “Services inside Container”
----------------------------------------------

This concept is especially useful for **legacy or monolithic applications** — where multiple dependent services must run together on the same system.

For example:

* ERP systems
* CRM applications
* Old in-house applications

Such apps often expect an **application server**, **database**, and **scheduler daemon** to all be on one OS.

Containerizing them into **separate microservices** would require major code and configuration changes. So, during **migration**, organizations often:

1. Package the entire legacy stack into a **single container**.
2. Use **systemd inside the container** to manage all dependent services (e.g., start `httpd`, `mariadb`, and `atd`).

This makes it easier to modernize legacy workloads while keeping compatibility.

**Summary**

“Services inside a Container” means running multiple services (like Apache and MariaDB) within a single container using systemd. To make this work safely on RHEL with SELinux enabled, you must allow containers to manage their own cgroups using:
    `sudo setsebool -P container_manage_cgroup true`

This approach is most often used for:

* Legacy workloads (monolithic apps)
* Testing and development environments
* Training or lab setups
---


### Connecting Both Concepts

We can also **combine both concepts** — *“Services inside a Container”* and *“Container as a Service.”*

This means:

1. First, we build a **systemd-based container** that runs **multiple services inside** it (for example, Apache and MariaDB).
2. Then, we use **Podman’s systemd integration** on the host to treat that container itself as a **system service**.

In this way:

* The **container** behaves like a **self-contained mini operating system**, running multiple internal services via its own systemd.
* The **host systemd** manages the **container’s lifecycle** — automatically starting, stopping, and restarting it like any other Linux service.

This hybrid setup combines the advantages of both concepts:

| Concept                       | Function                                                                      |
| ----------------------------- | ----------------------------------------------------------------------------- |
| **Services inside Container** | Runs multiple dependent services using systemd inside the container           |
| **Container as a Service**    | Ensures the container itself is managed by host systemd (auto-start, restart) |

**Use-case examples:**

* Legacy enterprise apps (ERP/CRM systems) that depend on multiple services but need to run persistently on reboot.
* Test environments where you want everything (web + DB + scheduler) inside one container but still managed like a normal service on RHEL.

---
