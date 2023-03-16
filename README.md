# JupyterHub on Swedish Science Cloud

This guide is based on [torkar](https://github.com/torkar)'s excellent [tutorial](https://torkar.github.io/comp.html) about RStudio for SSC, and on my own [guide](https://github.com/simonlindgren/jupyterhub-setup) for setting up JupyterHub.

## Get access to SSC

If you work at a Swedish university, you can apply for free computing resources from Swedish Science Cloud.

1.  Go to [SNIC SUPR](https://supr.snic.se/) and set up an account (login with your university account).
2.  Create a new proposal and wait for it to be approved (usually a day or two).
3.  Go to https://cloud.snic.se and choose the appropriate region (WEST-1, EAST-1, NORTH-1).

## Create a machine

1. Go to Compute → Images. Pick OS and click `Launch` to the right. I chose Ubuntu 22.04.
2. 'Launch Instance' settings:
    - Details: Enter an instance name, e.g. RebelAllianceMegaHub.
    - Source: Make sure that your chosen OS is picked.
    - Flavor: Pick desired hardware. I chose `ssc.xlarge.highcpu`.
    - Networks: Pick the NAISS network.
    - Network Ports: Skip for now.
    - Security Groups: Make sure `default` is picked.
    - Key Pair:
        - Create a Key Pair. Give it a name and leave Key Type empty.
        - Save the key in a *.pem file on your local system. I named mine `key.pem`.
        - Make sure to restrict permissions on that file (e.g.,`chmod 600 key.pem`).
    - Remaining settings: Do nothing.
    - Click `Launch Instance`.
  3. Go to Compute → Instances and make sure that the Power State is `Running`.
        - Click to the right and associate a floating IP with your internal IPv4 address.
  5. Go to Network → Security groups → Manage rules.
        - Click `Add Rule` and add the rule for SSH. Leave the CIDR at `0.0.0.0/0`.
        - Also open ports `443`(HTTPS) and `80`(HTTP).

## Install JupyterHub 
1. SSH to the machine: `ssh -i /path/to/key.pem ubuntu@<external-(floating)-server-ip>`.
2. Become root `sudo -i`
3. Set up Anaconda Python.
    - Download and install Anaconda (most recent). Then remove the installer, and activate anaconda.

        ```
        wget https://repo.anaconda.com/archive/Anaconda3-2022.10-Linux-x86_64.sh
        bash Anaconda3-2022.10-Linux-x86_64.sh

        # Choose to install under opt: [/root/anaconda3] >>> /opt/anaconda3
        # When asked, say 'yes' to 'conda init'

        rm Anaconda3-2022.10-Linux-x86_64.sh

        source /opt/anaconda3/bin/activate
        
        # Create and activate a conda environment for use with jupyterhub
        conda create -n jhub
        conda activate jhub
        ```
3. Install `pip` for Python.
    ```
    sudo apt update
    sudo apt upgrade
    sudo apt install python3-pip
    ```
4. Install Jupyter
    ```
    conda install -c conda-forge jupyterhub
    conda install -c conda-forge jupyterlab
    sudo pip install importlib-resources
    ```

5. Configure JupyterHub
    - `node` setup
        - To be able to run jupyterlab extensions, it it crucial to have a recent version of `node` installed under anaconda.
        
        Run `node -v` to make sure that your version is recent (i.e., 16.x).
        
        If not, install node:
        ```
        curl -sL https://deb.nodesource.com/setup_16.x | sudo -E bash -
        sudo apt install -y nodejs
        ```       
        Check where your `node`'s are and symlink to the right one. `which node` tells you where the node command is currently pointing. `which -a node` will find paths to any node installations. Use the `ln -s` command to symlink the node command to point to the newer version.

    - Enable JupyterLab for JupyterHub (optional).

        ```
        jupyter labextension install @jupyterlab/hub-extension
        ```
        
    - Set up SSL on the server to be able to use https:
        ```
        sudo apt install letsencrypt
        sudo certbot certonly
        ```

        Choose: `1: Spin up a temporary webserver (standalone)`, then go through the generation process.

        We now have certificates in the `/etc/letsencrypt/live/<your.address>` folder: `fullchain.pem` is the certificate, and `privkey.pem` is the key file.
     - As we want to use port 443, we must remove the Linux default setting that prohibits users other than root to open ports below 1024.
        - `sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80`
        - Also do this, to remember the change at reboot: 
            - `sudo -i`
            - `echo net.ipv4.ip_unprivileged_port_start=80 > /etc/sysctl.d/99-reduce-unprivileged-port-start-to-80.conf`
            - `exit`
     
     - Now generate a config file for Jupyterhub in a directory:

        ```
        mkdir jupyterhub # <-- note that this will then be created under /home/ubuntu 
        cd jupyterhub
        jupyterhub --generate-config
        ```
        
        Copy the SSL certificates here.
        
        `sudo cp /etc/letsencrypt/live/digsum.net/fullchain.pem fullchain.pem`
        
        `sudo cp /etc/letsencrypt/live/digsum.net/privkey.pem privkey.pem`
        
        `sudo chown -R ubuntu /home/ubuntu`
        
        Edit the config file:
        ```
        nano jupyterhub_config.py
        ```
        Add these things to it: 

        ```
        # CONFIGURATION FILE FOR JUPYTERHUB
        # Set up users
        c.Authenticator.admin_users = {'<name-of-your-first-admin-user>'}
        # Set up web stuff
        c.JupyterHub.ip = '<the.internal.ip.address.from.ssc>'
        c.JupyterHub.port = 443
        c.JupyterHub.ssl_key = '/etc/letsencrypt/live/<your.address>/privkey.pem'
        c.JupyterHub.ssl_cert = '/etc/letsencrypt/live/<your.address>/fullchain.pem'
        c.JupyterHub.cleanup_servers = True
        # Set up spawner
        c.Spawner.default_url = '/lab'
        c.Spawner.cmd = ['jupyter-labhub']
        c.Spawner.notebook_dir = '~/notebooks'
        ```

    - Make the first user:
    
        `sudo useradd -m <name-of-your-first-admin-user>`

        `sudo passwd <name-of-your-first-admin-user>`

    - Now make a `notebooks` directory under your first admin user's home dir, so that this user has a directory where the hub can spawn. Also give the admin user rights to that directory:

        ```
        sudo mkdir /home/<name-of-your-first-admin-user>/notebooks
        sudo chown -R <name-of-your-first-admin-user> /home/<name-of-your-first-admin-user>
        ```

