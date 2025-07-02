# ansible-patching-labAnsible demo lab setup using Docker, perfect for demonstrating Linux patching automation. This setup will include:
Ansible Controller: A Docker container running the Ansible engine.
Target Nodes: Multiple Docker containers simulating Linux servers, which will be patched.
Installation Steps: Detailed instructions to get this lab up and running.
Demonstration Playbook: A basic Ansible playbook to illustrate patching.

Ansible Demo Lab: Linux Patching Automation


1. Lab Architecture

Host Machine: Your local machine where Docker is installed.
Ansible Controller Container: A Docker container (ansible-controller) that will run Ansible commands and connect to the target nodes. It will have SSH client installed.
Target Node Containers: Multiple Docker containers (e.g., node1, node2) simulating Linux servers (e.g., Ubuntu, CentOS) that will be managed by Ansible and patched. They will have SSH server installed.

2. Prerequisites

Docker: Ensure Docker Desktop (Windows/macOS) or Docker Engine (Linux) is installed and running on your host machine.
Install Docker Desktop
Install Docker Engine on Linux
Basic understanding of Docker and Ansible.

3. Lab Setup Steps


Step 3.1: Create a Project Directory

Create a dedicated directory for your lab:

Bash


mkdir ansible-patching-lab
cd ansible-patching-lab



Step 3.2: Create Dockerfiles

We'll create two Dockerfiles: one for the Ansible controller and one for the target nodes.
3.2.1: Dockerfile.ansible-controller
This Dockerfile builds an image with Ansible installed and necessary tools.

Dockerfile
# Dockerfile.ansible-controller
FROM ubuntu:latest

# Install SSH client ,Ansible and Vim 
RUN apt-get update && \
    apt-get install -y openssh-client ansible python3-apt vim sshpass && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /ansible

# (Optional) Create an SSH key pair for the controller
# For a demo, we'll use password-based authentication or pre-shared keys.
# If you want to use SSH keys, you'd generate them here and copy to the host machine,
# or create them on the host and bind-mount them.
# For simplicity in this demo, we'll rely on Docker's networking and
# either password authentication or pre-shared keys for SSH.

CMD ["/bin/bash"]


3.2.2: Dockerfile.target-node
This Dockerfile builds a basic Ubuntu image with SSH server enabled. You can adapt this for CentOS/RedHat if needed.

Dockerfile
# Dockerfile.target-node
FROM ubuntu:latest

# Install SSH server and allow password authentication
RUN apt-get update && \
    apt-get install -y openssh-server sudo vim && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    mkdir -p /run/sshd && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Set root password (for demo purposes only - use SSH keys in production!)
RUN echo 'root:password' | chpasswd

# Expose SSH port
EXPOSE 22

# Start SSH service
CMD ["/usr/sbin/sshd", "-D"]




Note on Security: Using PermitRootLogin yes and a default password (root:password) is highly insecure and for demonstration purposes only. In a real-world scenario, you would use SSH keys, disable password authentication for root, and create dedicated unprivileged users with sudo access.

Step 3.3: Build Docker Images

Navigate to your ansible-patching-lab directory in your terminal and build the images:

Bash


docker build -t ansible-controller -f Dockerfile.ansible-controller .
docker build -t target-node -f Dockerfile.target-node .



Step 3.4: Create Docker Network

It's good practice to create a dedicated Docker network for your lab containers.

Bash


docker network create ansible-net



Step 3.5: Run Containers

Now, run the Ansible controller and target node containers on the ansible-net.

Bash


# Run Ansible Controller
docker run -dit --name ansible-controller --network ansible-net ansible-controller

# Run Target Node 1
docker run -dit --name node1 --network ansible-net target-node

# Run Target Node 2
docker run -dit --name node2 --network ansible-net target-node


You can verify that the containers are running:

Bash


docker ps



Step 3.6: Configure Ansible Inventory

We need to create an Ansible inventory file within the ansible-controller container.
3.6.1: Access the Ansible Controller Container

Bash


docker exec -it ansible-controller bash


3.6.2: Create inventory.ini
Inside the ansible-controller container, create the inventory.ini file:

Bash


vi /ansible/inventory.ini


Add the following content:

Ini, TOML


