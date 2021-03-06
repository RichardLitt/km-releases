# Kontain Release

Version: 0.10-beta
Document Status: WORK IN PROGRESS

**Copyright © 2020 Kontain Inc. All rights reserved.**

**By downloading or otherwise accessing the Kontain Beta Materials, you hereby agree to all of the terms and conditions of Kontain’s Beta License available at https://raw.githubusercontent.com/kontainapp/km-releases/master/LICENSE**

## Introduction

Kontain provides a mechanism to create a unikernel from an unmodified application code, and execute the unikernel it in a dedicated VM.
Kontain provides VM level isolation guarantees for security with very low overhead compare to regular linux processes.
For example, payloads run under Kontain are immune to Meltdown security flaw even on un-patched kernels and CPUs.

Kontain consists of two components - Virtual Machine runtime, and tools to build unikernels.
Together they provide VM-based sandboxing to run payloads.

Kontain Monitor (KM) is a host user-space process providing a single VM to a single application.
Kontain VM model dynamically adjusts to application requirements - e.g, automatically grow/shrink as application need more / less memory, or automatic vCPU add or remove as application manipulates thread pools.

Kontain does not require changes to the application source code in order to run the app as a unikernel.
Any program can be converted to unikernel and run in a dedicated VM.
If application issues fork/exec call, it is supported by spawning additional KM process to manage dedicated VMs.

* A payload is run as a unikernel in a dedicated VM - i.e. directly on virtual hardware,
  without additional software layer between the app the the virtual machine
* Kontain VM is a specifically optimized VM Model
  While allowing the payload to use full user space instruction set of the host CPU,
  it provides only the features necessary for the unikernel to execute
  * CPU lacks features normally available only with high privileges, such as MMU and virtual memory.
    VM presents linear address space and single low privilege level to the program running inside,
    plus a small landing pad for handing traps and signals.
  * The VM does not have any virtual peripherals or even virtual buses either,
    instead it uses a small number of hypercalls to the KM to interact with the outside world
  * The VM provides neither BIOS nor support for bootstrap.
    Instead is has a facility to pre-initialize memory using the binary artifact before passing execution to it.
    It is similar to embedded CPU that has pre-initialized PROM chip with the executable code

In Containers universe, Kontain provides an OCI-compatible runtime for seamless integration (Note: some functionality may be missing in this Beta release).

### Virtual Machine Prerequisites

The Linux kernel that Kontain runs on must have a Kontain supported virtual machine kernel module installed in order for Kontain to work. Currently regular Linux `KVM` module and Kontain proprietary `KKM` module are supported by Kontain.

The `KVM` module is available on most Linux kernels. Kontain requies Linux Kernel 5.x to properly function due to some changes and improvements in 5.x KVM Module.

Some cloud service providers, AWS in particular, do not supported nested virtualization with `KVM`. For these cases, the Kontain proprietary `KKM` module is used.

To make it easie to try Kontain on AWS, Kontain provides a pre-built AMI to experiment with KKM (Ubuntu 20 with KKM preinstalled). See below in `Amazon - pre-built AMI` section

Kontain manipulates VMs and needs acess to either `/dev/kvm` device, or `/dev/kkm` device , depending on the kernel module used.

## Install

Kontain releases are maintained on a public git repo https://github.com/kontainapp/km-releases, with README.md giving basic download and install instruction.

### Install on a machine with KVM enabled

Assuming the Linux kernel is 5.0 and above (check `uname -a`), and kvm module is loaded (check `lsmod | grep kvm`) and thus `/dev/kvm` file exist,  you can install and use Kontain support for building and running unikernels.

For the impatient, `wget -q -O - https://raw.githubusercontent.com/kontainapp/km-releases/master/kontain-install.sh | bash`.

The script validates prerequisites and places necessary files into `/opt/kontain` directory,
then tries to run "Hello, World!" unikernel.

Test the installation:

```bash
$ /opt/kontain/bin/km /opt/kontain/tests/hello_test.km Hello World
Hello, world
Hello, argv[0] = '/opt/kontain/tests/hello_test.km'
Hello, argv[1] = 'Hello'
Hello, argv[2] = 'World'
```

### Create and run your first unikernel

