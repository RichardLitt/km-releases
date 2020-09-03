# Kontain Release

Version: 0.9-Beta
Document Status: placeholder

/* Copyright © 2020 Kontain Inc. All rights reserved. */

## Introduction

Kontain provides a way to run unmodified or relinked Linux executable as a unikernel in a dedicated VM  - with VM level isolation guarantees for security, and delivers very low overhead compare to regular linux processes. For example, payloads run under Kontain are immune to Meltdown security flaw even on un-patched kernels and CPUs.

While providing VM-level isolation, Kontain requires container-level (or less) overhead. In many cases Kontain can accelerate start time and smoother burst costs.

Kontain VM model dynamically adjusts to application requirement - e.g, automatic grow/shrink as application need more / less memory, or automatic vCPU add or remove as application manipulates thread pools.

Kontain consists of library based unikernel and user-space Kontain Monitor providing application with VM model. Together they provide VM-based sandboxing to run payloads.

* Kontain Monitor (KM) is a host user-space process providing a single VM to a single application.
  * If application issues fork/exec call, it is supported by spawning additional KM process to manage dedicated VMs.
* A payload is run as a unikernel - i.e. directly on virtual hardware, without  additional software layer between the app the the virtual machine. The payload  may be the original executable, untouched (see below for more details) or original object files re-linked with Kontain libraries.
  * Kontain does not require changes to the application source code in order to run the app as a unikernel.
* Kontain VM is a specifically optimized VM Model. While allowing the executable to use full user space instruction set of the host CPU, it provides only the features necessary for the unikernel to execute.
  * CPU lacks features normally available only with high privileges, such as MMU and virtual memory. VM presents linear address space and single low privilege level to the program running inside, plus a small high privilege landing pad for handing traps and signals.
  * The VM does not have any virtual peripherals or even virtual buses either, instead it uses a small number of hypercalls to the KM to interact with the outside world.
  * The provides neither BIOS nor support for bootstrap. Instead is has a facility to pre-initialize memory using the binary artifact before passing execution to it. It is similar to embedded CPU that has pre-initialized PROM chip with the executable code.

In Containers universe, Kontain provides an OCI-compatible runtime for seamless integration (Note: some functionality may be missing on this Beta release).

## Install

Kontain releases are maintained on a public git repo https://github.com/kontainapp/km-releases, with README.md giving basic download and install instruction. These are binary-only releases, Kontain code is currently not open sourced and is maintained in the [private repo](https://github.com/kontainapp/km)

### Pre-requisites

Kontain manipulates VMs and needs either `KVM module and /dev/kvm device`, or Kontain proprietary `KKM module and related /dev/kkm device`.

At this moment, either KVM should be available locally on or Azure/GCP instance with nested virtualization enabled, or you can use AWS Kontain-Ubuntu pre-built AMI to experiment with KKM (Ubuntu 20 with KKM preinstalled).

We are working on install for KKM - it requires building with the exact kernel header version snd is being worked on for the release.

### Install on a machine with KVM enabled

* Generally, we just validate prerequisites and place necessary files into /opt/kontain
  * For now, we just un-tar the tarball
  * We plan to provide native packaging (.rpm/.deb/apkbuild) at the release time
* For the inpatient -  `wget -q -O - https://raw.githubusercontent.com/kontainapp/km-releases/master/kontain-install.sh | bash` - but please see the above link to km-releases for the latest.

#### Validate

Assuming you have gcc (and access to /dev/kvm) the easiest way to validate is to build a simple unikernel and run it:

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

