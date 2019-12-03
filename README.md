## AWS Certified Developer Associate
My personal notes.

## Author
Aditya Hajare ([Linkedin](https://in.linkedin.com/in/aditya-hajare)).

## Current Status
WIP (Work In Progess).

## License
Open-sourced software licensed under the [MIT license](http://opensource.org/licenses/MIT).

**************

## Table of Contents
- AWS Basics (IAM & EC2)
    - [AWS Regions](#aws-regions)
    - [IAM - Identity And Access Management](#iam-identity-and-access-management)
    - [Security Groups](#security-groups)
    - [Elastic IPs](#elastic-ips)
    - [SSH Into EC2 Instance And Install Apache](#ssh-into-ec2-instance-and-install-apache)
    - [EC2 User Data](#ec2-user-data)

- AWS Fundamentals: ELB + ASG + EBS

---

### AWS Regions
Each availability zone is a physical data center in the region, but separated from the other ones (so that they're isolated from disasters).

### IAM - Identity And Access Management
- IAM has a global view.
- Whole AWS security is there:
    * Users
    * Groups
    * Roles
- Never use **Root** account except for initial setup
- One IAM **User** per physical person.
- One IAM **Role** per application.

### Security Groups
- Security groups are like firewall on EC2 instances.
- They regulate:
    * Access to ports.
    * Authorised IP ranges - IPv4 and IPv6.
    * Inbound network (from outside to instance).
    * Outbound network (from instance to outside).
- Security groups are locked down to a region/VPC combination.
- They live outside EC2. i.e. Security groups are not something that is running on EC2 instance.
- It's a good practice to create/maintain one separate security group for SSH access.
- By default:
    * All inbound traffic is blocked.
    * All outbound traffic is authorised
- Common gotchas:
    * If your application is not accessible (time out), then it's a security group issue.
    * If your application is giving "connection refused", then it's an application error. Or may be your application is not launched.

### Elastic IPs
- It is quite an uncommon pattern to use Elastic IPs. That is because:
    * We can have only 5 Elastic IPs in our account (although we can ask AWS to increate this limit.).
- Always **try to avoid using Elastic IP**.
    * They often reflect poor architectural decisions.
    * Instead, use a random public IP and register a DNS name to it.
    * One should use a load balancer and not use public IP.
- If we are not using elastic IP, when EC2 instance is stopped and then started again, it can change its public IP.
- If we need fixed public IP for our instance (god knows for what reason), then we go for Elastic IP.
- Elastic IP is a public IPv4 we own as long as we don't delete it.
- One elastic IP can be attached to one EC2 instance at a time.
- With elastic IP, we can mask the failure of an EC2 instance or software by rapidly remapping the address to another instance in our account.

### SSH Into EC2 Instance And Install Apache
- `ec2-user` is the default user in our EC2 instance machine.
- SSH command syntax: `ssh -i [PATH_TO_KEYFILE] [USER]@[PUBLIC_IP]`
- Copy public IP (In below example, we will use `192.168.0.1`).
- In terminal, type:
    ```
        # SSH into EC2 using public ip.
        ssh -i my-key-file.pem ec2-user@192.168.0.1

        # Get sudo access for our user.
        sudo su

        # Update packages.
        yum update -y

        # Install httpd.
        yum install -y httpd.x86_64

        # Start httpd.
        # NOTE: If we get error "bash:systemctl command not found", make sure to use "Amazon Linux 2" and not just "Amazon Linux".
        systemctl start httpd.service

        # Make sure system remains enabled across reboots. Below command will create a symlink.
        systemctl enable httpd.service

        # Lets do a quick CURL on localhost:80. It will give us default test HTML page code.
        curl localhost:80
    ```
- At this point if we visit public ip in our browser, we will get timeout. So clearly it's an issue with security groups configurations.
- Go to `Inbound Rules` security group settings and configure HTTP rule on port 80. Refer to following settings:
    ```
        Type: HTTP
        Protocol: TCP
        Port Range: 80
        Source: Custom 0.0.0.0/0
    ```
- Now, if we visit public ip in browser, we shall see test page.
- Default test page is located at:
    `/var/www/html/index.html`

### EC2 User Data
- It is possible to bootstrap our EC2 instances using an **EC2 User Data script**.
    * `Bootstrapping` simply means running commands when a machine starts.
    * Bootstrapping script is run only once on EC2 instance's **first start**.
- EC2 user data is used to automate boot tasks such as:
    * Installing updates.
    * Installing softwares.
    * Downloading common files from the internet.
    * Almost anything you can think of.
- **EC2 User Data Script is run with a ROOT user**. i.e. every command will be ran with `sudo` rights.
- Example:
    1. Launch new EC2 instance: `Amazon Linux 2 AMI (HVM), SSD Volume Type`.
    2. Select `t2-micro` and click on **Configure Instance Details**.
    3. Scroll down to **Advanced Details** -->  **User Data**
    4. Select **As Text** radio button and paste the entire example script from below:
        ```
            #!/bin/bash

            #####################################################
            # USE THIS FILE IF YOU HAVE LAUNCHED AMAZON LINUX 2 #
            #####################################################

            # Get admin priviledges. Although it is not required to do so since the script will be run with ROOT user.
            sudo su

            # Install httpd (Linux 2 version)
            yum update -y
            yum install -y httpd.x86_64
            systemctl start httpd.service
            systemctl enable httpd.service
            echo "Hello world from Aditya at $(hostname -f)" > /var/www/html/index.html
        ```
    5. Make sure EC2 instance is having a security group with `HTTP PORT 80` and `SSH` policy enabled.
    6. Launch the instance and visit EC2's public ip in browser. We shall see the test page.
    7. **NOTE:** With above script, we have automated entire flow from above - [SSH Into EC2 Instance And Install Apache](#ssh-into-ec2-instance-and-install-apache)

### Load Balancing
- Types of load balancer:
    1. Classic Load Balancer (v1 - Old Generation) - 2009
    2. Application Load Balancer (v2 - New Generation) - 2016
    3. Network Load Balancer (v2 - New Generation) - 2016
- Amazon recommends using new generation/v2 load balancers as they provide more features.
- Health Checks:
    * Health checks are done on port and route (/health is common).
    * If the response is 200 (OK) then instance is healthy.
- Application Load Balancer (ALB) - Layer 7 - Allow to do:
    * Load balancing to multiple HTTP applications across machines (Target Groups).
    * Load balancing to multiple applications on the same machine (for e.g. containers).
    * Load balancing based on **route** in url.
    * Load balancing based on **hostname** in url.
- Basically load balancers are osum for mico-services and container-based applications (For e.g. Docker and Amazon ECS).
- Load balancers has a port mapping feature to redirect to dynamic port.
- **Application Load Balancer v2** - Layer 7 - osum features:
    * Stickiness can be  anabled at target group level:
        * Same request goes to the same instance.
        * Stickiness is directly generated by the ALB (not the application).
    * ALB supports HTTP, HTTPS and Websockets protocol.
    * The application servers don't see the ip of client directly.
        * The true ip of the client is inserted into header **X-Forwarded-For**
        * The port of client's request is inserted into header **X-Forwarded-Port**.
        * The prototype of client's request is inserted into header **X-Forwarded-Proto**.
- **Network Load Balancer v2** - Layer 4 - allow to:
    * Forward TCP traffic to your instance.
    * Handle millions of request per second.
    * Support for static ip or elastic ip.
    * Less latency ~100 ms (ALB has ~400 ms latency).
    * They are chosen for extreme performance and should not be the default load balancer you choose.
    * Creation process is quite same as Application Load Balancer.
- **Load Balancer 101**
    * Classic load balancers are deprecated.
    * `Application Load Balancer` is for HTTP, HTTPS and Websockets.
    * `Network Load Balancer` is for TCP.
    * `CLB` and `ALB` supports SSL certificates and provide SSL termination.
    * All load balancers have `health check` capability.
    * `ALB` can route based on `hostname` and `path (i.e. route based)`.
    * `ALB` is great fit with **ECS** (Docker).
    * Any load balancer has a static host name. IP resolved from hostname should never ever be used.
    * `Load Balancers` can scale but not instantaneously. Ask AWS for a `warm up`.
    * `NLB` (Network Load Balancer) directly see client ip at application tier.
        * There is no `X-Forwarded-For` or `X-Forwarded-Port` headers.
    * `4xx` errors are client induced errors.
    * `5xx` errors are application induced errors.
        * Load balancer error `503` means at capacity or no registered target.
    * If `Load Balancer can't connect to your application`, check your **Security Groups**.