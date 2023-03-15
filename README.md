# JupyterHub on Swedish Science Cloud

This guide is based on [torkar](https://github.com/torkar)'s excellent [tutorial](https://torkar.github.io/comp.html) about RStudio for SSC, and on my own [guide](https://github.com/simonlindgren/jupyterhub-setup) for setting up JupyterHub.

## Get the machine

If you work at a Swedish university, you can apply for free computing resources from Swedish Science Cloud.

1.  Go to [SNIC SUPR](https://supr.snic.se/) and set up an account (login with your university account).
2.  Create a new proposal and wait for it to be approved (usually a day or two).
3.  Go to https://cloud.snic.se and chose the appropriate region, for example 


As a researcher in computer science/software engineering, one often needs computational resources to conduct analyses. After having run computations on your local computers for some years you’ve finally figured out that this is neither sane nor cool. Hence, below follows instructions on how to get resources online for free.If you work at a Swedish university.

We will use SNIC as a resource since it’s free for university employees in Sweden. After some recent upgrades I would say that it’s very simple to use and setup. In short, you start by doing the following:If you find any problems following these instructions then please do tell me.

   
    Click Rounds and select SNIC Science Cloud.
    Create new proposal (make sure you ask for 1 year).
    Wait for it to be approved (24 h usually).
    Then go to https://cloud.snic.se and click WEST-1.If you belong to CTH/GU.

The current default seems to be that you’re issued 20 VCPUs, 50 GB RAM, 1 TB HD, and 5 floating IPs. Now is the time to make use of that :)

    Go to Compute → Images, and pick Ubuntu 18.04 (the CentOS version is way too old) by clicking Launch to the right.
    Details: Write an instance name, e.g., ProfTorkarIsALivingGod.
    Source: Make sure `Ubuntu 18.04` is picked.
    Flavor: Pick the hardware you want to use.YMMV but I personally use ssc.xlarge.highcpu with 16 VCPUs, since Stan now has support for within-chain parallelization.
    Networks: Pick `SNIC 20XX/YY-ZZ Internal IPv4 Network`.
    Network ports: Skip this for now.
    Security groups: Make sure `default` is picked.
    Key pair: Create Key Pair and make sure to download the private key in a safe space and make sure to restrict permissions on that file.chmod 600 key.pem on UNIX/LINUX/OSX.
    Click Launch instance and go to Compute → Instances.
    Once it’s up and running, click to the right and assign a floating IP to your internal IPv4 address.
    Go to Network → Security groups → Manage rules. Click Add Rule and add the rule for SSH. At the bottom you add the IP (or a range of IPs) that you want to allow access.e.g., 185.205.225.196/32 allows only that IP access to you environment, while 185.205.225.0/24 allows access to all IPs starting with 185.205.225.
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

         


