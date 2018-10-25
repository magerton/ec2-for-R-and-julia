# Julia and R on Amazon EC2

Thank you to @arnonerba for figuring out how to get Julia compiled with Intel MKL and major revisions below.

## Purpose

This guide explains how to set up a simple Amazon Linux 2 instance on EC2 for use with R and Julia. Renting EC2 space can be quite cheap, especially if you use a Spot Instance. Pricing is a continuous uniform-price auction, and if you are outbid, your instance is terminated w/out warning.

The guide assumes basic familiarity with a UNIX-like systems (e.g., navigating the file structure, copying, moving, ssh, etc). Note that Windows 10 now includes the "Windows Subsytem for Linux" (WSL), which provides a very nice terminal environment ([MSDN setup guide](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide)). The Git Bash terminal is also a good choice for Windows. macOS users may make use of the built-in macOS terminal.

If you have suggestions, pull requests & edits are welcome!

## Launch an EC2 Instance

1. Sign up for an Amazon AWS account. Sign up for a [GitHub education pack](https://education.github.com/pack) if eligible and you may get some free Amazon AWS credits.
2. Spin up an Amazon Linux 2 instance. The "compute optimized" tier is recommended, and you should not need more than 16GB of storage.
3. Create a new SSH key pair when prompted, or choose one already saved in your AWS account. If you create a new pair, you will be asked to download your private key.
    - **Your private key must be kept secure**. By convention it should be placed in your local `~/.ssh` directory and be protected by either `0400` or `0600` permissions [(note)](https://superuser.com/questions/215504/permissions-on-private-key-in-ssh-folder). Your `~/.ssh` directory should have `0700` permissions.
    - Private keys may take several different forms:
        + `*.pem` - Standard file format for cryptographic keys/certificates. AWS uses this format.
        + `*.key` - Alternate file extension for a PEM file only containing a private key.
        + `*.ppk` - Proprietary PuTTY format for private keys. PuTTY does not support the PEM format.
    - Public keys utilize the `*.pub` extension, but when copied to a server are appended to your remote `~/.ssh/authorized_keys` file. The presence of your public key in this **remote** file grants you access to the server.
        + If you are manually copying a new public key to an instance you already have access to, use the `ssh-copy-id` command [(note)](https://askubuntu.com/questions/4830/easiest-way-to-copy-ssh-keys-to-another-machine). Otherwise, the AWS setup guide handles this process for you.
4. Connect to your EC2 instance via SSH. You can find the IP address/hostname of your instance in your AWS dashboard.
    - Append the following to your local `~/.ssh/config` file, substituting the appropriate values as necessary:
        ```shell
        Host your_server_name
            HostName your_ip_address_or_hostname
            User ec2-user
            IdentityFile ~/.ssh/your_private_key.pem
        ```
    - Then, SSH into the server with `ssh your_server_name`. 
    - Alternatively, you can skip the instructions above and connect directly with:
        ```shell
        ssh ec2-user@your_ip_address_or_hostname -i ~/.ssh/your_private_key.pem
        ```

## Install Software

### Git
- Install Git
    ```shell
    sudo yum install git
    ```
- To push and pull from GitHub over SSH, you will need another public/private key pair that is tied to your GitHub account [(note)](https://help.github.com/articles/adding-a-new-ssh-key-to-your-github-account/). If you do not have a key pair, generate one on your EC2 instance with `ssh-keygen` and add the public key to your GitHub account. If you already have an authorized key pair, copy the private key to your EC2 instance and place it in your remote `~/.ssh` directory:
    + On the local machine, navigate to your directory with relevant keys (usually `~/.ssh` or `%USERPROFILE%/.ssh`).
    + Use `sftp` to put your `github_rsa` private key on the remote server.
    + Exit `sftp`, and then `ssh` back into the server.
    + Move the private key into .ssh: `mv github_rsa .ssh/`.
    + Check that the permissions are correct: `ls -al .ssh`.

### Git LFS
- See [install guide](https://github.com/git-lfs/git-lfs#getting-started). I had to use [PackageCloud](https://packagecloud.io/github/git-lfs/install) to install from command line.
    ```shell
    curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.rpm.sh | sudo bash
    sudo yum install git-lfs
    git lfs install # only run once for initial install
    ```

### LFTP
- Install LFTP to connect to Box accounts.
    ```shell
    sudo yum install lftp
    ```

### Intel MKL
- Intel MKL is [available for free](https://software.intel.com/en-us/articles/how-to-get-intel-mklippdaal) from Intel. The [yum repository](https://software.intel.com/en-us/articles/installing-intel-free-libs-and-python-yum-repo) can be easily added on an Amazon Linux 2 system:
    ```shell
    sudo yum-config-manager --add-repo https://yum.repos.intel.com/mkl/setup/intel-mkl.repo
    sudo rpm --import https://yum.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS-2019.PUB
    ```
- Multiple versions of MKL are available, but the latest can be easily installed:
    ```shell
    sudo yum install intel-mkl
    ```

### Julia (Build From Source)

These instructions are for building Julia from source. Binary files are also available on the [Julia Downloads page](https://julialang.org/downloads/), and installation instructions are available [here](https://julialang.org/downloads/platform.html)

- First, install the necessary dependencies:
    ```shell
    sudo yum groupinstall 'Development Tools'
    sudo yum install make gcc gcc-c++ libatomic python gcc-gfortran perl wget m4 patch pkgconfig
    sudo yum autoremove cmake # default version is too old
    ```
- Download the Julia source code:
    ```shell
    wget https://github.com/JuliaLang/julia/archive/v1.0.1.tar.gz
    ```
- Extract Julia source code tarball and move to `/usr/local`:
    ```shell
    tar -xzvf v1.0.1.tar.gz
    mv julia-1.0.1/ /usr/local/julia-1.0.1/
    cd /usr/local/julia-1.0.1/
    ```
- (Optional) Enable MKL in Julia:
    ```shell
    source /opt/intel/bin/compilervars.sh intel64
    echo "USE_INTEL_MKL = 1" > Make.user
    ```
- Use `make` to compile Julia:
    ```shell
    ./contrib/download_cmake.sh # force Julia to build an updated version of cmake
    make -j4 # where '4' is the number of available CPU threads
    ```
- Symlink Julia to `/usr/local/bin`:
    ```shell
    ln -s /usr/local/julia-1.0.1/julia /usr/local/bin/julia
    ```
- Test MKL Integration in a Julia prompt:
    ```julia
    using LinearAlgebra
    LinearAlgebra.BLAS.vendor()
    ```

### Julia Packages
- Open up a julia prompt and install packages into the default folder
    ```bash
    ]add AxisAlgorithms BenchmarkTools Calculus CategoricalArrays DataFrames Distributions FileIO Formatting GLM GR Gadfly IndirectArrays Interpolations JLD2 MixedModels NLSolversBase NLopt Optim PkgDev Plots Primes Profile ProgressMeter PyPlot RData Ratios StatsBase StatsFuns StatsModels
    ```
- ~~Initialize package repo with `Pkg.init()` in julia~~ (deprecated in v0.7)
- ~~Bulk install by updating `REQUIRE` in `~/.julia/v0.x/REQUIRE` and running `Pkg.resolve()`. You may need to run julia as `sudo` with elevated priveleges, but hopefully not.~~ (deprecated in v0.7)

### R

No guide yet for installing R or R Studio server. See [Louis Aslett's page](http://www.louisaslett.com/RStudio_AMI/)

## Using Amazon EFS

A great way to make sure log files persist across sessions and in case your spot instance gets killed is to use Amazon's EFS storage. EFS storage is accessible only by instances in the same security group in the same region. So to access files there, you have to go through a running instance: you cannot SSH directly to the the EFS drive.

- Add an EFS instance (encrypted?)
- Install EFS utilities
    + If running Amazon Linux: `sudo yum install -y amazon-efs-utils`
    + If running Ubuntu:
        * install `amazon-efs-utils` (manually?) using <https://docs.aws.amazon.com/efs/latest/ug/using-amazon-efs-utils.html#installing-other-distro>
        * `apt-get install libssl-dev`
        * Upgrade `stunnel` and symlinmk it `sudo ln -s /usr/local/bin/stunnel /bin/stunnel` ~~and/or `sudo ln -s /usr/bin/stunnel /bin/stunnel`~~
- **Make sure that EFS and EC2 instances are in the same security group (SG), and that the SG has an inbound rule allowing NFS traffic from the same SG**
- One-time mounting can be done with `sudo mount -t efs fs-[INSTANCE_ID]:/ [TARGET MOUNT POINT]`
- Permanent mounting can be done by opening `/etc/fstab` with `sudo` privileges and adding a new line to: `fs-[INSTANCE_ID]:/ [TARGET MOUNT POINT SUCH AS /mnt/efs] efs defaults,_netdev,nofail 0 0`
- It can be nice to symlink the efs volume to a directory: `ln -s /mnt/efs efsdir`.

## How to get stuff done in the terminal/REPL

1. Launch your AMI from the EC2 console: `AMI > Select on your AMI > Under "Actions," select "Spot Request"` Request a big instance, and set the MAX price you are willing to pay per hour (usually it's much lower than this)
2. Once your AMI is running (can take a bit), SSH into it & get to work or point your browser to the relevant IP address.

## Set up notifications when your job dies

Sometimes things error out. We have not yet figured out how to get the instance to email us when a job errors out. However, to be notified when an instance's CPU utilization falls below a threshold,

- Create a subscription at [AWS SNS](https://console.aws.amazon.com/sns/v2/home)
- Under EC2 instances > Monitoring, click "create alarm". Set alarm for CPU utilization <= X pct for less than Y min.

## Using nohup

`nohup` is a way to run a script that will stay running after your terminal session is killed and have the script dump all STDOUT and STDERR to a log file. For example, we can run

```bash
nohup julia --optimize=3 ~/dev-pkgs/ShaleDrillingEstimation/example/run_estimator.jl > ~/efs-ubuntu/JOBNAME\ "$(TZ=America/Los_Angeles date +on\ %Y-%m-%d\ at\ %Hh%Mm)"\ by\ ${MYIP}.out 2>&1 &
```

where `JOBNAME` is to be filled in by you.

## Using LFTP

- One can use `lftp` to transfer files between AWS instances and Box.net in lieu of Git and git-lfs. Note that special characters in password may have to be escaped or translated to HTML.
    ```shell
    lftp -p 990 -u "username@institution,PASSWORD" ftps://ftp.box.com
    mirror [project_dir_on_box]   [remote_project_dir]
    ```

## X11 and macOS

If required, X11 can be easily used to run remote GUI applications on macOS.
- Install [XQuartz](https://www.xquartz.org/)
- Log out and log back in, then connect using `ssh -YC user@server` in Terminal to enable X11 forwarding.

## Remote Desktop Over SSH

Sometimes it is nice to use a GUI. This is pretty straightforward to do on AWS or Azure Ubuntu instances. Note that the instructions below are not tested on AWS Amazon linux.

- Install Remote Desktop
    + Install `xrdp` and `xcfe4` software as per <https://docs.microsoft.com/en-us/azure/virtual-machines/linux/use-remote-desktop>. We'll connect over SSH, so no need to open a special RDP port.
    + On local machine, create ssh tunnel to remote with port forwarding `ssh -L [LOCALPORT]:localhost:[3389] username@remoteip`. Can do this with terminal, Bash for Windows, or Putty.
    + After connecting to remote instance, set a password on the remote machine so that the RDP can log in `sudo setpasswd [yourname]`. I have sometimes found that I need to do this step to set up a password each time I launch an instance... even if the AMI I'm launching had a password.
    + Open Remote Desktop Connection (search for mstsc.exe on Windows) & log in to `localhost:[LOCALPORT]`
    + To be able to reconnect to the same desktop, see <http://c-nergy.be/blog/?p=5305> and <https://askubuntu.com/questions/133343/how-do-i-set-up-xrdp-session-that-reuses-an-existing-session>. Basically, the idea is to edit the xrdp ini file to allow this. Run `sudo [gedit/pico/vim] /etc/xrdp/xrdp.ini` and change section `[xrdp1]` where it says `port=-1` to `port=ask-1`. When logging in for the first time, leave the port as `-1` and note the port number you get (will default to `5910`). Then on subsquent logins, change the port to whater the previous one was (I it *should* default to `5910`). Sessions seem to persist even when the SSH tunnel is closed.
- Install the gnome terminal `sudo apt-get install gnome-terminal`, or something better than the `xcfe` terminal. This should swap out automatically if you open a new terminal window
- Install unzip (at least if on Azure): `sudo apt-get install unzip` so that Julia can build `HttpParser` for Atom
- Fix tab-completion by following <https://www.starnet.com/xwin32kb/tab-key-not-working-when-using-xfce-desktop/>
- Install Firefox using `sudo apt-get install firefox`
- Installing Atom
    + Download .deb file from <https://atom.io/>
    + attempt to install with `sudo dpkg -i atom-amd64.deb`
    + After error, run `sudo apt-get install -f`
    + Then again `sudo dpkg -i atom-amd64.deb`
    + Follow <https://github.com/atom/atom/issues/4360#issuecomment-205122828> to get Atom to run. You can find the file with 
        ```bash
        dpkg -L libxcb1 # to find the file
        cd /usr/share/atom
        cp /usr/lib/x86_64-linux-gnu/libxcb.so.1
        sudo sed -i 's/BIG-REQUESTS/_IG-REQUESTS/' libxcb.so.1
        ```
