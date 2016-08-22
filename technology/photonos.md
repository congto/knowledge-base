Photon OS
=========


  * [GitLab](https://gitlab.com/gbraad/photon) buildwrapper
  * [Unofficial mirror](https://gitlab.com/unofficial-mirrors/photon) on GitLab, [my fork](https://gitlab.com/gbraad/vmware-photon)


Install on OpenStack
--------------------

```
$ wget https://bintray.com/artifact/download/vmware/photon/photon-ami-1.0-13c08b6.tar.gz
$ tar -zxvf photon-ami-1.0-13c08b6.tar.gz
$ openstack image create photon --file photon-ami-1.0-13c08b6.raw --container-format bare --disk-format raw
$ openstack server create photon --flavor m1.tiny --image photon --key-name c9
$ ssh root@[ipaddress]
root@photon [ ~ ]#
```