Create an example program:

#### Example Program

Create playground directory and example program:

```C
dir=$(mktemp -d)
file=kontain-example
cat <<EOF > $dir/$file.c

#include <stdio.h>
#include <sys/utsname.h>

int main(int argc, char* argv[])
{
   struct utsname name;

   printf("Hello World from the following runtime environment: \n");
   if (uname(&name) >= 0) {
      printf("sysname \t= %s\nnodename \t= %s\nrelease \t= %s\nversion \t= %s\nmachine \t= %s\n",
             name.sysname,
             name.nodename,
             name.release,
             name.version,
             name.machine);
   }
   return 0;
}
EOF
```

Assuming you have gcc installed, compile and link the code into Kontain unikernel:

```bash
gcc -c -o $dir/$file.o $dir/$file.c
/opt/kontain/bin/kontain-gcc -o $dir/$file.km $dir/$file.o
```

Assuming access to /dev/kvm or /dev/kkm run it the unikernel:

```bash
/opt/kontain/bin/km $dir/$file.km
```

Note that `.km` is the ELF file with Kontain unikernel, and the last command was executed in Kontain VM.

To get help for KM command line, `/opt/kontain/bin/km --help`

### Install for Docker

You can run Kontain payload wrapped in a native Docker container.... in this case, all Docker overhead is still going to be around, but you will have VM / unikernel without extra overhead. **No additional installation (other than regular `docker` or `moby` or `podman`) is required**

#### Kontain OCI runtime (krun)

**** KRUN is not in the bundle yet, skip this section ****

Or you can use Kontain `krun` runtime from docker/podman or directly. `krun` is installed together with KM and other components.
`krun` is forked from Redhat's `crun` runtime, and can be invoked as `krun` or as `crun --kontain`

Configuring `krun`: Documentation here is TBD

* Runtimes config
*  OCI spec and use runtimes for bringing in existing kms. Update doc on how to use then and what needs to be in. Add an example

```
  {
  "default-runtime": "runc",
  "runtimes": {
    "crun": {
      "path": "/opt/kontain/bin/crun",
      "runtimeArgs": [
              "--kontain"
      ]
    }
  }
  ```

#### Validate

To validate with regular runtime, and also to build/run the first container with Kontain Unikernel, build a simple docker container.
Assuming you validated the install so the `file` and `dir` vars , and the actual files are still around:

```bash
cat <<EOF | docker build -t kontain-hello $dir -f -
FROM scratch
COPY $file.km /
ENTRYPOINT [ "/opt/kontain/bin/km"]
CMD [ "/$file.km" ]
EOF

docker run --rm -v /opt/kontain/bin/km:/opt/kontain/bin/km:z --device /dev/kvm kontain-hello
```

To validate with `krun` (when we add `krun` to the bundle) `docker run --rm --runtime=kontain kontain-hello`

### Kubernetes

In order to run Kontainerized payloads in Kontain VM on Kubernetes, each
Kubernetes node needs to have KM installed, and we also need KVM Device
plugin which allows the non-privileged pods access to /dev/kvm

All this is accomplished by deploying KontainD Daemon Set: KontainD serves
both as an installer for kontain and a device manager for kvm/kkm devices on
a k8s cluster. To use kontain monitor within containers in k8s, we need to
first install kontain monitor onto the node to which we want to deploy
kontainers. Then we need a device manager for `kvm` devices so k8s knows how
to schedule workloads onto the right node. For this, we developed kontaind, a
device manager for `/dev/kvm` or `/dev/kkm` device, running as a daemonset on
nodes that has these devices available.

To deploy the latest version of `kontaind`, run:

```bash
kubectl apply \
  -f https://github.com/kontainapp/km-releases/blob/master/k8s/kontaind/deployment.yaml?raw=true
```

Alternatively, you can download the yaml and make your own modification:

``` bash
wget https://github.com/kontainapp/km-releases/blob/master/k8s/kontaind/deployment.yaml?raw=true -O kontaind.yaml
```

To verify:
```bash
kubectl get daemonsets
```
should show `kontaind` number of DESIRED equal to number of READY:
```bash
NAME               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kontaind           1         1         1       1            1           <none>          84d
```


## Getting started

