# JupyterHub on Swedish Science Cloud

## Get access to SSC

If you work at a Swedish university, you can apply for free computing resources from Swedish Science Cloud.

1.  Go to [SNIC SUPR](https://supr.snic.se/) and set up an account (login with your university account).
2.  Create a new proposal and wait for it to be approved (usually a day or two).
3.  Go to https://cloud.snic.se and open a dashboard for the appropriate region (WEST-1, EAST-1, NORTH-1).

## Create a machine

1. Go to Compute → Images. Pick OS and click `Launch` to the right. I chose Ubuntu 22.04.
2. 'Launch Instance' settings:
    - Details: Enter an instance name, and set `Volume Size (GB)` to 1000 (SSC allows max 1TB).
    - Source: Make sure that your chosen OS is picked.
    - Flavor: Pick desired hardware. I chose `ssc.xlarge.highmem`.
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
1. SSH to the machine: `ssh -i /path/to/key.pem ubuntu@<external-(floating)-server-ip>`
2. Now, follow through the TLHJ [installation instructions](https://tljh.jupyter.org/en/latest/install/custom-server.html).
3. Some additional stuff:
    - Change the default user interface to JupyterLab [here](https://tljh.jupyter.org/en/latest/howto/user-env/notebook-interfaces.html#changing-the-default-user-interface).
    - Activate https access [here](https://tljh.jupyter.org/en/latest/howto/admin/https.html).
    - Install stuff for all users by `sudo -E pip install something`; `sudo -E conda install something`
    - Give user permissions to their folder, if it gets messed up: `sudo chown -R jupyter-<user> /home/jupyter-<user>`
    - For my own use, I keep an updated script of all things I install to my hub --> [hubstuff.sh]()

