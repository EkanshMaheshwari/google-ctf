# Local Testing Walkthrough

The purpose of this walkthrough is to teach you how to use the kCTF infrastructure.

Following this walkthrough requires a local linux machine capable of running docker.

## Install docker
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER && newgrp docker
```

## Download kCTF
```
sudo apt-get install -y subversion
svn checkout https://github.com/google/google-ctf/trunk/infrastructure/kubernetes
PATH=$PATH:$PWD/kubernetes/bin
```

## Setup nsjail configuration
```
sudo mkdir -p /sys/fs/cgroup/memory/NSJAIL /sys/fs/cgroup/pids/NSJAIL /sys/fs/cgroup/cpu/NSJAIL
sudo chmod o+w /sys/fs/cgroup/*/NSJAIL
```

## Create basic demo challenge
```
kctf-setup-chal-dir $(mktemp -d)
kctf-chal-create test-1
```

## Test connecting to the challenge
This will take a bit longer the first time it is run, as it has to build a chroot.
```
kctf-chal-test-docker test-1
sudo apt-get install -y netcat
nc 127.0.0.1 1337
```

If all went well, you should have a shell inside an nsjail bash, if you didn't, there might be some issues with your system (support for nsjail, docker, or else).


## Debug failures

The instructions below might help you resolve errors in the setup.

### Errors in Windows Subsystem for Linux

Users of WSL1 will need to do testing through a local kubernetes cluster (todo: add instructions).

### Errors installing docker

If you get this error installing docker in Ubuntu:
```
E: Package 'docker-ce' has no installation candidate
```
Try to install Ubuntu's version of docker:
```
sudo apt-get install -y docker.io
```

### Errors running docker

If you get an error like this:
```
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post http://%2Fvar%2Frun%2Fdocker.sock/v1.40/build?buildargs=%7B%7D&cachefrom=%5B%5D&cgroupparent=&cpuperiod=0&cpuquota=0&cpusetcpus=&cpusetmems=&cpushares=0&dockerfile=Dockerfile&labels=%7B%7D&memory=0&memswap=0&networkmode=default&rm=1&session=&shmsize=0&t=kctf-nsjail&target=&ulimits=null&version=1: dial unix /var/run/docker.sock: connect: permission denied
```

Your user doesn't have permission to run docker, to fix this, run:
```
sudo usermod -aG docker $USER && newgrp docker
```

### Errors building the docker images

#### Docker daemon error
In some cases, docker leaves the socket on the FS, but it's no longer running. The error looks like this:
```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

To fix this, just run:
```
sudo service docker restart
```

#### IPv4 forwarding error
In some cases, docker might fail to run because of network errors. Check if you have this error
```
 ---> [Warning] IPv4 forwarding is disabled. Networking will not work.
```
If so, then you have to enable ipv4 forwarding, by running:
```
echo net.ipv4.ip_forward=1 | sudo tee -a /etc/sysctl.conf
```
And since this probably made docker cache an invalid `apt-get update`, you will also have to run `docker system prune -a` before running the `kctf-chal-test-docker` command again.

### Errors connecting to the challenge
If you can't connect, type:
```
docker ps -a
```
In the table, look for the last ran container, and then run:
```
docker logs CONTAINER_NAME
```
replacing CONTAINER_NAME with the name of the last ran container.

That will output the logs of the last time the container ran.

#### cgroup directory not found
if you see errors like:
```
[W][2020-01-31T19:49:47+0000][1] bool cgroup::createCgroup(const string&, pid_t)():43 mkdir('/cgroup/memory/NSJAIL/NSJAIL.10', 0700) failed: No such file or directory
```
That probably means the NSJAIL cgroup directories didn't get created in the nsjail setup step above, try running the [setup nsjail configuration](#setup-nsjail-configuration) step again and then run `kctf-chal-test-docker` again.

#### CLONE error in nsjail

If the error is like this:
```
[E][2020-01-31T20:16:39+0000][1] bool subproc::runChild(nsjconf_t*, int, int, int)():459 nsjail tried to use the CLONE_NEWCGROUP clone flag, which is supported under kernel versions >= 4.6 only. Try disabling this flag: Operation not permitted
[E][2020-01-31T20:16:39+0000][1] bool subproc::runChild(nsjconf_t*, int, int, int)():464 clone(flags=CLONE_NEWNS|CLONE_NEWCGROUP|CLONE_NEWUTS|CLONE_NEWIPC|CLONE_NEWUSER|CLONE_NEWPID|CLONE_NEWNET|SIGCHLD) failed. You probably need root privileges if your system doesn't support CLONE_NEWUSER. Alternatively, you might want to recompile your kernel with support for namespaces or check the current value of the kernel.unprivileged_userns_clone sysctl: Operation not permitted
```
That probably means that unprivileged user namespaces are not enabled, you can fix this by running:
```
(echo 1 | sudo tee /proc/sys/kernel/unprivileged_userns_clone) || (echo 'kernel.unprivileged_userns_clone=1' | sudo tee /etc/sysctl.d/00-local-userns.conf) 2>&1
sudo service procps restart
```
And then try connecting through netcat again.

### Errors inside the challenge
If you see errors like these:
```
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
```

That's normal, just ignore it. You should still get a shell afterwards.