### Building your own Kontain unikernel

To build a Kontain unikernel, you can do one of the following

* Build or use a regular no-libc executable, e.g. GOLANG
* Link existing object files into a musl-libc based executable (like you would do for Alpine containers)
* Link existing object files into a Kontain runtime-based executable

We recommend statically linked executable (dynamic linking is supported but requires payload access to shared libraries,
so we will leave it outside of this short intro).

Kontain provides a convenience wrappers (`kontain-gcc` and `kontain-g++`) to simplify linking - these tools are just shell wrapper to provide all proper flags to gcc/g++/ld. You can use them as a replacement for gcc/g++ during a product build, or just for linking.

#### GOLANG

Unmodified statically linked GO executable can be run as a unikernel in Kontain VM, as a unikernel.
Dynamically linked executable usually use glibc - we are currently working on supporting it; an attempt to run such an executable
as a unikernel in Kontain VM will cause `Unimplemented hypercall 273 (set_robust_list)` error and payload core dump.

Here is a simple go program to prints "Hello World" and machine / sysname:

```go
#dir=$(mktemp -d)   # reuse dir from the prior example
file=kontain-example-go
cat <<EOF > $dir/$file.go
package main

import (
  "fmt"
  "syscall"
)

func charsToString(ca []int8) string {
  s := make([]byte, len(ca))
  var lens int
  for ; lens < len(ca); lens++ {
     if ca[lens] == 0 {
     break
     }
   s[lens] = uint8(ca[lens])
   }
   return string(s[0:lens])
}

func main() {
    fmt.Println("Hello world !")
    utsname := syscall.Utsname{}
    syscall.Uname(&utsname)
    fmt.Printf("Machine=%s\n", charsToString(utsname.Machine[:]))
    fmt.Printf("Sysname=%s\n", charsToString(utsname.Sysname[:]))
}
EOF

go build -o $dir/$file.km $dir/$file.go

/opt/kontain/bin/km $dir/$file.km
```

For more optimal unikernel additional linker options:

```bash
go build -ldflags '-T 0x201000 -extldflags "-no-pie -static -Wl,--gc-sections"' -o test.km test.go
```

### Using with standard executable

Kontain supports running not-modified Linux executable as unikernel in Kontain VM, provided:

* The executable does not use GLIBC (glibc is not supported yet).
  * So the executables could be one build for Alpine, or one not using libc (e.g. GOLANG)

Build the musl-based executable with kontain-gcc, using the examples/vars above:

```bash
/opt/kontain/bin/kontain-gcc -alpine -o $dir/$file $dir/$file.o
```

And validate the result running as unikernel in Kontain VM and print out something like this:

```sh
/opt/kontain/bin/km $dir/$file
```

The result:

```sh
Hello World from the following runtime environment:
sysname         = kontain-runtime
nodename        = node-p330
release         = 4.1
version         = preview
machine         = kontain_VM
```

## _What about the static .km files?_

### Using pre-build unikernels for interpreted languages

Kontain supports a set of pre-built language systems, as containers you can use in FROM dockerfiles statement, or passing scripts dirs as volumes.

The images are available on dockerhub as `kontainapp/runenv-<language>-version` e.g. `kontainapp/runenv-python-3.7`. Kontain provides the following pre-built languages:

* jdk-11.0.8
* node-12.4 (js)
* python-3.7

We will extend the list by the release time. Also - see the section below on linking your own, if needed

Example:  You can run a interactive python as inside Kontain VM using this:

`docker run --device /dev/kvm -it --rm -v /opt/kontain/bin/km:/opt/kontain/bin/km:z kontainapp/runenv-python-3.7`

**NOTE** Currently you may see debug messages there, and the container size is not optimized yet.

### Using snapshots

To speedup startup time of an application with a long warm-up time Kontain provides a mechanism to create a "snapshot" unikernel that represents the state of the application.
A snapshot is an ELF file which can be run as directly as a unikernel in Kontain VM, but captures payload state-in time.

Snapshot can be triggered by an API called from the payload, or by an external `km_cli` running on host and communicating to Kontain Monitor (KM)

Limitations:

* no coordination of multi-process payload snapshots

TODO:
 * Doc and example

### Using with Kubernetes

