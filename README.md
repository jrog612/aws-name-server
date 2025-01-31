A DNS server that serves up your ec2 instances by name.

Usage
=====

```
aws-name-server --domain aws.bugsnag.com \
                --aws-region us-east-1 \
                --aws-access-key-id <access_key> \
                --aws-secret-access-key <secret_key>
```

This will serve up DNS records for the following:

* `<name>.aws.bugsnag.com` all your EC2 instances tagged with Name=&lt;name>
* `<n>.<name>.aws.bugsnag.com` the nth instances tagged with Name=&lt;name>
* `<role>.role.aws.bugsnag.com` all your EC2 instances tagged with Role=&lt;role>
* `<n>.<role>.role.aws.bugsnag.com` the nth instances tagged with Role=&lt;role>
* `<instance-id>.aws.bugsnag.com` all your EC2 instances by instance id.
* `<n>.<instance-id>.aws.bugsnag.com` all your EC2 instances by instance id.

It uses CNAMEs so that instances will resolve to internal IP addresses if you query from inside AWS,
and external IP addresses if you query from the outside.

Quick start
===========

There's a long-winded [Setup guide](#setup), but if you already know your way
around EC2, you'll need to:

1. Open up port 53 (UDP and TCP) on your security group.
2. Boot an instance with an IAM Role with `ec2:DescribeInstances` permission. (or use an IAM user and
   configure `aws-name-server` manually).
3. Install `aws-name-server`.
4. Setup your NS records correctly.

Parameters
==========

### `--domain`

This is the domain you wish to serve. i.e. `aws.example.com`. It is the
only required parameter.

### `--hostname`

The publically resolvable hostname of the current machine. This defaults
sensibly, so you only need to set this if you see a warning in the logs.

### `--aws-access-key-id` and `--aws-secret-access-key`

An Amazon key pair with permission to run `ec2:DescribeInstances`. This defaults to
the IAM role of the machine running `aws-name-server` or to the values of the environment
variables `$AWS_ACCESS_KEY_ID` and `$AWS_SECRET_ACCESS_KEY` (or `$AWS_ACCESS_KEY` and `$AWS_SECRET_KEY`).

### `--aws-region`

This defaults to the region in which `aws-name-server` is running, or `us-east-1`.

Setup
=====

These instructions assume you're going to launch a new EC2 instance to run
`aws-name-server`. If you want to run it on an existing server, adapt the
instructions to suit.

### 0. Go setting

```
git clone https://github.com/udhos/update-golang.git && cd update-golang && sudo ./update-golang.sh
echo "export GOPATH=$HOME/go" >> ~/.bashrc
echo "export GOROOT=/usr/local/go" >> ~/.bashrc
echo "export PATH=$PATH:$GORROT/bin:$GOPATH/bin" >> ~/.bashrc
```



### 1. Create an IAM role

