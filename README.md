# DevOps Tech-Talk: Puppet

## Team members

Yushu Huang (yhuang49)

Haoxu Ren (hren3)

Chang Yan (cyan3)

Mohit Sardar (msardar)

Darshan Patel (dpatel12)

## Links

[Screencast]()

[Report]()

## Puppet Setup Instructions

This section describes how to install Puppet master and agents on AWS EC2 instances.

### 1) Get EC2 Instances

Create 2 T2 micro instances (Free tier eligible Ubuntu Server 16.04 LTS) on EC2. 
One instance will act as master and the other as agent.

### 2) Update Security Groups

Edit inbound rules of the security group of master instance:

TCP 8140 - Agents will talk to the master on this port

TCP 22 - To login to the server/instance using SSH

### 3) Install Puppet Server

SSH to the master instance and run the below commands to install the puppetserver.
Puppet Agents pull the configuration from the Puppet server located on the Puppet master.

```
wget https://apt.puppetlabs.com/puppet5-release-xenial.deb
sudo dpkg -i puppet5-release-xenial.deb
sudo apt update
sudo apt install -y puppetserver
```

### 4) Configure memory allocation:

The Puppet Server is configured to use 2 GB of RAM by default which can be changed according to your requirements 
(like how many agent nodes your master will manage). To edit this memory allocation, run the below command and 
find the `JAVA_ARGS` line, 
and use the -Xms and -Xmx parameters to set the memory allocation.

`sudo vi /etc/default/puppetserver`

We have set it to use 512 MB.

`JAVA_ARGS="-Xms512m -Xmx512m -XX:MaxPermSize=256m"`

### 5) Start Puppet Server

In the master instance, open the 8140 port for communication with the agent nodes.

`sudo ufw allow 8140`

Start the Puppet Server using the following command:

`sudo systemctl start puppetserver`

You can check the status of Pupper Server using this command:

`sudo systemctl status puppetserver`

If it says `active (running)` in green color and `Started puppetserver Service` in the last line then the Puppet Server is running.

Now, use this command to configure Puppet Server to start at boot:

`sudo systemctl enable puppetserver`

### 6) Install the Puppet Agent

SSH to your agent node and edit the `/etc/hosts` file to add `<IP-address-of-puppet-server>  puppet`.
When you create the puppet master’s certificate, By default all puppet agents will try to find a master by name puppet.
We need to add this line in the hosts file so that puppet name should be resolved to your puppet master IP address.

Next run following commands to install and start the Puppet Agent:

```
wget https://apt.puppetlabs.com/puppet5-release-xenial.deb
sudo dpkg -i puppet5-release-xenial.deb
sudo apt-get update
sudo apt-get install -y puppet-agent
sudo systemctl enable puppet
sudo systemctl start puppet
```

### 7) Sign certificate on the Puppet master

When the puppet agent runs on an agent node for first time, it sends a certificate signing request to the Puppet master.
The Puppet Server needs to sign that certificate before it can communicate with the agent node.
You can see this certificate signing request on the master instance with the following command:

`sudo /opt/puppetlabs/puppet/bin/puppet cert list --all`

You will see an output similar to this - 

![screen shot 2018-04-08 at 3 35 24 am](https://media.github.ncsu.edu/user/6931/files/fe9c097c-3add-11e8-8fe2-285214e4042e)

The certificates with a `+` sign at the start indicates that it is already signed. 
Those certificates without a `+` sign are not yet signed. 
In the above output, `"ip-172-31-19-10.us-east-2.compute.internal"` is the fqdn of the agent node 
and there is no `+` sign for it which implies that it needs to be signed by the master.
Use the following command to sign the certificate:

`sudo /opt/puppetlabs/bin/puppet cert sign <fqdn-of-your-agent-node>`

If you have multiple requests, you can sign all of them at once using the `--all` option instead of writing the fqdn of a single agent nodes one at a time.

At this point, you have a working puppet master and agent. 
To apply configuration to Puppet agent we have to create manifests in puppet master.

### 8) Verify the Installation

Create a simple manifest in the master instance by making a default manifest in the default location:

`sudo vi /etc/puppetlabs/code/environments/production/manifests/site.pp`

Add the following code in the `site.pp` for creating a file `test.txt` with 0644 permissions in the `/tmp/test.txt` of the agent node.

```
file {'/tmp/test.txt':                            # resource type file and filename
  ensure  => present,                             # make sure it exists
  mode    => '0644',                              # file permissions
  content => "It works on ${ipaddress_eth0}!\n",  # Print the eth0 IP fact
}
```

Now run the following command to verify if the puppet is installed correctly:

`sudo /opt/puppetlabs/bin/puppet agent --test`

If the `test.txt` file is created in the agent node then you have successfully installed Puppet in Master/Agent mode.