After installation of kontaind is completed (see above), you can use API server/kubectl to deploy Kontain-based pods to a Kubernetes cluster.
A spec for the pod needs to request /dev/kvm device in the `resources` section, host volume mount,
and _entrypoint calling KM (unless symlinks are set up - see below)._??? I'd say make this default.

Example:

```yaml
  containers:
  ...
      resources:
         requests:
            devices.kubevirt.io/kvm: "1"
      command: ["/opt/kontain/bin/km", "/dweb/dweb.km", "8080"]
      volumeMounts:
        - name: kontain-monitor
          mountPath: /opt/kontain
   volumes:
      - name: kontain-monitor
        hostPath:
            path: /opt/kontain
```

## Debugging

Kontain supports debugging payloads (unikernels) via core dumps and `gdb`, as well as live debugging it `gdb`.
That also covers gdb-based GUI - e.g. Visual Studio Code `Debug`.

### Core dumps

Payload running as a unikernel in Kontain VM will generate a coredump in the same cases it would have generated if running on Linux. The file name is `kmcore` (can be changed with `--coredump=file` flag).
You can analyze the payload coredump as a regular Linux coredump, e.g. `gdb program.km kmcore`

### Live debugging - command line

Pass `-g` flag to km to start in "GDB" mode, and it will print `gdb` command to run to start debugging session right away.
 e.g. :

```
/opt/kontain/bin/km -g ./cpython/python.km
./cpython/python.km: Waiting for a debugger. Connect to it like this:
        gdb --ex="target remote localhost:2159" ./cpython/python.km
GdbServerStubStarted
```

If you want to run payload and connect gdb to it in-flight, pass `-gdb-listen` to KM.

**TODO**: more documentation and examples here.

### Visual Studio Code

VS code supports GUI debugging via GDB. `launch.json` needs to have configuration for Kontain payload debugging.... here is an example:

```json
      {
         "name": "Debugging of a Kontain unikernel exec_test.km",
         "type": "cppdbg",
         "request": "launch",
         "program": "/opt/kontain/bin/km",
         "args": [
            "exec_test.km",
            "-f"
         ],
         "stopAtEntry": true,
         "cwd": "${workspaceFolder}/your_dir",
         "environment": [],
         "externalConsole": false,
         "MIMode": "gdb",
         "setupCommands": [
            {
               "description": "Enable pretty-printing for gdb",
               "text": "-enable-pretty-printing",
               "ignoreFailures": true
            }
         ]
      },
```

See VS Code launch.json docs for more info

## Symlinks

## _Do we need this? I was thinking we use this mechanism in faktory or prepared base images, but don't expect users to do it directly._

If Kontain Monitor (KM) is invoked via a symlink,
it assumes the original file was side by side with the unikernel ELF and runs this elf.
The name of the elf is the \<original_file_name>.km

For example, if you have the following files in your directory:

```
python -> opt/kontain/bin/km
python.km
```

The running `./python` will result in running `/opt/kontain/bin/km ./python.km`. This allows to keep all original scripts and shebang files using Kontain unmodified.

**TODO**: example of Python virtenv and django using this

## Faktory

Faktory converts a docker container to kontain kontainer. For reference,
`container` will refer to docker container and `kontainer` with a `k` will
refer to kontain kontainer.

### Install Faktory

TODO: fix the download link.
```bash
curl -sSL <download link TBD> | bash -s
```

### Java Example

In this section, we will use a container build with JDK base image to
illustrate how `faktory` works.

#### Download kontain Java runtime environment

Download the kontain JDK 11 image:

```bash
docker pull kontainapp/runenv-jdk-11:latest
```

#### Using faktory to convert an existing java based image

To convert an existing image `example/existing-java:latest` into a kontain
based image named `example/kontain-java:latest`:

```bash
# sudo may be required since faktory needs to look at files owned by dockerd
# and containerd, which is owned by root under `/var/lib/docker`
sudo faktory convert \
    example/existing-java:latest \
    example/kontain-java:latest \
    kontainapp/runenv-jdk-11:latest \
    --type java
```

### Use kontain Java in dockerfiles

