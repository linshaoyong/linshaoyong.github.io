---
layout: post
title: "使用ansible批量格式化并挂载磁盘"
date: 2018-02-23 16:15:50 +0800
tags: [ansible, format disk]
---

初始化安装elasticsearch这类集群系统的时候，往往需要批量操作大量的磁盘。可以通过ansible快速完成这类工作。

假设每台设备有10块盘(sdxxx)，要挂载到/dataxxx，elasticsearch数据目录为/dataxxx/es，首先将变量写入vars/main.yaml，

```yaml
---

disks:
  /dev/sdb: /data1
  /dev/sdc: /data2
  /dev/sdd: /data3
  /dev/sde: /data4
  /dev/sdf: /data5
  /dev/sdg: /data6
  /dev/sdh: /data7
  /dev/sdi: /data8
  /dev/sdj: /data9
  /dev/sdk: /data10
```

然后，创建任务文件，tasks/main.yaml，

```yaml
---
{% raw %} 
- file:
    path: "{{ item.value }}/es"
    state: directory
    owner: elasticsearch
    group: elasticsearch
    mode: 0755
  with_dict: "{{ disks }}"

- name: umount datanode disks
  mount:
    path: "{{ item.value }}"
    state: absent
  with_dict: "{{ disks }}"

- name: format datanode disks
  filesystem: fstype=xfs dev="{{ item.key }}" force=true
  with_dict: "{{ disks }}"

- name: mount datanode disks
  mount:
    path: "{{ item.value }}"
    src: "{{ item.key }}"
    fstype: xfs
    opts: "defaults,noatime,nobarrier"
    state: mounted
  with_dict: "{{ disks }}"
{% endraw %}
```

创建目录、umount、format、mount、写入/etc/fstab，一气呵成。
