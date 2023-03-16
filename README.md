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
2. Set up Anaconda Python.
    - Download and install Anaconda (most recent). Then remove the installer, and activate anaconda.

        ```
        wget https://repo.anaconda.com/archive/Anaconda3-2022.10-Linux-x86_64.sh
        sudo bash Anaconda3-2022.10-Linux-x86_64.sh

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

The install will default to the `sh` shell (e.g. in users' Terminals in JupyterHub). 
We want the `bash` shell to be default:

```
rm /bin/sh
ln -s /bin/bash /bin/sh
```

### 5. Starting and re-starting the hub
Use `screen` to set up the server as a session that will continue running when closing terminal.

Note that `jupyterhub` must be run in the same directory where `jupyterhub_config.py` is.

```
screen -S jupyterhub 
cd /etc/jupyterhub 
jupyterhub
```

Exit the `screen` session by ctrl+A+D.

Access JupyterHub in your browser at https://<your.address>/ . 

Stop the hub:

```
screen -r jupyterhub
<press ctrl+C>
```

If stopping the hub, make sure before relaunching it that all instances of jupyterhub and configurable-http-proxy are terminated:

```
ps aux | grep jupyter 
kill <process id, one or several> 

ps aux | grep configurable 
kill <process id, one or several>
```

This is now a working JupyterHub.

Below are some additional notes on configurations that can be made.

----

#### Adding users

To set up new users, first create them as users of the Linux server. 

```
useradd -m <user>
passwd <user>
mkdir /home/<user>/notebooks
chown -R <user> /home/<user>/notebooks
```

----

#### Installing packages

```
conda install <package name>
```

or:

```
pip <or pip3> install <package name>
```

----

#### Renewing SSL

Stop jupyterhub. Then:

```
sudo certbot renew
```

Then relaunch jupyterhub.

----

#### Updating Jupyter

Update jupyter notebook, lab, hub and other packages through:

```
conda update --all 
# or conda update —packagename
# conda update jupyter
# conda update jupyterlab
# conda -c conda-forge install jupyterhub
<etc>
```
----

#### Install R kernel
Install R:

```
apt install r-base r-base-dev libssl-dev libcurl3-dev curl
R
install.packages('IRkernel') 
IRkernel::installspec(user = FALSE)
q()
```
This will install R in `/usr/bin/R` which makes it accessible by typing `R` in Terminal for all users. It also installs R as a selectable kernel in Jupyter.


Installing R packages:

```
R
install.packages(“<package-name>")
q()
```
----

#### Install Julia kernel

Install julia, by downloading the most recent version. Then untar it, copy to `/opt` and symlink it for all users to access it. Finally remove the installer.

```
wget https://julialang-s3.julialang.org/bin/linux/x64/1.1/julia-1.1.0-linux-x86_64.tar.gz
tar -xvzf julia-1.1.0-linux-x86_64.tar.gz
sudo cp -r julia-1.1.0 /opt/
sudo ln -s /opt/julia-1.1.0/bin/julia /usr/local/bin/julia
rm -rf julia-1.1.0 
rm julia-1.1.0-linux-x86_64.tar.gz
```

Each user can open a Terminal and enter `julia` to run it at command line. Close julia with ctrl+D.

To make Julia appear as a selectable kernel for Jupyter Notebooks on the hub installation, _each individual user_ must (once) open Terminal in the Jupyter GUI and do:

```
julia
import Pkg
Pkg.add("IJulia”)
<press ctrl + D>
```

Wait for it to install. Refresh the browser. Try to create a new notebook, and Julia will be available as a kernel.

----

#### Making custom keyboard commands (as user)

In the jupyter interface (in your browser) make menu selections: Settings -> Advanced Settings Editor -> Keyboard Shortcuts

In the box "User Preferences", add (as an example):

```
{
       "shortcuts": [
        {
            "command": "terminal:create-new",
            "keys": [
                "Shift Accel T"
            ],
            "selector": "body"
        }
    ]
}
```

Click save button. This sets shift + accelerator key (i.e. command) + T to as a shortcut to launch Terminal.

----

#### Make plotly graphs work in the notebooks

Install lab extensions for plotly. Note that the `lab build` might take quite a few minutes.

```
pip3 install ipywidgets
jupyter labextension install @jupyter-widgets/jupyterlab-manager --no-build
jupyter labextension install plotlywidget --no-build
jupyter labextension install jupyterlab-plotly --no-build
jupyter lab build
```



