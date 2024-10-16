# Irqbalance CoreOS Layering

In OpenShift 4.14 a new concept called RHCOS image layering was introduced which allows one to build a container layer they can then apply.  More details about it can be found [here](https://docs.openshift.com/container-platform/4.15/post_installation_configuration/coreos-layering.html).  We need to leverage this technology to apply an image layer that contains irqbalance since this package is not part of the base RHCOS nor is it available as an extension.  It will become part of RHCOS in the future based on this [merge request](https://github.com/openshift/os/pull/1602). The steps below will describe how create and apply the RHCOS image layer. 

The first step is to get the current `rhel-coreos` image from the cluster where we will be applying the image.

~~~bash
$ oc adm release info --image-for rhel-coreos
quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a2f1b7530956b765f1b0337b824fde28d6987b519eec0aaadc9d261e9fd1e550
~~~

Next we take the image output and place it into a Dockerfile that will have a run command to install and enable the package.

~~~bash
$ cat <<EOF > Dockerfile.irqbalance 
FROM quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a2f1b7530956b765f1b0337b824fde28d6987b519eec0aaadc9d261e9fd1e550


RUN rpm-ostree install irqbalance && \  #install the irqbalance package
    systemctl enable irqbalance && \    #enable irqbalance service
    rm -r -f /etc/yum.repos.d/* \       #remove entitlements from the system the image is being built on otherwise creates issues 
    ostree container commit             #commit the ostree container
EOF
~~~

Once we have created the Dockerfile we can use podman to build the container.  Note that I performed this process on an Arm64 host because I wanted my image to be made for an Arm64 node. 

~~~bash
$ podman build -t quay.io/redhat_emp1/ecosys-nvidia/ocp-4.15-irqbalance:4.15.23 -f Dockerfile.irqbalance .
STEP 1/2: FROM quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a2f1b7530956b765f1b0337b824fde28d6987b519eec0aaadc9d261e9fd1e550
Trying to pull quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a2f1b7530956b765f1b0337b824fde28d6987b519eec0aaadc9d261e9fd1e550...
Getting image source signatures
Copying blob b839e1a7e4d1 done   | 
Copying blob d178eea3fd50 done   |
(...)
Copying blob 413ee3c4305f done   | 
Copying config 9f41514ccf done   | 
Writing manifest to image destination
STEP 2/2: RUN rpm-ostree install irqbalance &&     systemctl enable irqbalance &&     ostree container commit
Enabled rpm-md repositories: rhel-9-for-aarch64-appstream-rpms rhel-9-for-aarch64-baseos-rpms
Updating metadata for 'rhel-9-for-aarch64-appstream-rpms'...done
Updating metadata for 'rhel-9-for-aarch64-baseos-rpms'...done
Importing rpm-md...done
rpm-md repo 'rhel-9-for-aarch64-appstream-rpms'; generated: 2024-09-12T14:26:05Z solvables: 19074
rpm-md repo 'rhel-9-for-aarch64-baseos-rpms'; generated: 2024-09-11T06:47:28Z solvables: 7179
Resolving dependencies...done
Will download: 1 package (71.6?kB)
Downloading from 'rhel-9-for-aarch64-baseos-rpms'...done
Installing 1 packages:
  irqbalance-2:1.9.2-3.el9.aarch64 (rhel-9-for-aarch64-baseos-rpms)
Installing: irqbalance-2:1.9.2-3.el9.aarch64 (rhel-9-for-aarch64-baseos-rpms)
Created symlink /etc/systemd/system/multi-user.target.wants/irqbalance.service → /usr/lib/systemd/system/irqbalance.service.
COMMIT quay.io/redhat_emp1/ecosys-nvidia/ocp-4.15-irqbalance:4.15.23
--> 624c70f71a77
Successfully tagged quay.io/redhat_emp1/ecosys-nvidia/ocp-4.15-irqbalance:4.15.23
624c70f71a77ce3c30fd973b4afc09fc4558b43e64ec43851de2cf4d7ad7f6a0
~~~

Once the container is built we can push it to our favorite location in the registry.  This image is pushed to a location that users of this documentation have access to already.

~~~bash
$ podman push quay.io/redhat_emp1/ecosys-nvidia/ocp-4.15-irqbalance:4.15.23
Getting image source signatures
Copying blob 79bb3562fc5d done   | 
Copying blob 6dbb46d6d565 done   | 
(...)
Copying blob 00ad2dbb774b done   | 
Copying config 624c70f71a done   | 
Writing manifest to image destination
~~~

Once the image is in a registry location we can generate the machine configuration file and specify the `osImageURL` with the location of our image.

~~~bash
$ cat <<EOF >irqbalance-machine.yaml 
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master 
  name: irqbalance-layer-machineconfig 
spec:
  osImageURL: quay.io/redhat_emp1/ecosys-nvidia/ocp-4.15-irqbalance:4.15.23
EOF
~~~

Once the file is generated we can create it on our cluster.  This will apply the image layer and the host will be rebooted.

~~~bash
$ oc create -f irqbalance-machine.yaml 
machineconfig.machineconfiguration.openshift.io/irqbalance-layer-machineconfig created
~~~

After the reboot we can validate that `irqbalance` is successfully installed and running by going into a debug container and checking for the package and the `systemctl` status.

~~~bash
$ oc debug node/$(oc get node -o json | jq -r '.items[0].metadata.name')
Starting pod/nvd-srv-37nvidiaengrdu2dcredhatcom-debug-7r745 ...
To use host binaries, run `chroot /host`
Pod IP: 10.6.135.16
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host
sh-5.1# rpm -q irqbalance
irqbalance-1.9.2-3.el9.aarch64
sh-5.1# systemctl status irqbalance
● irqbalance.service - irqbalance daemon
     Loaded: loaded (/usr/lib/systemd/system/irqbalance.service; enabled; preset: enabled)
     Active: active (running) since Thu 2024-09-12 19:20:27 UTC; 1h 21min ago
       Docs: man:irqbalance(1)
             https://github.com/Irqbalance/irqbalance
   Main PID: 14874 (irqbalance)
      Tasks: 2 (limit: 783503)
     Memory: 5.2M
        CPU: 3.980s
     CGroup: /system.slice/irqbalance.service
             └─14874 /usr/sbin/irqbalance --foreground

sh-5.1#
~~~
