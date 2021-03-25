# Running VS Code in Podman

Due to recent VS Code updates, the latest VS Code will not run under RHEL7. These notes will allow a RHEL7 users to run Code in a RHEL8 UBI. You'll need to replace the base image with something else if you're running CentOS7 (e.g., centos8/fedora/etc).

## Configure podman for unprivileged use

Enable unprivileged users to build containers and inform podman of the change (this must be performed as root):

```bash
echo "youruser:1000000:1000" >> /etc/subuid
echo "youruser:1000000:1000" >> /etc/subgid
echo "user.max_user_namespaces=10000" >> /etc/sysctl.d/20-podman.conf
sysctl -p /etc/sysctl.d/20-podman.conf
podman system migrate
```

The rest can be performed as your unprivileged user 'youruser'.

## Create the Microsoft VS Code yum repository

You can create the repository manually and copy it into the container, or use the instructions at https://code.visualstudio.com/docs/setup/linux during the container build to provision it the correct way. During testing it was easier for me to create the repository manually:

```bash
cat > microsoft-vscode.repo << EOF
[code]
name=Visual Studio Code
baseurl=https://packages.microsoft.com/yumrepos/vscode
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF
```

## Create the Dockerfile

This is primarily for python development, other dependencies will need to be added according to your needs. Note: libX11-xcb is a dependency which code doesn't pull in for some reason. An issue may need to be opened for it.

```bash
cat > Dockerfile << EOF
FROM registry.access.redhat.com/ubi8/ubi:latest
COPY microsoft-vscode.repo /etc/yum.repos.d/microsoft-vscode.repo
RUN yum -y install code libX11-xcb python2 python2-devel python38 python38-devel && yum clean all
RUN mkdir -p /data/git
CMD ["/usr/bin/code", "--user-data-dir=/root/.config/Code"]
EOF
```

## Build and run the container

```bash
buildah bud -t vs-code .
```

I need to tweak the CMD/ENTRYPOINT directive to get it to launch code properly. Regardless of what options I set the window doesn't persisted unless I run the container interactively and launch code manually.

Replace /data/git with a mountpoint where your existing git repos exist or where you'd like to check things out to so they persist after the container is finished.

```bash
podman run -it --rm -u 0 -e DISPLAY=$DISPLAY \
       -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
       -v /home/youruser/.config/Code:/root/.config/Code:rw \
       -v /home/youruser/.vscode:/root/.vscode:rw \
       -v /data/git:/data/git:rw \
       localhost/vscode bash
```

Then launch code from within the container:

```bash
/usr/bin/code --user-data-dir=/root/.config/Code
```
