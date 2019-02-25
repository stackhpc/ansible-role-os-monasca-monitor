stackhpc.os-monasca-monitor
===========================

[![Build Status](https://www.travis-ci.org/stackhpc/ansible-role-os-monasca-monitor.svg?branch=master)](https://www.travis-ci.org/stackhpc/ansible-role-os-monasca-monitor)

This role deploys [Monasca Monitor](https://github.com/stackhpc/monasca-monitor)
as a systemd service. It also configures alarms and notifications required for
Monasca-Monitor to work. After running it you should have a Prometheus endpoint
which can be used to monitor the status of the Monasca alerting pipeline.

Usage
-----

#. Decide which OpenStack project you want the monitor to be associated with
   and make sure that a user exists in that project, with the monasca user
   role.
#. Create a playbook to run the role, overriding any defaults you need to,
   for example:
   ```
   $ cat my_playbook.yml
   ---
   - name: Ensure Monasca Monitor is deployed
     hosts: some-node-with-public-openstack-api-access
     vars:
       monasca_monitor_port: 8000
     roles:
       - name: stackhpc.os_monasca_monitor
         monasca_monitor_bind_port: "{{ monasca_monitor_port }}"
         monasca_monitor_bind_host: "10.0.0.10"
         monasca_monitor_auth_url: "http://{{ public_os_vip_address }}:5000"
         monasca_monitor_project_name: "monasca-monitoring"
         monasca_monitor_password: "{{ secrets_monasca_monitoring_password }}"
         monasca_monitor_username: "monasca-monitor"
         monasca_monitor_region_name: "RegionOne"
         monasca_monitor_endpoint_type: "public"
     tasks:
       - name: Open port to allow ping-back from monasca notification service
         iptables:
           state: present
           jump: ACCEPT
           in_interface: enp0s1
           chain: IN_public_allow
           protocol: tcp
           destination_port: "{{ monasca_monitor_port }}"
           comment: Enable ping back to Monasca Metrics service.
         become: yes
   ```
#. Run the playbook against the host where you want the monitor to run:
   ```
   $ ansible-playbook -i localhost my_playbook.yml
   ```

#. Configure Prometheus to scrape the endpoint. By default this is bound to all
   interfaces on the host. The metric of interest is
   `monasca_monitor_heartbeat_total` which by default should increment every
   1 minute. The idea is to set an alarm in Prometheus to ensure that the counter
   continues to increment. If it stops then the Monasca alerting pipeline should
   be checked. Example output from the Prometheus endpoint is given below:
   ```
   [stack@sv-b16-u22 monasca-monitor]$ curl http://10.0.0.100:8000/metrics
   # HELP process_virtual_memory_bytes Virtual memory size in bytes.
   # TYPE process_virtual_memory_bytes gauge
   process_virtual_memory_bytes 2.64052736e+08
   # HELP process_resident_memory_bytes Resident memory size in bytes.
   # TYPE process_resident_memory_bytes gauge
   process_resident_memory_bytes 2.9036544e+07
   # HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
   # TYPE process_start_time_seconds gauge
   process_start_time_seconds 1.55068281646e+09
   # HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
   # TYPE process_cpu_seconds_total counter
   process_cpu_seconds_total 0.72
   # HELP process_open_fds Number of open file descriptors.
   # TYPE process_open_fds gauge
   process_open_fds 13.0
   # HELP process_max_fds Maximum number of open file descriptors.
   # TYPE process_max_fds gauge
   process_max_fds 1024.0
   # HELP python_info Python platform information
   # TYPE python_info gauge
   python_info{implementation="CPython",major="2",minor="7",patchlevel="5",version="2.7.5"} 1.0
   # HELP monasca_monitor_heartbeat_total Monasca system level heartbeat
   # TYPE monasca_monitor_heartbeat_total counter
   monasca_monitor_heartbeat_total 8.0
   # TYPE monasca_monitor_heartbeat_created gauge
   monasca_monitor_heartbeat_created 1.550682817310154e+09
   ```

License
-------

Apache

Author Information
------------------

http://www.stackhpc.com
