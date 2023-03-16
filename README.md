# JupyterHub on Swedish Science Cloud

This guide is based on [torkar](https://github.com/torkar)'s excellent [tutorial](https://torkar.github.io/comp.html) about RStudio for SSC, and on my own [guide](https://github.com/simonlindgren/jupyterhub-setup) for setting up JupyterHub.

## Get access to SSC

If you work at a Swedish university, you can apply for free computing resources from Swedish Science Cloud.

1.  Go to [SNIC SUPR](https://supr.snic.se/) and set up an account (login with your university account).
2.  Create a new proposal and wait for it to be approved (usually a day or two).
3.  Go to https://cloud.snic.se and chose the appropriate region (WEST-1, EAST-1, NORTH-1).

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
        - Create a Key Pair.
        - Put it in a *.pem file on your local system. I named mine `key.pem`.
        - Make sure to restrict permissions on that file (e.g.,`chmod 600 key.pem`).
    - Remaining settings: Do nothing.
    - Click `Launch Instance`.
  3. Go to Compute → Instances and make sure that the Power State is `Running`.
    - Click to the right and associate a floating IP with your internal IPv4 address.
  4. Go to Network → Security groups → Manage rules.
    - Click `Add Rule` and add the rule for SSH. Leave the CIDR at `0.0.0.0/0`. 

    - 
  
        If you would like to run RStudio Server then you might as well also make sure that port 8787 is open for your IP.

Now you can SSH to your environment:The IP you’ve been assigned you can find on the page Compute → Instances.

            
ssh -i /path/to/key.pem ubuntu@<server-ip>

          

If you want RStudio Server, then continue reading. First, create a user,

            

sudo adduser rstudio
            

          

Next, let’s update the list where Ubuntu finds installation files (below we pick the R version 4 repository) and add the signed keys for the packages we will install. Start a shell as superuser by typing sudo -s in the prompt and then enter (do not forget to logout of the superuser shell with ctrl-D after having executed the below lines),

            

echo "deb https://cloud.r-project.org/bin/linux/ubuntu bionic-cran40/" >> /etc/apt/sources.list
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E298A3A825C0D65DFD57CBB651716619E084DAB9

            

          

Then, update the system, install a package to handle deb installation files (which RStudio Server is delivered as), install R, and install wget to download files,

            

sudo apt-get update
sudo apt-get upgrade
sudo apt-get install gdebi-core r-base r-base-dev wget
            

          

Finally, download and install the latest version of RStudio, which you can find here,At the time of writing, rstudio-server-1.4.1717 was the latest version.

            

wget https://download2.rstudio.org/server/bionic/amd64/rstudio-server-2022.02.1-461-amd64.deb
sudo gdebi rstudio-server-2022.02.1-461-amd64.deb
            

          

If you now point your browser to your server’s IP, you can login with rstudio as username and the password you set previously for that username,

            
http://<server-ip>:8787

          

If it doesn’t work, login with SSH and run,

          
sudo rstudio-server verify-installation

         


