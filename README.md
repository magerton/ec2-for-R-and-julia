# Julia and R on Amazon EC2

| Author | Date |
| --- | --- |
| Mark Agerton | 2017-06-12 |
| Arnon Erba | 2018-10-12 |

## Purpose

This guide explains how to set up a simple Amazon Linux 2 instance on EC2 for use with R and Julia. Renting EC2 space can be quite cheap, especially if you use a Spot Instance. Pricing is a continuous uniform-price auction, and if you are outbid, your instance is terminated w/out warning.

The guide assumes basic familiarity with a UNIX-like systems (e.g., navigating the file structure, copying, moving, ssh, etc). Note that Windows 10 now includes the "Windows Subsytem for Linux" (WSL), which provides a very nice terminal environment ([MSDN setup guide](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide)). The Git Bash terminal is also a good choice for Windows. MacOS users may make use of the built-in MacOS terminal.

If you have suggestions, pull requests & edits are welcome!

## Launching an EC2 instance

1. Sign up for an Amazon AWS account. Sign up for a [GitHub education pack](https://education.github.com/pack) if eligible and you may get some free Amazon AWS credits.
2. Spin up an Amazon Linux 2 instance. The "compute optimized" tier is recommended, and you should not need more than 16GB of storage.
3. Create a new SSH key pair when prompted, or choose one already saved in your AWS account. If you create a new pair, you will be asked to download your private key.
    - **Your private key must be kept secure**. By convention it should be placed in your local `~/.ssh` directory and be protected by either `0400` or `0600` permissions [(note)](https://superuser.com/questions/215504/permissions-on-private-key-in-ssh-folder). Your `~/.ssh` directory should have `0700` permissions.
    - Private keys may take several different forms:
        + `*.pem` - Standard file format for cryptographic keys/certificates. AWS uses this format.
        + `*.key` - Alternate file extension for a PEM file only containing a private key.
        + `*.ppk` - Proprietary PuTTY format for private keys. PuTTY does not support the PEM format.
    - Public keys may utilize the `*.pub` extension, but when copied to a server are concatenated onto your remote `~/.ssh/authorized_keys` file. The presence of your public key in this **remote** file grants you SSH access to the server.
4. Connect to your EC2 instance via SSH.
    - Method 1:
        + `ssh ec2-user@ip-address-or-hostname -i ~/.ssh/privatekey.pem`
    - Method 2:
        + Create a `~/.ssh/config` file with the following format:
        ```
        Host my-ec2-server
            HostName ip-address-or-hostname
            User ec2-user
            IdentityFile ~/.ssh/privatekey.pem
        ```
        + `ssh my-ec2-server`

# Software installation

1. On the remote, the directory `~/.ssh` has a file `authorized_keys`, which contains the public key counterpart for your private (local) `.pem` key. You'll want to add a github private key to this folder, and also deposit `authorized_keys` and your github private key in other uses (such as `rstudio`) with appropriate permissions. `ssh-copy-id` might be a better option to put a public key in another user's folder.
    - On the local machine, navigate to your directory w/ relevant keys (usually `~/.ssh` or `%USERPROFILE%/.ssh`).
    - Use `sftp` to put your `github_rsa` private key (and possibly also `config`) on the remote server
    - Exit `sftp`, and then `ssh` into the remote
    - Move the private key into .ssh: `mv github_rsa .ssh/`
    - Check that the permissions are correct: `ls -al .ssh`. See [stackexchange](https://unix.stackexchange.com/questions/210228/add-a-user-wthout-password-but-with-ssh-and-public-key).
    - You may need to make a new config file:
        ```shell
        echo IdentityFile ~/.ssh/github_rsa > .ssh/config
        chown ubuntu .ssh/config
        chmod 700 .ssh/config
        ```
    - If you need to access git or other services, copy over .ssh files to `~/.ssh`. `chmod` the directory to 700 and files to 600.
<!--     - Copy contents of .ssh to remote user `rstudio` and set up. 
        ```shell
        sudo cp -r .ssh ../rstudio/.ssh
        sudo ls -al ../rstudio/.ssh
        sudo chown -R rstudio:rstudio ../rstudio/.ssh
        sudo chmod 700 ../rstudio/.ssh
        ```--->
2. Install needed programs
    - Add `lftp` to machine to connect to Box.net accounts.
        ```shell
        sudo apt-get update
        sudo apt-get install lftp
        ```
    - Add `git-lfs`.
        + See [install guide](https://github.com/git-lfs/git-lfs#getting-started). I had to use [PackageCloud](https://packagecloud.io/github/git-lfs/install) to install from command line.
        ```shell
        curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash
        sudo apt-get install git-lfs
        git lfs install  # only run once for initial install
        ```

# Julia setup

- Install julia
    + `wget ` the file on <https://julialang.org/downloads/index.html>
    + `tar -xvzf [download name]`
    + Symlink to `/usr/local/bin` by running `sudo ln -s <where you extracted the julia archive>/bin/julia /usr/local/bin/julia`. Note, you'll want to use the FULL path of the directory julia got extracted to (eg, `/home/ME/juliaarchive/bin/julia`)
- Update pkgs `sudo apt-get update`
- Install build tools: `sudo apt-get install build-essential`
- Install `sudo apt-get install hdf5-tools` from command line
- Open up a julia prompt and install packages into the default folder

    ```bash
    ]add AxisAlgorithms BenchmarkTools Calculus CategoricalArrays DataFrames Distributions FileIO Formatting GLM GR Gadfly IndirectArrays Interpolations JLD2 MixedModels NLSolversBase NLopt Optim PkgDev Plots Primes Profile ProgressMeter PyPlot RData Ratios StatsBase StatsFuns StatsModels

    dev ssh://git@github.com/magerton/CountPlus.git
    dev ssh://git@github.com/magerton/Halton.git
    dev ssh://git@github.com/magerton/JuliaTex.jl.git
    dev ssh://git@github.com/magerton/GenGlobal.jl.git
    dev ssh://git@github.com/magerton/MarksRandomEffects.git
    dev ssh://git@github.com/magerton/OrderedResponse.jl.git
    dev ssh://git@github.com/magerton/MarkovTransitionMatrices.jl.git
    dev ssh://git@github.com/magerton/ShaleDrillingModel.jl.git
    dev ssh://git@github.com/magerton/ShaleDrillingData.jl.git
    dev ssh://git@github.com/magerton/ShaleDrillingEstimation.jl.git
    dev ssh://git@github.com/magerton/ShaleDrillingPostEstimation.jl.git
    ```
- ~~Initialize package repo with `Pkg.init()` in julia~~
- ~~Bulk install by updating `REQUIRE` in `~/.julia/v0.x/REQUIRE` and running `Pkg.resolve()`. You may need to run julia as `sudo` with elevated priveleges, but hopefully not.~~

# Setting up GUI/ remote desktop (RDP) for remote machine via secure SSH tunnel

- Remote Desktop
    + Install `xrdp` and `xcfe4` software as per <https://docs.microsoft.com/en-us/azure/virtual-machines/linux/use-remote-desktop>. We'll connect over SSH, so no need to open a special RDP port.
    + On local machine, create ssh tunnel to remote with port forwarding `ssh -L [LOCALPORT]:localhost:[3389] username@remoteip`. Can do this with terminal, Bash for Windows, or Putty.
    + After connecting to remote instance, set a password on the remote machine so that the RDP can log in `sudo setpasswd [yourname]`
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

# Subsequent work using terminal or REPL

1. Launch your AMI from the EC2 console: `AMI > Select on your AMI > Under "Actions," select "Spot Request"` Request a big instance, and set the MAX price you are willing to pay per hour (usually it's much lower than this)
2. Once your AMI is running (can take a bit), SSH into it & get to work or point your browser to the relevant IP address.
3. To save intermediate log-files, you could use a `cron` job to run a script (named like `myscript.sh`, has a shebang w/ your shell path at the top--google it) that would ssh into the server & pipe log file to your home computer. (Might also be something as simple as a command -- no script needed) See <https://www.cyberciti.biz/faq/howto-use-tar-command-through-network-over-ssh-session/>

# Using LFTP

- Alternately, transfer using `lftp` and Box.net. Note that special characters in password may have to be escaped or translated to HTML.
    ```shell
    lftp -p 990 -u "netid@rice.edu,PASSWORD" ftps://ftp.box.com
    mirror [project_dir_on_box]   [remote_project_dir]
    ```

# X11 forwarding and OS X

The above instructions are for Windows. Can try using X11 on a Mac or Linux box

- Install `xpra` locally on OS X using Homebrew. See <https://github.com/google/homebrew-xpra>
- Install `xpra` and `winswitch` on remote by following <http://winswitch.org/downloads/debian-repository.html?dist_select=xenial>. After installing `winswitch` as per direcions, run `apt-get install xpra` to install `xpra` and run `exit` to exit root.

# Using EFS

- Add an EFS instance (encrypted?)
- Install `amazon-efs-utils` (manually?) using <https://docs.aws.amazon.com/efs/latest/ug/using-amazon-efs-utils.html#installing-other-distro>
- `apt-get install libssl-dev`
- Upgrade `stunnel` and symlinmk it `sudo ln -s /usr/local/bin/stunnel /bin/stunnel` ~~and/or `sudo ln -s /usr/bin/stunnel /bin/stunnel`~~
- **Make sure that EFS and EC2 instances are in the same security group, and that the SG has an inbound rule allowing NFS traffic from the same SG** (currently this is group 4)
- `sudo mount -t efs FILESYSTEMID:/ efs`
- add line to `/etc/fstab`: `fs-59778a40:/ /mnt/efs efs defaults,_netdev,nofail 0 0`

# Notifications when usage falls

Sometimes things error out... To be notified when CPU utilization falls,

- Create a subscription at [AWS SNS](https://console.aws.amazon.com/sns/v2/home)
- Under EC2 instances > Monitoring, click "create alarm". Set alarm for CPU utilization <= 50 pct for less than 5 min.

# Using nohup

```bash
nohup julia --optimize=3 ~/dev-pkgs/ShaleDrillingEstimation/example/run_estimator.jl > ~/efs-ubuntu/JOBNAME\ "$(TZ=America/Los_Angeles date +on\ %Y-%m-%d\ at\ %Hh%Mm)"\ by\ ${MYIP}.out 2>&1 &
```
