# Installing Cisco Secure Access Connector on Proxmox LXC Container

This guide walks you through the installation and configuration of a Cisco Secure Access Connector docker image on a Ubuntu 22.04 LTS Proxmox Container. The guide includes creating a suitable container with TUN/TAP device enabled, which is necessary for the Cisco Secure Access Connector to function properly.

Reference: [Deploy a Resource Connector in Docker](https://docs.sse.cisco.com/sse-user-guide/docs/deploy-a-resource-connector-in-docker)

## Prerequisites

- A Proxmox VE server
- Root access to the Proxmox host

## Creating an Ubuntu 22.04 LTS Container Using Web GUI

1. Log in to the Proxmox web interface

2. Download the Ubuntu 22.04 template:
   - Select your node in the server view
   - Go to "Local" storage under the "Storage" section
   - Click on the "CT Templates" tab
   - Click "Templates" button
   - Search for "ubuntu-22.04" and click "Download"

3. Create a new container:
   - Click on "Create CT" button in the top right corner
   - Enter a Container ID (e.g., 100)
   - Set a hostname (e.g., ubuntu2204)
   - Set a password and confirm it
   - Optionally, add an SSH public key

4. Select the template:
   - Choose "ubuntu-22.04-standard_22.04-1_amd64.tar.zst" from the Template dropdown

5. Configure disk:
   - Set storage location and disk size (e.g., 8 GB)

6. Configure CPU:
   - Set the number of cores (e.g., 2)

7. Configure memory:
   - Set the amount of RAM (e.g., 2048 MB)
   - Set swap size if needed (e.g., 512 MB)

8. Configure network:
   - Keep default settings or customize as needed
   - Make sure "Bridge" is set to your network bridge (usually vmbr0)
   - Choose either DHCP or a static IP

9. Review and confirm:
   - Check all settings
   - Click "Finish"

10. Enable TUN/TAP device for the container:
    - SSH into your Proxmox host as root
    - Locate your container's configuration file:
      ```bash
      cd /etc/pve/lxc/
      ls -la
      # Find your container's ID (e.g., 100.conf, 101.conf, etc.)
      ```
    - Edit the container configuration file:
      ```bash
      nano /etc/pve/lxc/YOUR_CONTAINER_ID.conf
      ```
    - Add these two lines at the end of the file:
      ```
      lxc.cgroup2.devices.allow: c 10:200 rwm
      lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
      ```
    - Save the file and exit the editor (Ctrl+O, Enter, Ctrl+X)

11. Start the container:
    - Select your new container from the server view
    - Click "Start"

## Preparing the Container

Before installing the Cisco Secure Access Connector, we need to prepare the container environment:

1. Access your new container using the Proxmox web GUI console

2. Update the system packages:
   ```bash
   # Update the package lists
   apt-get update
   
   # Upgrade installed packages
   apt-get upgrade -y
   
   # Install required packages
   apt-get install -y curl sudo vim
   ```

## Installing Cisco Secure Access Connector

After creating and starting your Ubuntu 22.04 container with TUN/TAP support, follow these steps to install the Cisco Secure Access Connector.

The Secure Access Resource Connector setup script (`setup_connector.sh`) deploys the resource connector in Docker. This script downloads the scripts and configuration files required to set up and run the resource connector in the Docker container. You can run the `connector.sh` script to manage the resource connector (start and stop the connector). The `setup_connector.sh` script saves the scripts and configuration files in the `/opt/connector/install` directory.

1. Access your new container using the Proxmox web GUI console

2. Create a new user with sudo permissions:
   ```bash
   # Create a new user
   adduser cisco
   
   # Add the user to sudo group
   usermod -aG sudo cisco
   
   # Switch to the new user
   su - cisco
   ```

3. Run curl to get the Secure Access Resource Connector setup script (`setup_connector.sh`):
   ```bash
   curl -o setup_connector.sh https://us.repo.acgw.sse.cisco.com/scripts/latest/setup_connector.sh
   ```

4. Set the permissions on the script:
   ```bash
   chmod +x setup_connector.sh
   ```

5. Run the `setup_connector.sh` script:
   ```bash
   sudo ./setup_connector.sh
   ```

## Launch the Resource Connector in the Docker Container

Run the `connector.sh` script to deploy the Resource Connector in the Docker container. During the setup, the `connector.sh` script was saved in the `/opt/connector/install` directory on the host or VM.

Set the name of the resource connector with the `connector_name` variable. `connector_name` accepts a sequence of 1–40 alphanumeric characters, and the minus sign (-) or underscore (_) characters.

The `connector.sh` script sets up a service in the `/etc/service/connector_svc` directory. If the Docker container stops, the `connector.sh` service relaunches the container with the resource connector automatically.

1. Log into the Proxmox Container with a user that has sudo permissions.

2. Execute the following command to modify the connector.sh script to enable IP forwarding:
   ```bash
   sudo sed -i 's/ --cap-add=NET_ADMIN --device \/dev\/net\/tun:\/dev\/net\/tun \\/
   --cap-add=NET_ADMIN --sysctl net.ipv4.ip_forward=1 --device
   \/dev\/net\/tun:\/dev\/net\/tun \\/' /opt/connector/install/connector.sh
   ```

3. Run the `connector.sh` script. Provide the name of the Resource Connector (`connector_name`) and the provisioning key (`provisioning_key`) for the Resource Connector Group that includes the resource connector:
   ```bash
   sudo /opt/connector/install/connector.sh launch --name <connector_name> --key <provisioning_key>
   ```

4. Sign into Secure Access and confirm that the Resource Connector deployed. The resource connector that is deployed in the Docker container should provision and then display in Secure Access. For more information, see Add Connectors to a Connector Group: Confirm Connectors.

5. Configure the connector to start automatically at boot:
   ```bash
   # Edit the crontab for sudo
   sudo crontab -e
   
   # Add these lines to the crontab
   @reboot sudo /opt/connector/install/connector.sh stop
   @reboot /bin/sleep 30 ; sudo /opt/connector/install/connector.sh start
   ```
   This ensures that the connector stops and then restarts after a 30-second delay when the system boots.
