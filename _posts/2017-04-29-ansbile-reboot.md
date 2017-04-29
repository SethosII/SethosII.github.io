---
layout: post
title:  "Ansbile reboot task"
date:   2017-04-29 13:15:00 +0100
author: Paul Jähne
---

I recently answered a [question on serverfault](https://serverfault.com/questions/836596/how-can-i-test-that-a-reboot-has-completed/836715#836715) about rebooting a host and executing tasks afterwards with Ansible. I got this from a [support article](https://support.ansible.com/hc/en-us/articles/201958037-Reboot-a-server-and-wait-for-it-to-come-back) which isn't available anymore. So I guess it might be helpful to share this again here and as a [Gist](https://gist.github.com/SethosII/4f822530d125fdf60ef70a3fa23f2676).

```yaml
- name: "reboot hosts"
  shell: "sleep 2 && shutdown -r now 'Reboot triggered by Ansible'" # sleep 2 is needed, else this task might fail
  async: "1" # run asynchronously
  poll: "0" # don't ask for the status of the command, just fire and forget
  ignore_errors: yes # this command will get cut off by the reboot, so ignore errors
- name: "wait for hosts to come up again"
  wait_for:
    host: "{{ inventory_hostname }}"
    port: "22" # wait for ssh as this is what is needed for ansible
    state: "started"
    delay: "120" # start checking after this amount of time
    timeout: "360" # give up after this amount of time
  delegate_to: "localhost" # check from the machine executing the playbook
```
