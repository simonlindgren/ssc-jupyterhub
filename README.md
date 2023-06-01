# JupyterHub on Swedish Science Cloud

This guide is based on [torkar](https://github.com/torkar)'s excellent [tutorial](https://torkar.github.io/comp.html) about RStudio for SSC, and on my own [instructions](https://github.com/simonlindgren/jupyterhub-setup) for setting up JupyterHub.

## Get access to SSC

If you work at a Swedish university, you can apply for free computing resources from Swedish Science Cloud.

1.  Go to [SNIC SUPR](https://supr.snic.se/) and set up an account (login with your university account).
2.  Create a new proposal and wait for it to be approved (usually a day or two).
3.  Go to https://cloud.snic.se and open a dashboard for the appropriate region (WEST-1, EAST-1, NORTH-1).

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
1. SSH to the machine: `ssh -i /path/to/key.pem ubuntu@<external-(floating)-server-ip>`
2. We will use [The Littlest Jupyterhub](https://tljh.jupyter.org/en/latest/) installation. This installation procedure needs to write to the `/opt/tjlh` and to the `/tmp` directories, but SSC does not allow storage space in those locations of the file system. Therefore, we create locations by these two names on an available storage drive, and mount them to the locations that the installer needs.

    - Our example storage location is `/mnt/blob`. So: `cd /mnt/blob`; `sudo mkdir tmp`; `sudo mkdir opt`; `cd opt`; `sudo mkdir tljh`; `cd /mnt/blob`.
    - With these locations created, we mount them: `sudo mount --bind /mnt/blob/tmp /tmp`; and `sudo mount --bind /mnt/blob/opt/tljh /opt/tljh`.
3. Now, follow through the TLHJ [installation instructions](https://tljh.jupyter.org/en/latest/install/custom-server.html).
