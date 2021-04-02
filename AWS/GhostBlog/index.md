# Setting up new Ghost blog on AWS EC2

This is a Ghost blog set up cookbook for AWS using a simple VPC with a pair of public and private subnets. The server initially will run on a single public subnet and instance. The database, if kept external to the host, will run on the private subnets.

I won’t go into details or the steps to set up a blog lab, so this requires a bit of understanding and experience on AWS. However, it should provide some learning challenges to get experience with troubleshooting.

## Some of the infrastructure needed

Let us create network and database components, EC2, security group and DNS hostname to start up the Ghost blog. Later we can add more AWS components such as CloudFront and ALB, log sessions, System Manager, AWS Inspector and others.

### First will set up network and security group environments

Create a new VPC with the following items:

- Name and Project tag for billing.
- IPv6 enabled.
- An Internet Gateway.
- Two public subnets. Remember to change their settings to provide public IP address. Name and tag them.
- Two private subnets. Name and tag them.
- A public route table with name, tag and routes to public IP space for both IPv4 and IPv6. It should have the **Internet Gateway** as the target. Associate both public subnets with the route table.

### Security Groups

Creating SG ahead saves time with back and forth moves when creating resources that depend on them. We will start with 3 SG for the following elements. Please create them in the order below:

- ELB-SG: grant HTTP and HTTPS access from the internet initially. Later when using CloudFront we can tight inbound traffic only to it, but to start setting up the blog, a simpler approach will help when troubleshooting.
- EC2-SG: The EC2 instance security group will need SSH from the internet so it can be managed (it will be an Ubuntu host). It should also allow HTTP and HTTPs connections from the **ELB-SG**. For now the inbound HTTP and HTTPS should be allowed from the internet as well as we don’t have ELB set up, which will be done later.
- DB-SG: This should be a simpler SG with a MySQL tcp/3306 inbound rule specifically from the **EC2-SG**. Only application server should be granted access to the MySQL server.

### Database set up

Whichever DB engine you choose, there is a need of a **subnet group** first. Create the subnet group that used both AZs private subnets.

Set up an RDS or Aurora MySQL instance. I will use an **Aurora Serverless** for this lab.

Launch the DB with admin name, password, subnet group, and the DB-SG created earlier. This will allow the DB to be reached from the EC2 instance.

### EC2 instance set up

Launch a new EC2 instance with the following characteristics :

- use one of the public subnets. Ideally we would have an application private  subnet for this, and run the set up using the load balancer for inbound connections and a NAT Gateway for outbound ones. However, to save time and cost during lab set up, will skip those resources at this time.
- Select an Ubuntu machine. Since we will use Ghost 3 we can launch an Ubuntu 20 now. We can use a **t4g.micro** 64-bit ARM instance. 
- Make sure the instance has a public IP address.
- Use the default 8GiB disk size, turn on encryption.
- Tag the machine name and project.
- Create or select an access key and launch the new EC2.

Log into the instance and update it using **apt-get**:

```bash
sudo apt-get update
sudo apt-get upgrade
sudo reboot #to apply the changes
```


### Set up DNS hostname 

Let us get Route 53 with our blog domain set up temporary to point to the blog.

Create a new **A** record to the machine public IP address. Do not enter a hostname at this time.

It is a good practice to add a record for the database server. For example, we can set a new **CNAME** record such as _db.domain.com_ pointing to the longer RDS or Aurora DB hostname. If you want to migrate the DB to other engine, restore a snapshot into newer instance or even move to a local MySQL install, you can just repoint the DNS record.

## Configuring Ghost

Now it is possible to test access to the EC2 by SSH. Connect to it using _ssh -i **name-of-key.pem** ubuntu@hostname_.

### New end user account to manage the blog

