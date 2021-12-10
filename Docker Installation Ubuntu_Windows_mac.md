
# Install and Use Docker and Docker-Compose on Ubuntu, Windows and Mac

# Ubuntu

## Step 1 — Installing Docker
First, update your existing list of packages:
```
sudo apt update
```
```
sudo apt install apt-transport-https ca-certificates curl software-properties-common
 ```
Then add the GPG key for the official Docker repository to your system:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
Add the Docker repository to APT sources:
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
```
Make sure you are about to install from the Docker repo instead of the default Ubuntu repo:
```
apt-cache policy docker-ce
``` 
You’ll see output like this, although the version number for Docker may be different:

Output of apt-cache policy docker-ce
```
docker-ce:
  Installed: (none)
  Candidate: 5:19.03.9~3-0~ubuntu-focal
  Version table:
     5:19.03.9~3-0~ubuntu-focal 500
        500 https://download.docker.com/linux/ubuntu focal/stable amd64 Packages
```

Finally, install Docker:
```
sudo apt install docker-ce
``` 
Docker should now be installed, the daemon started, and the process enabled to start on boot. Check that it’s running:
```
sudo systemctl status docker
``` 
The output shouldshow that the service is active and running.

## Step 2 - Executing the Docker Command Without Sudo (Optional)

By default, the docker command can only be run the root user or by a user in the docker group, which is automatically
created during Docker’s installation process. If you attempt to run the docker command without prefixing it with sudo 
or without being in the docker group, you’ll get an error.

If you want to avoid typing sudo whenever you run the docker command, add your username to the docker group:
```
sudo usermod -aG docker ${USER}

```

To apply the new group membership, log out of the server and back in, or type the following:
```
su - ${USER}
```

Verify installation by typing the command: 
```
docker info
```

## Step 3 - Install Docker-Compose

Run the following command to copy docker-compose binary from GitHub:

```
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

```

Fix permissions after download:
```
sudo chmod +x /usr/local/bin/docker-compose
```
Verify installation:
```
docker-compose version
```

 
# Windows

#### Windows installation is pretty simple, follow the official documentation below:
https://docs.docker.com/desktop/windows/install/

## Docker-Compose
Docker Desktop for Windows includes Compose along with other Docker apps, so Mac users
do not need to install Compose separately. 

# Mac 

### Follow the link below to download and install Docker desktop

https://docs.docker.com/desktop/mac/install/

## Docker-Compose
Docker Desktop for Mac includes Compose along with other Docker apps, so Mac users
do not need to install Compose separately. 