gcc -c -o $dir/$file.o $dir/$file.c
/opt/kontain/bin/kontain-gcc -o $dir/$file.km $dir/$file.o
/opt/kontain/bin/km $dir/$file.km
```

Note that `.km` is the ELF file with Kontain unikernel, and the last command was executed in Kontain VM.
If you want to see the trace of KVM ioctl, you can enable cpu tracing: `/opt/kontain/bin/km -Vcpu $dir/$file.km`. To get help for KM command line, `/opt/kontain/bin/km --help`

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

In order to run Kontainerized payloads in Kontain VM on Kubernetes, each Kubernetes node needs to have KM installed, and we also need KVM Device plugin which allows the non-privileged pods access to /dev/kvm

All this is accomplished by deploying KontainD Daemon Set:

Documentation TBD

* Install kontaind
* validate a simple run

## Getting started

### Building your own Kontain unikernel

To build a Kontain unikernel, you can do one of the following

* Build or use a regular no-libc executable, e.g. GOLANG
* Link existing object files into a musl-libc based executable (like you would do for Alpine containers)
* Link existing object files into a Kontain runtime-based executable

We recommend statically linked executable (dynamic linking is supported but requires payload access to shared libraries , so we will leave it outside of this short into).

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

### Using with existing executables

Kontain currently support running not-modified Linux executables as unikernels in Kontain VM, provided:

* The executable does not use GLIBC (glibc is not supported yet).
  * So the executables could be one build for Alpine, or one not using libc (e.g. GOLANG)
* The executable is not using some of the exotic/not supported syscalls (more details on this later, but apps like Java or Python or Node are good with this requirement)

kontain-gcc has `-alpine` flag to build musl-based executable. E.e. using the examples/vars above, build it with `/opt/kontain/bin/kontain-gcc -alpine -o $dir/$file $dir/$file.o` and validate the results:

`$dir/$file` will run the result as regular linux binary and print out something like this:

```sh
Hello World from the following runtime environment:
sysname         = Linux
nodename        = msterin-p330
release         = 5.7.16-200.fc32.x86_64
version         = #1 SMP Wed Aug 19 16:58:53 UTC 2020
machine         = x86_64
```

`/opt/kontain/bin/km $dir/$file` will run it as unikernel in Kontain VM and print out something like this:

```sh
Hello World from the following runtime environment:
sysname         = kontain-runtime
nodename        = msterin-p330
release         = 4.1
version         = preview
machine         = kontain_VM
```

### Using pre-build unikernels for interpreted languages

Kontain supports a set of pre-built language systems, as containers you can use in FROM dockerfiles statement, or passing scripts dirs as volumes.

The images are available on dockerhub as `kontainapp/runenv-<language>` e.g. `kontainapp/runenv-python`. Kontain provides the following prebuilt languages:

* jdk-11.0.8
* node (Java script)
* python (3.6)

We will extend the list by the release time. Also - see the section below on linking your own, if needed

Example:  You can run a interactive python as inside Kontain VM using this:

`docker run --device /dev/kvm -it --rm  -v /opt/kontain/bin/km:/opt/kontain/bin/km:z kontainapp/runenv-python`

**NOTE** Currently you may see debug messages there, and the container size is not optimized yet.

Please check * java
* node.js

### Using snapshots

Kontain supports snapshots and running from snapshots. A snapshot is an ELF file which can be run as directly as a unikernel in Kontain VM, but captures payload state-in time.

Snapshot can be triggered by an API called from the payload, or by an external `km_cli` running on host and communicating to Kontain Monitor (KM)

Limitations:

* no coordination of multi-process payload snapshots


TODO:
 * Doc and example


### Using with Kubernetes

After installation of kontaind is completed (see above), you can use API server/kubectl to deploy Kontain-based pods to a Kubernetes cluster.
A spec for the pod needs to request /dev/kvm device in the `resources` section, host volume mount,
and entrypoint calling KM (unless symlinks are set up - see below).

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

Payload running as a unikernel in Kontain V< will generate a coredump in the same cases it would have generated if running on Linux. The file name is `kmcore` (can be changed with `--coredump=file` flag). You can debug witha payload  coredump as a regular Linux coredump, e.g. `gdb program.km kmcore`

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

If Kontain  Monitor (KM) is invoked via a symlink , it assumes the original file was side by side with the unikernel ELF and runs this elf.
The name of the elf is the \<original_file_name>.km

For example, if you have the following files in your directory:

```
python -> opt/kontain/bin/km
python.km
```

The running `./python` will result in running `/opt/kontain/bin/km ./python.km`. This allows to keep all oriinal scripts and shebang files using Kontain unmodified.

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

To use kontain java runtime environment with dockerfile, user can substitude
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
avaliable. For Java we also requires kontain's version of `lic.so`.

```bash
docker run -it --rm \
    --device /dev/kvm \
    -v /opt/kontain/bin/km:/opt/kontain/bin/km:z \
    -v /opt/kontain/runtime/libc.so:/opt/kontain/runtime/libc.so:z \
    example/kontain-java