Setting up Ghost is described on their resource [page](https://ghost.org/docs/install/ubuntu/). Some tips below.

```bash
sudo adduser blog          #naming the new user blog
sudo usermod -aG sudo blog #adding user blog the sudoers group
sudo su - blog             #jump into the new account
```

### Install NGINX, MySQL & Node.js

```bash
#install NGINX
sudo apt-get install nginx

#Allow inbound firewall connections
sudo ufw allow 'Nginx Full'

#Install MySQL
sudo apt-get install mysql-server

#Add the NodeSource APT repository for Node 12
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash

#Install Node.js
sudo apt-get install -y nodejs

#Install Ghost-CLI
sudo npm install ghost-cli@latest -g
```


### Set up a new project directory

```bash
# We'll name ours 'ghost' in this example; you can use whatever you want
sudo mkdir -p /var/www/ghost

# Replace <user> with the name of your user who will own this directory
sudo chown blog:blog /var/www/ghost

# Set the correct permissions
sudo chmod 775 /var/www/ghost

# Then navigate into it
cd /var/www/ghost
```

And the final step configuring Ghost project

```bash
ghost install
```

Configure the website following Ghost setup procedure. You need to choose a  site hostname and set up SSL certificate with Let’s Encrypt. Even if you don’t have a domain registered to your lab, you can simulate requests by adding the hostname to your **etc/hosts** file pointing to the current site IP address.

If you are using Serverless Aurora, a change in the DB connections should help keep the cost to a minimal if you edit ** config.production.json** file and bring the minimal DB connections to zero.

```bash
  "database": "ghost_prod"
},
"pool": {
  "min": 0,
  "max": 10
}
  },
```

Now restart Ghost with _ghost restart_ command and hopefully the new set up will work fine.

## Certificate Manager

If we are adding an Amazon provide site certificate, and we have a domain registered (and it does not need to be an Amazon registered domain or in Route 53), we need to create a new one in Certificate Manager. 

So go to **Certificate Manager** in AWS console, request a new certificate for the **domain.com** and use the wildcard as an aditional name, such as **\*.domain.com**.

Amazon needs to validate your certificate using either DNS records you control or by a valid email address from the same domain. When using Route 53, AWS can create the records automatically, usually a CNAME. But if you use another DNS, just follow the instructions and create the very same record on your provider server.

![](CertManager.png "Amazon Certificate Manager issued certifcate")

After awhile you should get a valid and unused certificate. This certificate usually can be used inside some AWS services such as Elastic Load Balancer and CloudFront, preventing you from exporting to use elsewhere, however, Amazon recently has added a new feature that allows using it certificate in EC2 instances with the help of [Nitro Enclaves](https://aws.amazon.com/about-aws/whats-new/2020/10/announcing-aws-certificate-manager-for-nitro-enclaves/).

## Elastic Load Balancer

So setting up the ELB (Application type or ALB) will help us demonstrate the integration with other services later and allow possibility of adding more EC2 instances to support the blog and multiple blogs in different instances. Just mind that usually Ghost is not a clustered application meant be run on multiple machines, at least up to some v2.2 recent documentation says so. Maybe v3 or newer ones.

Here will have a listener for https tcp/443. When setting up the load balancer, we can use dual-stack internet facing to support IPv6 and external connections, and select both public subnets we created before inside the project VPC. Make sure you select both public subnets and not the private ones. 

Remember to tag the ELB with a Project name so you can filter costs down the road to this project, use the previously created security group for the ELB. We will update the EC2 security group later to allow only HTTP and HTTPS connections from the load balancer security group to protect the web server(s) from direct internet requests. This helps preventing DDoS and stack attacks to the web servers. Also we could apply AWS WAF (web application firewall) to the load balancer or to a CloudFront distribution. The TLS policy can be a newer one than 2016 standard offered by the AWS setup.

When setting up the ELB choose a targe group type of instance and manually select the instance for the web server. Use a regular health check option for it. Somehow I got no registered targets when setting up the ELB, so make sure revisiting the Target Group option add the instance later.

One comment, we are using a https listener, so we use only TLS from the load balancer to the web server. Usually this is not needed but this make simpler Ghost setup which expects https URL we get from ELB or CloudFront. Not the best way to set it up because it will require Lets Encrypt external connection monthly to renew the internal NGINX certificate, specially if you would like to confine the webserver to private subnets. Encrypting ELB connection to the webservers make them spent more CPU time, so it could cost more money on busy sites and you are already incurring encryption costs in the ELB or CloudFront.  On the other hand, some company policty could demand encryption on all legs of traffic flow making this setup valid.

We are keeping the demo simpler here onthe app setup, but please keep in mind assessing this choice. 

The target group should be healthy and should one registered host, but not much metrics. Let’s redirect the Route 53 site record from the manual IP address to the ELB as an Alias record, which is a R53 feature. If you are using an external DNS, you can point the site name to the external ELB endpoint record. You can find the DNS name in the ELB main page description tab.

So, go to the Route 53 site record and have it like below. You can add a health check to (mind the costs) and set up SNS alarms later.

![](Screen%20Shot%202020-11-21%20at%2008.12.14.png)

As a last note, if you are not using a domain you manage, just edit your **hosts** file and replace the manual IP address with the dynamic provided IP address from the ELB. Use **nslookup** or **dig** to find the IP address of your ELB. Just note that ELB IP addresses change from time to time and you might have to update your hosts file from time to time.

```bash
(base) iMac:Documents rodrigo$ dig CapricaTech-1953167413.us-east-1.elb.amazonaws.com

; <<>> DiG 9.10.6 <<>> CapricaTech-1953167413.us-east-1.elb.amazonaws.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55259
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;CapricaTech-1953167413.us-east-1.elb.amazonaws.com. IN A

;; ANSWER SECTION:
CapricaTech-1953167413.us-east-1.elb.amazonaws.com. 59 IN A 34.193.84.176
CapricaTech-1953167413.us-east-1.elb.amazonaws.com. 59 IN A 52.204.68.74

;; Query time: 10 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Sat Nov 21 08:15:08 -03 2020
;; MSG SIZE  rcvd: 111
```


So, to summarise, get a target group already using tls for health check and traffic to the instance, and so the Load Balancer. 

Test requests to the website and watch for metrics in the load balacer and the target group.

## CloudFront

CloudFront distribution can be created with the following characteristics:

- Use alternate name for the domain you have set up.
- Use the normal http to https redirection here. We all all traffic using TLS.
- There should be a regular or default behaviour here. But for the admin page which is usually access by the /ghost path, please create another behaviour with:
	- Path changed to /ghost\*
	- Redirect HTTP to HTTPS
	- Allowed HTTP Methods to the last option, all methods.
	- Keep legacy cache settings for now.
	- Make sure host header is whitelisted.
	- Forward all cookies
	- Query String Forwarding is set to fwd all based on all cache.
- Save and the blog should allow login and administration as usual.

Remember to edit the Route 53 setting to reflect the site alias from the original ELB to the new CloudFront distribution.

The website should be accessible by its URL using either Route53 alias record, another DNS providers pointing to the CloudFront distribution host or an entry on your hosts file.