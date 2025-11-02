# Deploying and Managing a Containerized React Application as a System Service on RHEL (Using Podman + VMware Port Forwarding)
============================================================================================================================
I deployed a static portfolio site (React + TypeScript) inside a Podman container on a RHEL VM running in VMware Workstation. The goal was to manage it as a system service (systemd unit file) and access it externally.

## Containers as a Service — Practical Session
----------------------------------------------

I containerized my personal portfolio application developed with React and TypeScript, then deployed it on RHEL 9 using Podman integrated with systemd.

````bash
sudo podman pull docker.io/adapaladocker/personal_portfolio_3d:Prod
sudo podman run -d --name mystatapp -p 8080:80 docker.io/adapaladocker/personal_portfolio_3d:Prod
````

Inside the VM, the application worked fine:

````bash
curl http://localhost:8080
````

But while accessing it from my host browser, I got an error:

“This site can’t be reached”

Even after allowing the port in the RHEL firewall:

````bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
````

That’s when I realized that VMware’s NAT network does not forward ports from the host → guest automatically. we need to configure that manually in VMware.

## Option 1: Configure Port Forwarding in VMware Workstation
------------------------------------------------------------

1. **Open VMware Workstation**
   → Menu bar → **Edit → Virtual Network Editor** → Change settings

2. **Select your NAT network**
   In your screenshot it’s **VMnet8 (NAT)**.
   Make sure it’s highlighted.

3. Click **“NAT Settings…”**

![NAT settings edit](./imgs/Virtual_network.png)

![NAT settings edit](./imgs/Virtual_network_editor.png) 

4. In the new window, click **“Add…”**

![NAT settings edit](./imgs/NAT_portforwarding.png) 


5. **Fill in the fields exactly like this:**

   | Field                          | What to Enter  | Explanation                                                                                   |
   | ------------------------------ | -------------- | --------------------------------------------------------------------------------------------- |
   | **Host Port**                  | `8080`         | The port on your Windows host that browsers will use                                          |
   | **Type**                       | `TCP`          | Because HTTP uses TCP                                                                         |
   | **Virtual Machine IP Address** | `192.168.88.x` | The IP of your RHEL VM (run `ip addr show ens160` inside VM to see it, e.g. `192.168.88.128`) |
   | **Virtual Machine Port**       | `8080`         | The port exposed by Podman (`-p 8080:80` maps host 8080 → container 80)                       |
   | **Description**                | `mystatapp`    | Any label you like                                                                            |

6. Click **OK → Apply → OK** to save everything.

7. **Start your RHEL VM** again (the network service reloads when the VM restarts).


---

### Verify Inside the VM

Inside RHEL:

```bash
sudo podman ps
```

You should still see:

```
0.0.0.0:8080->80/tcp
```

---

### Test from Your Host

Open a browser on your Windows machine and visit:

```
http://localhost:8080
```

✅ You should now see your **personal portfolio site** served from the container.

If it doesn’t load:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

and refresh the browser.

![Result](./imgs/application.png) 

---


**Traffic Flow**

```
Browser (Host:8080)
   ↓
VMware NAT rule (mystatapp)
   ↓
VM (192.168.88.x:8080)
   ↓
Podman → Container (80/tcp)
   ↓
Nginx serving your web app
```