```

## Architecture

TODO

### Kontain Monitor (VM hardware model) and Kontain unikernels

TODO. Outline:

Link to google doc if we have something.
System design diagram
Hardware model - threads, memory, IO
Virtualization levels (in-KM, outside of KM)
Pillars (no memory mgmt, delegation to host, etc)
KM code components

Supported syscalls - philosophy
Sandboxing vi OUT/hcalls
Sandboxing via syscall intercept
Supported syscals and delegation to host
  relations to seccomp

### Solution for no-nested-virtualization machines

When nested virtualization is not available, Kontain provides a Kontain Kernel module (`kkm`) that implements a subset of KVM ioclts.  It doies not reuse KVM code or algorithms, because the requiements are much simpler - but it does implement a subset if `KVM ioctls` and uses the same control paradigm. It communicates via special device `/dev/kkm`.

TODO: KKM architecture (high level) goes here

`kkm` requires to be built with the same kernel version and same kernel config as the running system. We are working on proper installation.
TODO: doc on installation-with-build, when installation is ready

#### Amazon - pre-built AMI

Meanwhile, we provide an AWS image (Ubuntu 20 with pre-installed and pre-loaded KKM) whih can be used to experiment / test Kontain on AWS.

AMI is placed in N. California (us-west-1) region. AMI ID is `ami-047e551d80c79dbb7`.

To create a VM:

```
aws ec2 create-key-pair  --key-name aws-kkm --region us-west-1
aws ec2 run-instances --image-id ami-047e551d80c79dbb7 --count 1 --instance-type t2.micro --region us-west-1 --key-name aws-kkm
# before next step  save the key to ~/.ssh/aws-kkm.pem and chown it to 400
```

You can then ssh to the VM , install the latest Kontain per instructions above and run it.
The only difference- please use `/dev/kkm` instead of `/dev/kvm`

### Kontainers

Regular runtime - mount, namespaces, using snapshots. Pluses and minuses
krun - why and how to use
*  OCI spec and use runtimes for bringing in existing kms. Update doc on how to use then and what needs to be in. Add an example
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
knows how to scheudle workloads onto the right node. For this, we developed
kontaind, a device manager for `kvm` and `kkm` device, running as a daemonset
on nodes that has these devices avaliable.

To deploy the latest version of `kontaind`, run:

```bash
kubectl apply \
  -f https://github.com/kontainapp/km-releases/blob/master/k8s/kontaind/deployment.yaml?raw=true
```

Alternatively, you can download the yaml and make your own modification:

``` bash
wget https://github.com/kontainapp/km-releases/blob/master/k8s/kontaind/deployment.yaml?raw=true -O kontaind.yaml
```

TODO: fix the link here after deciding where the yaml file will be.

## dev guide

* Tools to build simple unikernels with kontain-gcc
* Example of building complex unikernels (e.g python)
* Running alpine exec as-is in a unikernel

## Cloud

### Azure
Azure supports nested virtualization for some types of instances since 2017: https://azure.microsoft.com/en-us/blog/nested-virtualization-in-azure/.
Kontain CI/CD process uses `Standard_D4s_v3` insance size.

Create one of these instances, SSH to it , then install and try Kontain as described above.

Kontain runs it's own CI/CD pipeline on Azure Managed Kubernetes, and AWS for no-nested-virtualization code.

### AWS

For AWS, Kontain can run on `*.metal` instances. For virtual instances, Kontain provides `kkm` kernel module and pre-build AMI with Ubuntu20.
Unfortunately AWS does not support nested virtualization  on non-metal instanses, so a kernel modules is required.

### Other clouds (vSphere, GCP, ...)

Kontain will work on other clouds (e.g. vSphere or GCP) in a Linux VM with nested virtualization enabled


======= END OF THE DOC ====