To use kontain java runtime environment with dockerfile, user can substitute
the base image with kontain image.
```dockerfile
FROM kontainapp/runenv-jdk-11

# rest of dockerfile remain the same ...
```

Here is an example dockerfile without kontain, to build and package the
`springboot` starter `gs-rest-service` repo from `spring-guides` found
[here](https://github.com/spring-guides/gs-rest-service.git). We use
`adoptopenjdk/openjdk11:alpine` and `adoptopenjdk/openjdk11:alpine-jre` as
base image as an example, but any java base image would work.

```dockerfile
FROM adoptopenjdk/openjdk11:alpine AS builder
COPY gs-rest-service/complete /app
WORKDIR /app
RUN ./mvnw install

FROM adoptopenjdk/openjdk11:alpine-jre
WORKDIR /app
ARG APPJAR=/app/target/*.jar
COPY --from=builder ${APPJAR} app.jar
ENTRYPOINT ["java","-jar", "app.jar"]
```

To package the same container using kontain:

```dockerfile
FROM adoptopenjdk/openjdk11:alpine AS builder
COPY gs-rest-service/complete /app
WORKDIR /app
RUN ./mvnw install

FROM kontainapp/runenv-jdk-11
WORKDIR /app
ARG APPJAR=/app/target/*.jar
COPY --from=builder ${APPJAR} app.jar
ENTRYPOINT ["java","-jar", "app.jar"]
```

Note: only the `FROM` from the final docker image is changed. Here we kept
using a normal jdk docker image as the `builder` because the build
environment is not affected by kontain.

### Run

To run a kontain based container image, the container will need access to
`/dev/kvm` and kontain monitor `km`, so make sure the host has these
available. For Java we also require kontain's version of `libc.so`. E.g.

```bash
docker run -it --rm \
    --device /dev/kvm \
    -v /opt/kontain/bin/km:/opt/kontain/bin/km:z \
    -v /opt/kontain/runtime/libc.so:/opt/kontain/runtime/libc.so:z \
    example/kontain-java
```

Note: we are working on OCI Runtime that will automate the above. This document will be updated once it's available

## Architecture

TODO

### Kontain Monitor (VM hardware model) and Kontain unikernels

TODO. Outline:

Link to google architecture doc (need to make it public after review https://docs.google.com/document/d/1UckXwTfVgcJ5g4hvz20yJf51bDkTJs4YIwuRzFjxrXQ/edit#heading=h.z80ov6pz80hk)

System design diagram
Hardware model - threads, memory, IO
Virtualization levels (in-KM, outside of KM)
Pillars (no memory mgmt, delegation to host, etc)
KM code components

Supported syscalls - philosophy
Sandboxing vi OUT/hcalls
Sandboxing via syscall intercept
Supported syscalls and delegation to host
  relations to seccomp

### Solution for no-nested-virtualization machines

When nested virtualization is not available, Kontain provides a Kontain Kernel module (`kkm`) that implements a subset of KVM ioctls. It does not reuse KVM code or algorithms, because the requiements are much simpler - but it does implement a subset if `KVM ioctls` and uses the same control paradigm. It communicates via special device `/dev/kkm`.

TODO: KKM architecture (high level) goes here

`kkm` requires to be built with the same kernel version and same kernel config as the running system. We are working on proper installation.
TODO: doc on installation-with-build, when installation is ready

#### Amazon - pre-built AMI

Meanwhile, we provide an AWS image (Ubuntu 20 with pre-installed and pre-loaded KKM) which can be used to experiment / test Kontain on AWS.

AMI is placed in N. California (us-west-1) region. AMI ID is `ami-047e551d80c79dbb7`.

To create a VM:

```
aws ec2 create-key-pair --key-name aws-kkm --region us-west-1
aws ec2 run-instances --image-id ami-047e551d80c79dbb7 --count 1 --instance-type t2.micro --region us-west-1 --key-name aws-kkm
# before next step save the key to ~/.ssh/aws-kkm.pem and chown it to 400
```

You can then ssh to the VM , install the latest Kontain per instructions above and run it.
The only difference- please use `/dev/kkm` instead of `/dev/kvm`

### Kontainers

Content TODO. Below is an outline. This is mainly explanation/intro, no specific steps.

Regular runtime - mount, namespaces, using snapshots. Pluses and minuses

krun - why and how to use

* OCI spec and use runtimes for bringing in existing kms. Update doc on how to use then and what needs to be in. Add an example
```
  {
  "default-runtime": "runc",
  "runtimes": {
    "crun": {
      "path": "/usr/local/bin/crun",
      "runtimeArgs": [
              "--kontain"
      ]
    }
  }
  ```

### Kubernetes

#### kontaind

kontaind serves both as an installer for kontain and a device manager for
kvm/kkm devices on a k8s cluster. To use kontain monitor within containers in
k8s, we need to first install kontain monitor onto the node to which we want
to deploy kontainers. Then we need a device manager for `kvm` devices so k8s
knows how to schedule workloads onto the right node. For this, we developed
kontaind, a device manager for `/dev/kvm` or `/dev/kkm` device, running as a daemonset on nodes that has these devices available.

To deploy the latest version of `kontaind`, run:

```bash
kubectl apply \
  -f https://github.com/kontainapp/km-releases/blob/master/k8s/kontaind/deployment.yaml?raw=true
```

Alternatively, you can download the yaml and make your own modification:

``` bash
wget https://github.com/kontainapp/km-releases/blob/master/k8s/kontaind/deployment.yaml?raw=true -O kontaind.yaml
```

## Dev guide

Kontain unikernel can be created by replacing 'gcc' (or g++) on the final link phase of C/C++ program with /opt/kontain/bin/kontain-gcc (or g++).
Kontain Monitor (KM) can also run some unmodified Linux binaries as unikernels

TODO: details

* Example of building complex unikernels (e.g python)
* Running alpine (musl-based) or regular (glibc) based executable as-is as a unikernel

## Cloud

### Azure

Azure supports nested virtualization for some instances size since 2017: https://azure.microsoft.com/en-us/blog/nested-virtualization-in-azure/.
Kontain CI/CD process uses `Standard_D4s_v3` instance size.

Create one of these instances, ssh to it , then install and try Kontain as described above.

For example, assuming you have Azure CLI installed and you are logged in Azure (you may want to replace username/password and /or use ssh keys), this will create a correct VM

```bash
az group create --name myResourceGroup --location westus
az vm create --resource-group myResourceGroup --name kontain-demo --image Canonical:UbuntuServer:18.04-LTS:latest --size Standard_D4s_v3 --admin-username kontain --admin-password KontainDemo1-now?
```

Note: Kontain runs it's own CI/CD pipeline on Azure Managed Kubernetes, and AWS for no-nested-virtualization code.

### AWS

For AWS, Kontain can run on `*.metal` instances. For virtual instances, Kontain provides `kkm` kernel module and pre-build AMI with Ubuntu20.
Unfortunately AWS does not support nested virtualization  on non-metal instances, so a kernel modules is required. AMI info and example of AWS commands are documented earlier in this paper.

### Google Cloud Platform

For GCP, nested virtualization support and configuraration are described here: https://docs.google.com/document/d/1UckXwTfVgcJ5g4hvz20yJf51bDkTJs4YIwuRzFjxrXQ/edit#heading=h.z80ov6pz80hk.

Also, default Linux distro GCP creates is an older Debian with 4.9 Kernel. Kontain requires version 5.x.
Please choose either Ubuntu 18 or Ububtu 20, or Debian 10 when creating a VM

### Other clouds

Kontain works on other clouds  in a Linux VM with (1) nested virtualization enabled (2) Linux kernel 5.x

## FAQ

### Is it OSS?

These are binary-only releases, Kontain code is currently not open sourced and is maintained in the [private repo](https://github.com/kontainapp/km)


### How to install KKM

We are working on install process for KKM - it requires building with the exact kernel header version and is being worked on for the release.

### How to see Kontain interaction with KVM

* You can use `trace-cmd record` and `trace-cmd report` to observe kvm activity. [For more details on trace-cmd see](https://www.linux-kvm.org/page/Tracing).

### Are there any limitations on code running as unikernel

* The code shouldn't use not supported system calls
* VM monitor will report an error as "Unimplemented hypercall". Payload will abort
* Apps like Java or Python or Node are good with this requirement

======= END OF THE DOC ====
