Ceph
====


Deployment
----------

  * [ceph-ansible](https://github.com/ceph/ceph-ansible)
  * [ceph-docker](https://github.com/ceph/ceph-docker)


### Play `ceph-ansible` with CentOS cloud hosts

`site.yml`
```
- hosts: mons # gather mon facts first before we process ceph-mon serially
  tasks: []

- hosts: mons
  become: True
  gather_facts: false
  roles:
  - ceph-mon
  serial: 1

- hosts: osds
  become: True
  roles:
  - ceph-osd
```

`hosts`
```
[mons]
centos-01 ansible_ssh_host=10.3.0.15
centos-02 ansible_ssh_host=10.3.0.18
centos-03 ansible_ssh_host=10.3.0.14

[osds]
centos-04 ansible_ssh_host=10.3.0.16
centos-05 ansible_ssh_host=10.3.0.17
centos-06 ansible_ssh_host=10.3.0.19
```

`group_vars/all`
```
cluster_network: 10.3.0.0/24
generate_fsid: 'true'
ceph_origin: upstream
ceph_stable: true
monitor_interface: eth0
public_network: 10.3.0.0/24
journal_size: 5120
```

`group_vars/mons`
```
cephx: false
```

`group_vars/osds`
```
journal_collocation: true
devices:
   - /dev/vdb
cephx: false
```


### Play `ceph-ansible` with Atomic hosts

`hosts`
```
[mons]
atomic-01 ansible_ssh_host=10.3.0.15
atomic-02 ansible_ssh_host=10.3.0.18
atomic-03 ansible_ssh_host=10.3.0.14

[osds]
atomic-04 ansible_ssh_host=10.3.0.16
atomic-05 ansible_ssh_host=10.3.0.17
atomic-06 ansible_ssh_host=10.3.0.19
```

`ansible.cfg`
```
[defaults]
action_plugins = plugins/actions
retry_files_enabled = False
host_key_checking = False
remote_user = centos
```

```
$ ansible -i hosts all -m ping
atomic-05 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
atomic-01 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
atomic-04 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
atomic-03 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
atomic-06 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
atomic-02 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

```
$ cp site.yml.sample site.yml
$ cp group_vars/all.sample group_vars/all
$ cp group_vars/mons.sample group_vars/mons
$ cp group_vars/osds.sample group_vars/osds
```

`group_vars/all`
```
mon_containerized_deployment: true
osd_containerized_deployment: true
mds_containerized_deployment: true
rgw_containerized_deployment: true
nfs_containerized_deployment: true
restapi_containerized_deployment: true
rbd_mirror_containerized_deployment: true
cluster_network: 10.3.0.0/24
ceph_docker_on_openstack: true
ceph_rgw_civetweb_port: 8080
generate_fsid: 'true'

ceph_docker_dev_image: false
ceph_origin: distro
monitor_interface: eth0
public_network: 10.3.0.0/24
journal_size: 5120
```

`group_vars/osds`
```
journal_collocation: true
devices:
   - /dev/vdb
ceph_osd_docker_devices: ['/dev/vdb']
ceph_osd_docker_extra_env: "CEPH_DAEMON=OSD_CEPH_DISK_ACTIVATE,OSD_JOURNAL_SIZE=100"
cephx: false
```

`group_vars/mons`
```
cephx: false
ceph_mon_docker_interface: eth0
ceph_mon_docker_subnet: 10.3.0.0/.0/24
```

```
# upgrade and reboot the nodes
$ ansible -i hosts all -m shell -a "rm -rf /etc/ceph/*.conf" -u centos -s
$ ansible -i hosts all -m shell -a "rm -rf /var/lib/ceph/*" -u centos -s
$ ansible-playbook -i hosts site.yml --skip-tags with_pkg
```

### Ceph-ansible; Ceph Atomic

  * https://gitlab.com/gbraad/ceph-atomic
  * https://gitlab.com/gbraad/byo-atomic-ceph

```
$ git clone https://github.com/ceph/ceph-ansible.git
$ cp site.yml.sample site.yml
$ cp group_vars/all.sample group_vars/all
$ cp group_vars/mons.sample group_vars/mons
$ cp group_vars/osds.sample group_vars/osds
```

`group_vars/mons`
```
cluster_network: 10.3.0.0/24
generate_fsid: 'true'
skip_tags: 'with_pkg'
ceph_origin: distro
monitor_interface: eth0
public_network: 10.3.0.0/24
journal_size: 5120
cephx: false
```

`group_vars/osds`
```
journal_collocation: true
cephx: false
devices:
   - /dev/vdb
```

```
$ ansible -i hosts all -u centos -s -m shell -a "setenforce 0"
$ ansible -i hosts all -u centos -s -m shell -a "ostree remote add --set=gpg-verify=false byo-atomic-ceph https://gbraad.gitlab.io/byo-atomic-ceph/"
$ ansible -i hosts all -u centos -s -m shell -a "rpm-ostree rebase byo-atomic-ceph:centos-atomic-host/7/x86_64/ceph-jewel"
$ ansible -i hosts all -u centos -s -m shell -a "systemctl reboot"  # will throw errors as connection is dropped
$ ansible -i hosts all -u centos -s -m ping
$ ansible-playbook -i hosts site.yml --skip-tags with_pkg,package-install
```

[Gist](https://gist.github.com/gbraad/9111e00e91170d91a1d180c3b62423c6)


### Ceph-ansible: installation source

`group_vars/all`
```
#ceph_origin: distro
ceph_origin: upstream
ceph_stable: true
```

`distro` does not install any repos, while `upstream` will and needs you to specify and installation type; `ceph_stable` used here.


Building
--------

  * Build wrapper: [GitLab](https://gitlab.com/gbraad/ceph), [GitHub](http://github.com/gbraad/ceph-build-wrapper)  
    Runs automated builds of the master branch at `ceph/ceph`.


### Standard build using cmake
The following script performs a basic build using `cmake` on CentOS 7 (Cloud image).

```
# prepare
$ yum install -y git
$ git clone git://github.com/ceph/ceph --depth 1
$ cd ceph
$ git submodule update --init --recursive
$ ./install-deps.sh
# build
$ ./do_cmake.sh
$ cd build
$ make
```

Links
-----

  * [Ceph](http://ceph.com/) homepage
