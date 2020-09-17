# Foundry Virtual TableTop (VTT) Docker Deployment

[Foundry](https://foundryvtt.com)VTT can be deployed on a remote server and have it accessible anywhere and anytime. The server itself does not need to be powerful as all the rendering and heavy task is done on the browser of the client. I am running my FoundryVTT instance on a 1vCPU with 1GB RAM and have not encountered any issues as of yet.

This repository contains instructions for self-hosting [Foundry](https://foundryvtt.com)VTT service on your own computer or server using Docker. These instructions have been tested on ArchLinux and Ubuntu 20.04 LTS, however they should would unmodified on other platforms. 

This instance has been deployed on a [Digital Ocean](https://m.do.co/c/cd25d8b32728) VPS provider (contains affiliate link) but other VPS cloud providers can be used.

## Pre-requisites

Before proceeding, make sure the following are installed and configured 

- `Docker`
    - On Ubuntu 20.04
        ```bash
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

        sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

        sudo apt install docker-ce docker-ce-cli

        sudo usermod -a -G docker <username> # replace <username> with your own username. You need to logout and login again for changes to take effect

        sudo systemctl enable docker

        sudo systemctl start docker
        ```
    - On ArchLinux (make sure the commands are run as the `root` user)
        ```bash
        pacman -Syu docker

         sudo usermod -a -G docker <username> # replace <username> with your own username. You need to logout and login again for changes to take effect

        sudo systemctl enable docker

        sudo systemctl start docker
        ```

- `Docker Compose`
    - On Ubuntu 20.04
        ```bash
        sudo apt install docker-compose
        ```
    - On ArchLinux (make sure the commands are run as the `root` user)
        ```bash
        pacman -Syu docker-compose
        ```

- `Git` Distributed Version Control System
    - On Ubuntu 20.04
        ```bash
        sudo apt install git
        ```
    - On ArchLinux (make sure the commands are run as the `root` user)
        ```bash
        pacman -Syu git
        ```

- License key for FoundryVTT

- FoundryVTT Linux Zip file copied to the server. Assume the location is `$HOME/foundryvtt-0.6.6.zip`. Replace `foundryvtt-0.6.6.zip` with the latest filename of FoundryVTT.


## Initial setup

1. Start by cloning this repository to a folder on your server
```bash
git clone https://github.com/svicknesh/foundryvtt-docker-deploy $HOME/docker/foundryvtt
```

2. Extract Foundry installer. Replace `foundryvtt-0.6.6.zip` with the latest filename of FoundryVTT. The destination folder **MUST** be `latest`.
```bash
unzip $HOME/foundryvtt-0.6.6.zip -d $HOME/docker/foundryvtt/build/latest
```

### Customization

The default setup is sufficient for a single use deployment. If the need arises to run multiple instances for co-hosting them on a single server.

Edit `$HOME/docker/foundryvtt/docker-compose.yml` and add the following at the end of the file, matching the indentation of the existing `default` content in the file

```yaml
<instance_name>:
    volumes:
      - ./data/<data_folder>:/data 
    ports:
      - <port_number>:30000 
    restart: unless-stopped
```

Replace the following placeholders with their respective values

Placeholder | Format | Description 
:--- | :--- | :---
<instance_name> | **string** | Instance name to identify this container. Alphanumeric and underscore preferred.
<data_folder> | **string** | folder to store data related to `<instance_name>`. Alphanumeric and underscore preferred.
<port_number> | **number** | Port number this instance will listen on.

The above can be repeated as many times as needed. Each instance **MUST** have its own License Key.


## Starting FoundryVTT

Once you have your customizations set-up, run the following in the `$HOME/docker/foundryvtt` folder
```bash
docker-compose up -d
```

To view the logs for docker, run the following command
```bash
docker-compose logs -f
```

Open up your browser (Firefox, Vivaldi, Chrome) on your workstation and enter `http://<address>:<port>/` of your FoundryVTT instance where `<address>` is the IP Address of your computer or server and `<port>` is the one configured in the `docker-compose.yml` file. By default, the port is `30001`.

Foundry is set to start automatically on every system start-up.


## Securing connection to FoundryVTT using TLS (Optional)

To have FoundryVTT securely accessible anywhere from the world, I recommend using TLS. While you can generate keys for it manually and configure it in FoundryVTT, I find it much easier to deploy a web server that will manage certifiate management and renewal automatically using a domain name. My preferred web server is [caddy](https://caddyserver.com) but you can use any other webserver of your choice.

### Pre-requisites

This requires you to have a valid domain name that you can control. There are many providers for domain names that offer it cheaply.

The instructions below can be used on any Linux distributions. Run the following commands as the `root` user.


### Caddy v2

1. Download the `caddy` webserver
    ```bash
    sudo curl "https://caddyserver.com/api/download?os=linux&arch=amd64&idempotency=81212248794919" -o /usr/bin/caddy
    ```

2. Make the `caddy` binary executable
    ```bash
    sudo chmod 755 /usr/bin/caddy
    ```

3. Copy the following files to their respective destinations
    ```bash
    sudo cp $HOME/docker/foundryvtt/caddy/caddy.service /etc/systemd/system/caddy.service
    sudo cp $HOME/docker/foundryvtt/caddy/caddy.tmpfiles /usr/lib/tmpfiles.d/caddy.conf
    sudo cp $HOME/docker/foundryvtt/caddy/caddy.sysusers /usr/lib/sysusers.d/caddy.conf
    ```

4. Create the user and folders necessary for caddy
    ```bash
    sudo systemd-sysusers /usr/lib/sysusers.d/caddy.conf
    sudo systemd-tmpfiles --create /usr/lib/tmpfiles.d/caddy.conf 
    ```

5. Create the caddy configuration file and folder
    ```bash
    sudo mkdir -p /etc/caddy/config.d
    sudo cp $HOME/docker/foundryvtt/caddy/Caddyfile /etc/caddy/Caddyfile
    sudo cp $HOME/docker/foundryvtt/caddy/foundryvtt.conf /etc/caddy/config.d/foundryvtt.conf
    ```

6. Edit the `/etc/caddy/config.d/foundryvtt.conf` config file and change `https://foundryvtt.example.com:30001` to your own domain name and port number. Change `30001` in the line `reverse_proxy * http://localhost:30001` to the port number used during the docker configuration. Save and quit the file.

7. Start caddy
    ```bash
    sudo systemctl enable caddy
    sudo systemctl start caddy
    ```

8. Visit `https://foundryvtt.example.com:30001` on your browser and you will be presented with your FoundryVTT secured using TLS.