Note: your web app is reachable only on your host machine (http://localhost:8080) because VMware’s NAT creates a private network between your host and the VM.

---

**Container as service**


````bash
sudo podman generate systemd --name mystatapp | sudo tee /etc/systemd/system/container-mystatapp.service

sudo systemctl daemon-reload

sudo systemctl status container-mystatapp

sudo systemctl start container-mystatapp

sudo systemctl enable container-mystatapp
````


Test self-healing by killing the container process::

````bash
kill -9 <pid>


[user1@localhost ~]$ sudo kill -9 1607
sudo kill -9 1607

[user1@localhost ~]$ sudo systemctl status container-mystatapp
● container-mystatapp.service - Podman container-mystatapp.service
     Loaded: loaded (/etc/systemd/system/container-mystatapp.service; enabled; preset: disabled)
     Active: active (running) since Sun 2025-11-02 11:17:37 CET; 3s ago
       Docs: man:podman-generate-systemd(1)
    Process: 3253 ExecStart=/usr/bin/podman start mystatapp (code=exited, status=0/SUCCESS)
   Main PID: 3345 (conmon)
      Tasks: 1 (limit: 10718)
     Memory: 2.6M
        CPU: 181ms
     CGroup: /system.slice/container-mystatapp.service
             └─3345 /usr/bin/conmon --api-version 1 -c c522bd7f8909aec02e78dc64bfb9ee9911c00ac08b6a3d88a1766b83cd018a35 -u c522bd7f8909aec02e78dc64bfb9ee9911c00ac08b6a3d88a1766b83cd018a35 -r /usr/bin/crun -b /va>

Nov 02 11:17:37 localhost.localdomain mystatapp[3345]: /docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
Nov 02 11:17:37 localhost.localdomain mystatapp[3345]: /docker-entrypoint.sh: Configuration complete; ready for start up
Nov 02 11:17:37 localhost.localdomain mystatapp[3345]: 2025/11/02 10:17:37 [notice] 1#1: using the "epoll" event method
Nov 02 11:17:37 localhost.localdomain mystatapp[3345]: 2025/11/02 10:17:37 [notice] 1#1: nginx/1.29.1
Nov 02 11:17:37 localhost.localdomain mystatapp[3345]: 2025/11/02 10:17:37 [notice] 1#1: built by gcc 14.2.0 (Alpine 14.2.0) 
Nov 02 11:17:37 localhost.localdomain mystatapp[3345]: 2025/11/02 10:17:37 [notice] 1#1: OS: Linux 5.14.0-570.58.1.el9_6.x86_64
Nov 02 11:17:37 localhost.localdomain mystatapp[3345]: 2025/11/02 10:17:37 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
Nov 02 11:17:37 localhost.localdomain mystatapp[3345]: 2025/11/02 10:17:37 [notice] 1#1: start worker processes
Nov 02 11:17:37 localhost.localdomain mystatapp[3345]: 2025/11/02 10:17:37 [notice] 1#1: start worker process 24
Nov 02 11:17:37 localhost.localdomain mystatapp[3345]: 2025/11/02 10:17:37 [notice] 1#1: start worker process 25

[user1@localhost ~]$ sudo podman ps
CONTAINER ID  IMAGE                                               COMMAND               CREATED            STATUS         PORTS                 NAMES
c522bd7f8909  docker.io/adapaladocker/personal_portfolio_3d:Prod  nginx -g daemon o...  About an hour ago  Up 16 seconds  0.0.0.0:8080->80/tcp  mystatapp


[user1@localhost ~]$ sudo journalctl -u container-mystatapp
````
---

## Option 2: Use Bridged Networking (Easier for Local LAN)

If you frequently need external devices to reach your VM:

1. In VM settings → **Network Adapter**, switch from **NAT** to **Bridged**.
2. Start the VM and check its IP:

   ```bash
   ip addr show ens160
   ```

   e.g. `192.168.1.25`
3. Make sure port 8080 is open inside RHEL:

   ```bash
   sudo firewall-cmd --permanent --add-port=8080/tcp
   sudo firewall-cmd --reload
   ```
4. Access directly from mobile:

   ```
   http://192.168.1.25:8080
   ```

Works on any device in the same Wi-Fi / LAN.

⚠️ **Note:** Bridged mode exposes your VM directly on the network (like another PC).
Use it only in trusted networks.

---


## ✅ Recommended (Safe) Setup for You

| Goal                                    | Best Choice                | Why                               |
| --------------------------------------- | -------------------------- | --------------------------------- |
| Access from same Wi-Fi (mobile, laptop) | **Bridged Network**        | Simplest, direct access via VM IP |
| Access from only your laptop            | **NAT + Port Forwarding**  | Safe, contained                   |        |

---


## What I Learned

Through this lab, I understood how Podman + systemd **simplifies container lifecycle management on single-node systems**, and how VMware networking modes (NAT vs Bridged) impact container accessibility.

- NAT + Port Forwarding: Best for isolated setups — safe and controlled.

- Bridged Networking: Best for LAN access — allows mobile and other devices to connect directly.

Now my containerized application runs persistently as a system service, and I can access it both from the host system and, when bridged, from any device on my Wi-Fi network.
