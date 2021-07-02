# Apache Httpd Webserver configuration on Docker using Ansible(by updating inventory with container’s IP)

![apache_httpd_ansible_docker](https://miro.medium.com/max/1400/1*uM7U5yjWA8J0ZDI8Kk4Jcw.jpeg)

## Objective
Configuration of **Apache Httpd Webserver** on Docker container using Ansible by dynamic addition of container’s IP address to the inventory file.
<br><br>

## Content
- **Project Understanding : Ansible Roles Setup**
- **Output**
<br><br>

## Project Understanding : Ansible Roles Setup
Let’s understand each Ansible role implementation one by one

### Part 1 : Ansible Role (container_creation)

|| **tasks : main.yml** ||

```yaml
---
# tasks file for container_creation

- name: Mount Directory Creation
  file:
    state: directory
    path: "{{ mount_directory }}"
    
- name: Mount Directory to CDROM
  mount:
    src: "/dev/cdrom"
    path: "{{ mount_directory }}"
    state: mounted
    fstype: "iso9660"
    
- name: Yum Repository - AppStream
  yum_repository:
    baseurl: "{{ mount_directory }}/AppStream"
    name: "dvd1"
    description: "Yum Repository - AppStream"
    enabled: true
    gpgcheck: no
    
- name: Yum Repository - BaseOS
  yum_repository:
    baseurl: "{{ mount_directory }}/BaseOS"
    name: "dvd2"
    description: "Yum Repository - BaseOS"
    enabled: true
    gpgcheck: no
    
- name: Docker Repository Setup
  yum_repository:
    baseurl: https://download.docker.com/linux/centos/7/x86_64/stable/
    name: "docker"
    description: "Docker Repository"
    enabled: true
    gpgcheck: no
    
- name: Check whether Docker is installed or not
  command: "rpm -q docker-ce"
  register: docker_check
  
- name: Docker Installation
  command: "yum install docker-ce --nobest -y"
  when: '"is not installed" in docker_check.stdout'
  
- name: Starting Docker Service
  service:
    name: "docker"
    state: started
    enabled: yes
    
- name: Python3 Installation
  package:
    name: "python36"
    state: present
    
- name: Installation of Docker SDK for Python
  pip:
    name: "docker-py"
    
- name: Changing SELinux state to permissive
  selinux:
    policy: targeted
    state: permissive
    
- name: Stopping Firewall
  service:
    name: "firewalld"
    state: stopped
    
- name: DockerHub Login
  docker_login:
    username: "{{ dockerhub_username }}"
    password: "{{ dockerhub_password }}"
    
- name: Build Directory Creation
  file:
    path: /root/dockerfile
    state: directory
    
- name: Copying Dockerfile to the Build Directory
  copy:
    src: files/Dockerfile
    dest: /root/dockerfile/Dockerfile
    
- name: Building Image from Dockerfile and pushing it to DockerHub
  community.docker.docker_image:
    build:
      path: /root/dockerfile
    name: "{{ dockerhub_username }}/centos_ssh"
    tag: v1
    push: yes
    source: build
    
- name: Creation of container from the image created
  docker_container:
    name: centos_ssh_container
    image: "{{ dockerhub_username }}/centos_ssh:v1"
    pull: yes
    detach: yes
    state: started
    ports:
    - "{{ port_number }}:80"
  register: container_info
  
- name: Retrieving Container IP
  shell: echo '{{container_info["ansible_facts"]["docker_container"]["NetworkSettings"]["IPAddress"] }}'
  register: container_ip
  
- name: Adding Container IP to the inventory file
  template:
    src: templates/inventory.yml
    dest: /etc/inventory.yml
    
- name: Removing EDCSA key from known_hosts file
  lineinfile:
    path: /root/.ssh/known_hosts
    state: absent
    regexp: "ecdsa-sha2-nistp256"
```

#### Important Points

- _Mount directory is created along with repository for Yum and Docker._
- _Docker is installed and respective service are enabled._
- _Python 3.6 is installed, also Python SDK for Docker is installed as it is required for usage of Docker modules in Ansible._
- _**SELinux** state is changed to permissive to avoid hindrance in Docker container creation using Ansible modules. Also Firewall is disabled so web page to be set up in the next part could be accessed from outside as well._
- _Docker image are built using **Dockerfile**(mentioned below) and then pushed to DockerHub. Container from the same is created with SSH connectivity set up within it, also port 80 used for webserver is exposed using the concept of **PAT(Port Address Translation)**._
- _Container’s IP address is retrieved and dynamically placed in the inventory file. Also **EDCSA** key is removed from **known_hosts** file present in Docker Host to avoid issues in accessing the new container generated via SSH for Ansible._

**Note:** _In order to use **“community.docker.docker_image”** module, run the command mentioned below:_

```shell
ansible-galaxy collection install community.docker
```

<br>
-------

|| **files : Dockerfile** ||

```Dockerfile
FROM centos:latest
RUN yum install openssh-server -y
RUN echo "PasswordAuthentication yes" >> /etc/ssh/sshd_config
RUN echo "root:centos" | chpasswd
RUN ssh-keygen -A
CMD ["/usr/sbin/sshd","-D"]
EXPOSE 22
```

#### Important Points

- _CentOS is used as base image and openssh-server is installed to set up SSH._
- _**Password Authentication** is enabled and mentioned in SSH configuration file._
- _Password value is mentioned i.e., “**centos**” and generates host keys using **ssh-keygen**._
- _**sshd** service is enabled and port **22** is exposed._

<br>
-------

|| **vars : main.yml** ||

```yaml
---
# vars file for container_creation

mount_directory: "/dvd1"
dockerhub_username: XXXX
dockerhub_password: XXXX
port_number: "80"
```

#### Important Points

- _Name of directory to be mounted is mentioned._
- _Username and password required to login to **DockerHub** so as to push the image created is mentioned._
- _Port number is mentioned for **PAT(Port Address Translation)** for the **Apache Httpd** webserver setup within the container._

<br>
-------

|| **templates : inventory.yml** ||

```yaml
[container]
{{ container_ip.stdout }}  ansible_ssh_user=root  ansible_ssh_pass=centos
```

#### Important Points

- _“**container_ip**” variable is mentioned in the file using **Jinja2 templating**._
- _The value of the variable is obtained by retrieving the IP address of container created._

---

### Part 2 : Ansible Role (container_webserver_setup)

|| **tasks : main.yml** ||

```yaml
---
# tasks file for container_webserver_setup

- name: Httpd Installation
  package:
    name: "httpd"
    state: present
  register: httpd_install_status
  
- name: Web Page Setup
  copy:
    dest: "/var/www/html/index.html"
    src: "files/index.html"
    
- name: Starting Httpd Service
  command: "/usr/sbin/httpd"
  changed_when: httpd_install_status.changed == true
```

#### Important Points

- _**httpd** is installed in the container accessed via SSH by the Ansible using the inventory file updated in the previous role after container is created._
- _Webpage to be accessed via webserver are copied from the files directory to the **Document Root** within the container._
- _**httpd** service is enabled only when the state of “**Httpd Installation**” changes._

<br>
-------

|| **files : index.html** ||

```html
<b>Hello from Container</b>
```

<br>
<p align="center"><b>. . .</b></p><br>

## Output

![outside_virtual_machine](https://miro.medium.com/max/1400/1*Lysnah-T9QeX3XuseyiaPQ.png)

<p align="center"><b>Usage of Virtual Machine’s IP Address to access the web page launched within Docker container</b></p><br><br>

![within_virtual_machine](https://miro.medium.com/max/1016/1*dYkb8jyuQwFzyT_ltaazAw.png)

<p align="center"><b>curl command used to access the content of web page using Docker container’s IP within Virtual Machine</b></p><br><br>

<br>
<p align="center"><b>. . .</b></p><br>

**Note**:<br>
_The above Ansible roles could be executed in **RedHat Linux** only._

<h2>Thank You :smiley:<h2>

[![Linkedin](https://i.stack.imgur.com/gVE0j.png) LinkedIn](https://www.linkedin.com/in/satyam-singh-95a266182)

