# cvat-deploy

- [Mounting cloud storage](#mounting-cloud-storage)
  - [AWS S3 bucket](#aws-s3-bucket-as-filesystem)
    - [Ubuntu 20.04](#aws_s3_ubuntu_2004)
      - [Mount](#aws_s3_mount)
      - [Automatically mount](#aws_s3_automatically_mount)
        - [Using /etc/fstab](#aws_s3_using_fstab)
        - [Using systemd](#aws_s3_using_systemd)
      - [Check](#aws_s3_check)
      - [Unmount](#aws_s3_unmount_filesystem)
- [Quick installation guide](#quick-installation-guide)
  - [Ubuntu 18.04 (x86_64/amd64)](#ubuntu-1804-x86_64amd64)

# Mounting cloud storage
## AWS S3 bucket as filesystem
### <a name="aws_s3_ubuntu_2004">Ubuntu 20.04</a>
#### <a name="aws_s3_mount">Mount</a>

1.  Install s3fs:

    ```bash
    sudo apt install s3fs
    ```

1.  Enter your credentials in a file  `${HOME}/.passwd-s3fs`  and set owner-only permissions:

    ```bash
    echo ACCESS_KEY_ID:SECRET_ACCESS_KEY > ${HOME}/.passwd-s3fs
    chmod 600 ${HOME}/.passwd-s3fs
    ```

1.  Uncomment `user_allow_other` in the `/etc/fuse.conf` file: `sudo nano /etc/fuse.conf`

For more details see [here](https://github.com/s3fs-fuse/s3fs-fuse).

#### <a name="aws_s3_automatically_mount">Automatically mount</a>

##### <a name="aws_s3_using_systemd">Using systemd</a>

1.  Create unit file `sudo nano /etc/systemd/system/s3fs.service`
    (replace `user_name`, `bucket_name`, `mount_point`, `/path/to/.passwd-s3fs`):

    ```bash
    [Unit]
    Description=FUSE filesystem over AWS S3 bucket
    After=network.target

    [Service]
    Environment="MOUNT_POINT=<mount_point>"
    User=<user_name>
    Group=<user_name>
    ExecStart=s3fs <bucket_name> ${MOUNT_POINT} -o passwd_file=/path/to/.passwd-s3fs -o allow_other
    ExecStop=fusermount -u ${MOUNT_POINT}
    Restart=always
    Type=forking

    [Install]
    WantedBy=multi-user.target
    ```

1.  Update the system configurations, enable unit autorun when the system boots, mount the bucket:

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable s3fs.service
    sudo systemctl start s3fs.service
    ```

# Quick installation guide

Before you can use CVAT, you’ll need to get it installed. The document below
contains instructions for the most popular operating systems. If your system is
not covered by the document it should be relatively straight forward to adapt
the instructions below for other systems.

Probably you need to modify the instructions below in case you are behind a proxy
server. Proxy is an advanced topic and it is not covered by the guide.

## Ubuntu 18.04 (x86_64/amd64)

- Open a terminal window. If you don't know how to open a terminal window on
  Ubuntu please read [the answer](https://askubuntu.com/questions/183775/how-do-i-open-a-terminal).

- Type commands below into the terminal window to install `docker`. More
  instructions can be found [here](https://docs.docker.com/install/linux/docker-ce/ubuntu/).

  ```sh
  sudo apt-get update
  sudo apt-get --no-install-recommends install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  sudo add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
  sudo apt-get update
  sudo apt-get --no-install-recommends install -y docker-ce docker-ce-cli containerd.io
  ```

- Perform [post-installation steps](https://docs.docker.com/install/linux/linux-postinstall/)
  to run docker without root permissions.

  ```sh
  sudo groupadd docker
  sudo usermod -aG docker $USER
  ```

  Log out and log back in (or reboot) so that your group membership is
  re-evaluated. You can type `groups` command in a terminal window after
  that and check if `docker` group is in its output.

- Install docker-compose (1.19.0 or newer). Compose is a tool for
  defining and running multi-container docker applications.

  ```bash
  sudo apt-get --no-install-recommends install -y python3-pip python3-setuptools
  sudo python3 -m pip install setuptools docker-compose
  ```

- Clone _CVAT_ source code from the
  [GitHub repository](https://github.com/opencv/cvat).

  ```bash
  sudo apt-get --no-install-recommends install -y git
  git clone https://github.com/opencv/cvat
  cd cvat
  ```

- Build docker images by default. It will take some time to download public
  docker image ubuntu:16.04 and install all necessary ubuntu packages to run
  CVAT server.

  ```bash
  docker-compose build
  ```

- Run docker containers. It will take some time to download public docker
  images like postgres:10.3-alpine, redis:4.0.5-alpine and create containers.

  ```sh
  docker-compose up -d
  ```

- You can register a user but by default it will not have rights even to view
  list of tasks. Thus you should create a superuser. A superuser can use an
  admin panel to assign correct groups to the user. Please use the command
  below:

  ```sh
  docker exec -it cvat bash -ic 'python3 ~/manage.py createsuperuser'
  ```

  Choose a username and a password for your admin account. For more information
  please read [Django documentation](https://docs.djangoproject.com/en/2.2/ref/django-admin/#createsuperuser).

- Google Chrome is the only browser which is supported by CVAT. You need to
  install it as well. Type commands below in a terminal window:

  ```sh
  curl https://dl-ssl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
  sudo sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google-chrome.list'
  sudo apt-get update
  sudo apt-get --no-install-recommends install -y google-chrome-stable
  ```

- Open the installed Google Chrome browser and go to [localhost:8080](http://localhost:8080).
  Type your login/password for the superuser on the login page and press the _Login_
  button. Now you should be able to create a new annotation task. Please read the
  [CVAT user's guide](/cvat/apps/documentation/user_guide.md) for more details.

### Semi-automatic and Automatic Annotation


> **⚠ WARNING: Do not use `docker-compose up`**
>  If you did, make sure all containers are stopped by `docker-compose down`.
- To bring up cvat with auto annotation tool, from cvat root directory, you need to run:
  ```bash
  docker-compose -f docker-compose.yml -f components/serverless/docker-compose.serverless.yml up -d
  ```
  If you did any changes to the docker-compose files, make sure to add `--build` at the end.

  To stop the containers, simply run:

  ```bash
  docker-compose -f docker-compose.yml -f components/serverless/docker-compose.serverless.yml down
  ```

- You have to install `nuctl` command line tool to build and deploy serverless
  functions. Download [version 1.5.8](https://github.com/nuclio/nuclio/releases).
  It is important that the version you download matches the version in
  [docker-compose.serverless.yml](/components/serverless/docker-compose.serverless.yml)
  After downloading the nuclio, give it a proper permission and do a softlink
  ```
  sudo chmod +x nuctl-<version>-linux-amd64
  sudo ln -sf $(pwd)/nuctl-<version>-linux-amd64 /usr/local/bin/nuctl
  ```