[IAM Roles](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
let you give EC2 instances permission to access the AWS API. We will need our
dns machine to run `ec2:DescribeInstances`.

1. Log into the AWS web console and navigate to IAM.
2. Create a new role called *iam-role-aws-name-server*
3. Select the *Amazon EC2* role type.
4. Create a *Custom Policy* called *describe-instances-only* with the content:

    ```
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Action": ["ec2:DescribeInstances"],
        "Effect": "Allow",
        "Resource": "*"
      }]
    }
    ```

### 2. Create a security group

[Security groups](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html)
describe what traffic is allowed to get to your instance. DNS servers use UDP port 53 and TCP port 53.

1. Log into the AWS web console and navigate to EC2.
2. Create a new security group called *aws-name-server*
3. Configure it to have:

    ```
    # Type   # Protocol  # Port  # Source
    SSH      TCP         22      My IP     x.x.x.x/32
    DNS      UDP         53      Anywhere  0.0.0.0/0
    Custom   TCP         53      Anywhere  0.0.0.0/0
    ```

This will let you ssh in to the DNS server, and let anyone run DNS queries.

### 3. Launch an instance

I recommend running 64bit HVM-based EBS-backed Ubuntu 14.04 on a `t2.micro`
([ami-acff23c4](https://console.aws.amazon.com/ec2/home?region=us-east-1#launchAmi=ami-acff23c4)). You
can use whatever distro you like the most.

1. Log into the AWS web console and navigate to EC2.
2. Click "Launch Instance"
3. Select your favourite AMI (e.g. *ami-acff23c4*).
3. Select your favourite cheap instance type (e.g. *t2.micro*) (If you don't have VPCs yet, choose *t1.micro* instead)
4. Set IAM role to *iam-role-aws-name-server*
5. Skip through disks (the default is fine)
6. Skip through tags (though if you set Name=dns1 and Role=dns you can test the server :)
7. Select an existing security group `sg-aws-name-server`.
8. Launch!

### 4. Build binary from source

1.  Clone this repository and build

    ```
    git clone https://github.com/jrog612/aws-name-server.git && cd aws-name-server
    go mod init aws-name-server
    go get github.com/jrog612/goamz/aws github.com/jrog612/goamz/ec2 github.com/miekg/dns
    go build
    ```

2. Move the binary into /usr/bin.

    ```
    sudo cp aws-name-server /usr/bin
    sudo chmod +x /usr/bin/aws-name-server
    ```

3. (optional) Set the capabilities of aws-name-server so it doesn't need to run as root.

    ```
    # the cap_net_bind_service capability allows this program to bind to ports below 1024
    # when it us run as a non-root user.
    sudo setcap cap_net_bind_service=+ep /usr/bin/aws-name-server
    ```

### 5. Configure dns server

First, you need change aws ec2 default dns server.

1. Open `/etc/resolv.conf`
2. Remove `nameserver 127.0.0.53`
3. Add `nameserver 8.8.8.8` and `nameserver 1.1.1.1`
4. run command `sudo systemctl stop systemd-resolved` for stop aws default dns resolver


### 6. Configure supervisor

If you use supervisor you can use the provided supervisor script.
You'll need to change the script to reflect your hostname:

1. Open `supervisor/aws-name-server.conf` and change domain option value
2. `sudo apt install -y supervisor`
3. `sudo cp supervisor/aws-name-server.conf /etc/supervisor/conf.d`
4. `sudo supervisorctl update`

### 7. Configure NS Records

To add your DNS server into the global DNS tree, you need to add an `NS` record
from the parent domain to your new server.

Let's say you currently have DNS for `example.com`, and you're running
`aws-name-server` on the machine `ec2-12-34-56-78.compute-1.amazonaws.com`.  In
the admin page for `example.com`s DNS add a new record of the form:

```
# name             # ttl            # value
aws.example.com    300    IN   NS   ec2-12-34-56-78.compute-1.amazonaws.com
```

The TTL can be whatever you want, I like 5 minutes because it's not too long to wait if I make a mistake.

The value should be a hostname for your server that is directly resolvable (i.e. not a CNAME). The public
hostnames that Amazon gives instances are perfect for this.

Troubleshooting
===============

There's a lot that can go wrong, so troubleshooting takes a while.

### Did it start?

First try looking in the logs (`/var/log/aws-name-server.log` if you're
using upstart). If there's nothing there, then try `/var/log/syslog`.

### Is it running?
Try running `dig dns1.aws.example.com @localhost` while ssh'd into the machine.
It should return a `CNAME` record. If not, look in the logs, the chances are
the DNS server is not running.  This happens if your EC2 credentials are wrong.

### Is the security group configured correctly?
Assuming you can make DNS lookups to localhost, try running
`dig dns1.aws.example.com @ec2-12-34-56-78.compute-1.amazonaws.com` from your
laptop. If you don't get a reply, double check the security group config.

### Are the NS records set up correctly?
Assuming you can make DNS lookups correctly when pointing dig at the DNS
server, try running `dig NS aws.example.com`. If this doesn't return anything,
you probably need to update your `NS` records. If you've already done this, you
might need to wait a few minutes for caches to clear.

### Are you getting a warning about NS records in the logs but everything seems fine?
This happens when the `--hostname` parameter has been set or auto-detected to
something different from what you've configured the `NS` records to be. This
may cause hard-to-debug issues, so you should set `--hostname` correctly.
