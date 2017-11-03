---
title: Mark's Guide to Amazon EC2 for R and Julia
author: Mark Agerton
date: 2017-06-12
---

This is a guide on how to set up an Amazon EC2 cluster for use with RStudio and/or Julia. There are a few ways to get data on and off the servers in addition to the Dropbox directions provided by <http://www.louisaslett.com/RStudio_AMI/>. Renting EC2 space can be quite cheap, especially if you use a Spot Instance. Pricing is a continuous uniform-price auction, and if you are outbid, your instance is terminated w/out warning.

The guide assumes you have some basic familiarity with using a linux-like terminal (e.g., navigating the file structure, copying, moving, ssh, etc). Note that Windows 10 now includes the "Windows Subsytem for Linux" (WSL), which provides a very nice terminal environment ([MSDN setup guide](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide)).  

If you have suggestions, pull requests & edits are welcome!!

# Steps to get the EC2 running the first time

1. Sign up for Amazon AWS account. Sign up at github.com for an education pack if eligible and you may get some free Amazon AWS credits.
2. Get your SSH keys fixed on your computer so you can log in to your EC2 instances.
    - You may need to add a file called `config` to local `~/.ssh` folder. For example it could be
        ```
        IdentityFile ~/.ssh/github_rsa
        IdentityFile ~/.ssh/Magerton_Key_Pair.pem
        ```
    - Make sure permissions are correct for SSH folder & keys. See [Stackexchange](https://superuser.com/questions/215504/permissions-on-private-key-in-ssh-folder). If using symlinked directory in Windows Subsystem for Linux (WSL) on Windows 10, might need to change permissions to read only on Windows side if can't do this in WSL.
    - What files are:
        + `github_rsa` and `*.pem` are private keys. KEEP SECURE--these are like your password
        + `*.ppk` is PuTTy version of private key ([StackOverflow](https://stackoverflow.com/questions/20367694/whats-the-difference-between-ppk-and-pem-where-pem-is-stored-in-amazons-ec2))
        + `*.pub` files are public keys. These are placed on server-side and used to verify that your private key is correct.
3. Go to <http://www.louisaslett.com/RStudio_AMI/> and click on the AMI you want (first time only to set up)
4. Start the instance in EC2.
    - If permanent setup, use a smaller, cheaper one to get set up. You can save money by using a Spot instance, but could get booted if your maximum willingness to pay (bid) is too low. Otherwise, get as much power as you need.
    - Make sure that you have enough drive storage, and that the HTTP Protocol (port 80) is open in addition to SSH (port 22).
5. SSH into the instance (right click on it & hit "connect" to get the terminal command, should be something like `ssh ubuntu@ec2-54-196-121-83.compute-1.amazonaws.com`). Note that the RStudio AMI has superuser `ubuntu`, not `root` as Connect page suggests.
6. Set up EC2 cluster how you want to and save as an AMI image so you can re-use without having to setup again. See [next section](#ec2-initial-setup).

# EC2 initial setup

1. On the remote, the directory `~/.ssh` has a file `authorized_keys`, which contains the public key for your private (local) `.pem` key. You'll want to add your github private key to this folder, and also deposit `authorized_keys` and your github private key in the user `rstudio` with appropriate permissions
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
    - Copy contents of .ssh to remote user `rstudio` and set up. 
        ```shell
        sudo cp -r .ssh ../rstudio/.ssh
        sudo ls -al ../rstudio/.ssh
        sudo chown -R rstudio:rstudio ../rstudio/.ssh
        sudo chmod 700 ../rstudio/.ssh
        ```
2. Install some programs
    - Add `lftp` to machine to connect to Rice Univ's Box.net accounts.
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
    - Install julia pkg `NLopt` with `Pkg.add("NLopt")` as `ubuntu` requires elevated permissions to install system libraries via Ubuntu package manager.
        + Open Julia prompt by typing `julia`.
        + Run `Pkg.add("NLopt").`
    - Can now log in to `rstudio` account and get git working. Note that it's nicer to use `bash` since it has autocomplete.
        ```shell
        bash            # use bash
        git lfs install # run as rstudio to initiall install (I think)
        ```
3. Set up R/Rstudio
    - Point browser to rstudio server's IP address, for example: `http://ec2-54-196-121-83.compute-1.amazonaws.com/`
    - Update the password in Rstudio to something reasonable by following directions.
    - Add R packages. This will take time because they have to be compiled from source. For example
        ```R
        install.packages(c("data.table", "stringr", "haven", "ggplot2", "zoo", "lubridate", "xts", "Quandl", "ggthemes", "RColorBrewer", "quantmod", "viridis", "sp", "rgdal", "raster", "gridExtra", "ggmap", "mlogit", "grid", "coefplot", "texreg", "lme4", "nlme", "xtable"))
        ```
4. Add Julia packages in bulk
    - Log back in to `rstudio` user via SSH. Open new Julia REPL (run `julia`)
    - Initialize a Package repo: `Pkg.init()`
    - Exit julia `exit()`
    - Add some packages to the `~/.julia/v0.5/REQUIRE` file to install in bulk. (Can use `pico` or `vim` to do this.) Open Julia back up & run `Pkg.resolve()`. Note instructions on installing `NLopt` first as `ubuntu`. See [docs](https://docs.julialang.org/en/stable/manual/packages/#adding-and-removing-packages) re: bulk install
5. Transfer necessary project files over SSH and unzip files:
    - `tar cvz DIRECTORY/ | ssh ubunbtu@ec2ip "cd /wherever && tar xvz"`
    - <http://unix.stackexchange.com/questions/10026/how-can-i-best-copy-large-numbers-of-small-files-over-scp>
    - In reverse, adapt <http://meinit.nl/using-tar-and-ssh-to-efficiently-copy-files-preserving-permissions>
    - ~~From your computer, make a gzipped tar file w/ everything you need to run your program: `tar -cvzf myfile.tar.gz [files/dirs to zip]`~~
    - ~~Use `sftp` to transfer the file.~~
        1. ~~Login to EC2 w/ sftp as you did w/ SSH, replacing `ssh` with `sftp`~~
        2. ~~Once logged in, upload your file with `put myfile.tar.gz`~~
        3. ~~Exit `sftp`~~
    - ~~Unzip your files on Amazon~~
        1. ~~`ssh` into your EC2 account~~
        2. ~~Use `tar -xvzf myfile.tar.gz` to unzip~~
    - `ssh` into your AMI to make sure stuff transferred
6. Alternately, transfer using `lftp` and Box.net. Note that special characters in password may have to be escaped or translated to HTML.
    ```shell
    lftp -p 990 -u "netid@rice.edu,PASSWORD" ftps://ftp.box.com
    mirror [project_dir_on_box]   [remote_project_dir]
    ```
6. Save your instance as an AMI so you can use it again!!
7. Terminate the instance (after verifying the AMI was made) so you can get a bigger, badder, better one.

# Subsequent work using terminal or REPL

1. Launch your AMI from the EC2 console: `AMI > Select on your AMI > Under "Actions," select "Spot Request"` Request a big instance, and set the MAX price you are willing to pay per hour (usually it's much lower than this)
2. Once your AMI is running (can take a bit), SSH into it & get to work or point your browser to the relevant IP address.
3. To save intermediate log-files, you could use a `cron` job to run a script (named like `myscript.sh`, has a shebang w/ your shell path at the top--google it) that would ssh into the server & pipe log file to your home computer. (Might also be something as simple as a command -- no script needed) See <https://www.cyberciti.biz/faq/howto-use-tar-command-through-network-over-ssh-session/>

# Subsequent work connecting local Atom to Remote Julia
You can use local instance of Atom & remote server. The [Docs](http://docs.junolab.org/latest/man/faq.html#Connecting-to-an-external-julia-session-on-a-remote-machine-1) are a tad confusing. Here are (revised) steps. Note that on Windows 10, port-forwarding can be done through the WSL (sweet!)

1. Launch local Atom/Julia session
2. Bring up command palette (`cmd/ctrl-shift-p`) and open `Julia Client: Connect External Process`
3. The local Atom/Juno tells you `[LOCALPORT_NUM]` where Atom/Juno is listening. **The message Julia lets you copy does not need to be run locally, and the port it lists is for the local machine, not the remote's port. You can safely ignore the message.**
4. Set up port forwarding via SSH. Enter the following into local terminal and log in to remote with port forwarding.
    ```shell
    ssh -R [REMOTEPORT_NUM]:localhost:[LOCALPORT_NUM] rstudio@EC2_IP_ADDRESS
    ```
5. Now, on the remote, open a Julia REPL by typing `julia`
6. In the remote Julia REPL, type `using Juno; Juno.connect([REMOTEPORT_NUM])`
7. A local Atom notification will pop up that "Julia is connected"
8. The local Atom/Julia instance is now running on the EC2 cluster. You should be able to see this by evaluating `pwd()` in the local Atom instance. Note that any libraries, files, etc that you load must be on the Remote server now in this session---the remote cannot see local files.


# Azure installation
- SSH into an instance
- Copy local .ssh files to `~/.ssh`. `chmod` the directory to 700 and files to 600.
- Install julia
    + `wget ` the file on <https://julialang.org/downloads/index.html>
    + `tar -xvzf [download name]`
    + Symlink to `/usr/local/bin` by running `sudo ln -s <where you extracted the julia archive>/bin/julia /usr/local/bin/julia`. Note, you'll want to use the FULL path of the directory julia got extracted to (eg, `/home/ME/juliaarchive/bin/julia`)
- Install build tools: `sudo apt-get build-essentials`
- Initialize package repo with `Pkg.init()` in julia
- Bulk install by updating `REQUIRE` in `~/.julia/v0.x/REQUIRE` and running `Pkg.resolve()`
- You are good to go!