[targets]
node1 ansible_host=node1 ansible_user=root ansible_password=password
node2 ansible_host=node2 ansible_user=root ansible_password=password


Save and exit the vi editor (press Esc, then :wq, then Enter).
3.6.3: (Optional) Test Connectivity
Still inside the ansible-controller container, you can test connectivity:

Bash


ansible all -m ping -i /ansible/inventory.ini --extra-vars "ansible_python_interpreter=/usr/bin/python3"


You should see output similar to this, indicating successful ping to both nodes:



node1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
node2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}


Exit the controller container:

Bash


exit



4. Linux Patching Automation Demonstration

Now that the lab is set up, let's create an Ansible playbook to perform patching.

Step 4.1: Create a Playbook

On your host machine (outside the Docker containers), create a file named patch_linux.yml in your ansible-patching-lab directory:

Bash


vi patch_linux.yml


Add the following content:

YAML


---
- name: Apply Linux Patches
  hosts: targets
  become: true # Use sudo for root privileges on target nodes
  vars:
    apt_update_cache_valid_time: 3600 # Cache valid for 1 hour

  tasks:
    - name: Update apt cache (Ubuntu/Debian)
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: "{{ apt_update_cache_valid_time }}"
      when: ansible_os_family == "Debian"

    - name: Upgrade all installed packages (Ubuntu/Debian)
      ansible.builtin.apt:
        upgrade: dist
      when: ansible_os_family == "Debian"

    - name: Autoremove unnecessary packages (Ubuntu/Debian)
      ansible.builtin.apt:
        autoremove: yes
      when: ansible_os_family == "Debian"

    # Add similar tasks for CentOS/RHEL if your target-node Dockerfile was for those OSes
    # - name: Update yum cache (CentOS/RHEL)
    #   ansible.builtin.yum:
    #     update_cache: yes
    #   when: ansible_os_family == "RedHat"

    # - name: Upgrade all installed packages (CentOS/RHEL)
    #   ansible.builtin.yum:
    #     name: '*'
    #     state: latest
    #   when: ansible_os_family == "RedHat"

    - name: Reboot if necessary
      ansible.builtin.reboot:
        reboot_timeout: 600 # Wait up to 10 minutes for reboot
      when: ansible_reboot_required|default(false)
      register: reboot_status
      ignore_errors: true # Continue even if reboot fails (e.g., in a rapid demo)

    - name: Wait for systems to come back online after reboot
      ansible.builtin.wait_for_connection:
        timeout: 600
      when: reboot_status.changed # Only wait if a reboot was actually triggered


Save and exit the vi editor.

Step 4.2: Copy Playbook to Ansible Controller

We need to copy this playbook into the ansible-controller container.

Bash


docker cp patch_linux.yml ansible-controller:/ansible/



Step 4.3: Run the Patching Playbook

Access the Ansible controller container again:

Bash


docker exec -it ansible-controller bash


Now, run the playbook:

Bash


ansible-playbook /ansible/patch_linux.yml -i /ansible/inventory.ini --extra-vars "ansible_python_interpreter=/usr/bin/python3"


Expected Output:
You will see Ansible connecting to node1 and node2, updating their package lists, upgrading packages, and potentially removing unnecessary ones. If there are kernel updates or other changes requiring a reboot, Ansible will attempt to reboot the containers and wait for them to come back online.
Since these are fresh Ubuntu containers, there might not be many immediate updates, but the tasks will execute. To simulate a scenario where updates are available, you could build your target-node image with an older FROM image or by intentionally installing outdated packages.

5. Cleaning Up the Lab

Once you're done with the demonstration, you can clean up the Docker containers and network:

Bash


# Stop and remove containers
docker stop ansible-controller node1 node2
docker rm ansible-controller node1 node2

# Remove Docker network
docker network rm ansible-net

# (Optional) Remove Docker images
docker rmi ansible-controller target-node



Conclusion

This Docker-based Ansible lab provides a quick and isolated environment to experiment with Linux patching automation. You can extend this lab by:
Adding more target nodes.
Creating different types of target nodes (e.g., CentOS/RHEL, Alpine).
Implementing more complex patching strategies (e.g., staggered rollouts, pre/post-patching scripts, change management integration).
Using SSH key-based authentication instead of passwords for better security.
Integrating with a source control system (Git) to manage your playbooks.

