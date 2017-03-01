# minecraft-container
 
**Dockerfile + Instructions for Deploying Stateful Minecraft Server via Container**

  Inspired by [blog article from Julia Ferraioli](http://www.blog.juliaferraioli.com/2015/06/running-minecraft-server-on-google.html)
***

### Build and run a Dockerfile to run a stateless Minecraft server on local host
- Clone and pull the [Minecraft Server Dockerfile](../master/Dockerfile)
- Build Dockerfile with
```docker build -t {YOUR DOCKERHUB USERNAME}/ubuntu-minecraft-server:1.11.2 . ```
- Run minecraft server container with
```docker run -d -p 25565:25565 {YOUR DOCKERHUB USERNAME}/ubuntu-minecraft-server:1.11.2```
- Server should be accessible on local machine at ```127.0.0.1:25565```

**Note that this server will NOT save world data when shut down**
- Terminate the server with
```docker kill {CONTAINER ID}```

##### Adding state with local volume mounts
- Run the minecraft server container with the following volume mount:
```docker run -d -p 25565:25565 -v ~/minecraft-server-data/:/data/ {YOUR DOCKERHUB USERNAME}/ubuntu-minecraft-server:1.11.2```
- Terminate the server with
```docker kill {CONTAINER ID}```

**Bring up the server with the same command above, and world data will be persistent**

---

### Using AWS and REX-Ray to Host a Persistent Minecraft Server in the Cloud

#### Set up AWS VM
**Required: Amazon AWS Account +Account Access & Secret Keys**
- Navigate to the [AWS EC2 Dashboard](https://aws.amazon.com/console/ "AWS Management Console") and click the blue 'Launch Instance' button
 - Step 1: below select the "Ubuntu Server 16.04 LTS (HVM), SSD Volume Type" AMI
**Note, to advance to the next step, click the grey 'Next:...' button at the bottom of each page, NOT the blue 'Review and Launch' Button until instructed.**
 - Step 2: choose 't2.micro' - this determines the size and memory of our Virtual Machine, you may upgrade this later.
 - Step 3: the default under the subnet category will choose an availability zone for your vm automatically.
                This is fine, however upon creating future VMs, make sure they are all in the same subnet
                if you would like them to share storage volumes.

 - Step 4: leave defaults
 - Step 5: give the virtual machine a unique, descriptive name. Under 'Key' enter "Name" and under 'Value' enter your desired name such as "Minecraft_Server"
 - Step 6: configure the security group. Next to 'Security group name:' enter "minecraft_security_group". Below leave the SSH port, but underneath click 'Add Rule' and add a Custom TCP Rule
                enter "25565" in the Port Range field and set Source to 'Anywhere'
 - Step 7: review and hit the 'Launch' button, configure your [SSH keys](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) and launch the instance.
 You can [generate your own SSH key pair](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2) to import for use as well.
- Startup may take several minutes, navigate to the 'Instances' menu and when the 2 status checks complete, SSH into the machine

#### Install REX-Ray and Docker
At this point you should be SSH-ed into your AWS Ubuntu virtual machine, enter the following commands into the terminal:

``` sudo apt-get update ```

``` sudo apt-get upgrade -y ```

Then install [REX-Ray](http://rexray.readthedocs.io/en/stable/)

``` curl -sSL https://dl.bintray.com/emccode/rexray/install | sh ```

Modify the /etc/rexray/config.yml file(requires sudo)

``` sudo vi /etc/rexray/config.yml ```

Enter the following, replacing the AWS keys appropriately (from [REX-Ray Configuration Generator](http://rexrayconfig.codedellemc.com/)):
```
libstorage:
  service: ebs
ebs:
  accessKey: {YOUR AWS ACCESS KEY}
  secretKey: {YOUR AWS SECRET KEY}
```

Save and exit the config.yml file.

Continue in the SSH terminal, next we'll install Docker using curl and Docker's install script

``` curl -sSL https://get.docker.com/ | sh ```

To use Docker without sudo, we’ll have to add our user to the “docker” group.  Our username in this case is “ubuntu”.  You might have to change your username based on your settings for the AWS instance.

``` sudo usermod -aG docker ubuntu ```

Then logout

 ``` exit ```

#### Pulling and Running the Minecraft-Server Container 
 
SSH back into your AWS VM
Check that docker and rexray are installed and working correctly

```rexray version```

```docker ps```

List available storage volumes with 

```sudo rexray volume list```

Make sure the rexray daemon is running so we can link a persistent storage volume

``` sudo service rexray start ```

Let's create a storage volume for our Minecraft server

``` sudo rexray volume create --size=16 --volumename=mc-server-volume ```

Now if you look under Volumes in your AWS webpage "mc-server-volume" should show up

To run the minecraft server enter the following:

``` docker run -d -p 25565:25565 --volume-driver=rexray -v mc-server-volume:/data akiraw95/ubuntu-minecraft-server:1.11.2 ```

Wait for the image to pull from [dockerhub](https://hub.docker.com/r/akiraw95/ubuntu-minecraft-server/ "DockerHub image link") or build it yourself from the [Dockerfile](../master/Dockerfile)

Once docker spits out a container ID, the server is accessible from anywhere at the same IP that you SSH-ed into at port 25565.

To terminate the server enter: 

`docker kill {CONTAINER ID}`

**Bring up the server with the same command above, and world data will be persistent**
