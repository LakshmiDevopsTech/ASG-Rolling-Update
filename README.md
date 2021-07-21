# ASG Rolling Update(Ansible)
## Description
Ansible Role that can be used either for doing Rolling Deployment on the ASG. 
Usually in a ELB working with ASG while updateing the new site contents through git, we prefer increase the number of instances in ASG and once contents updated will remove old instances. Its practically very difficuly. 
Using this ansible playbook we can easily update new contents withour terminating instances.
### Features
- Included Auto Scaling Group, and ELB in this playbook
- No need to create manuel inventory file. We're fetching instances details created by asg through Dynamic Inventory file.
### Pre-Requests
- Install Ansible on your Master Machine (localhost)
- Install pip, boto, boto3 and botocore
- Create an IAM user role under your AWS account and please enter the values once the playbook running time or attach the role to ansible master server.
##### Ansible Modules used
- [yum](https://docs.ansible.com/ansible/2.9/modules/yum_module.html)
- [pip](https://docs.ansible.com/ansible/2.9/modules/pip_module.html)
- [ec2_group](https://docs.ansible.com/ansible/2.3/ec2_group_module.html)
- [ec2_lc](https://docs.ansible.com/ansible/2.3/ec2_lc_module.html)
- [ec2_elb_lb](https://docs.ansible.com/ansible/2.3/ec2_elb_lb_module.html)
- [ec2_asg](https://docs.ansible.com/ansible/2.3/ec2_asg_module.html)
- [ec2_instance_info](https://docs.ansible.com/ansible/2.9/modules/ec2_instance_info_module.html)
- [add_host](https://docs.ansible.com/ansible/2.9/modules/add_host_module.html)
- [git](https://docs.ansible.com/ansible/2.9/modules/git_module.html)
- [file](https://docs.ansible.com/ansible/2.9/modules/file_module.html)
- [pause](https://docs.ansible.com/ansible/2.9/modules/pause_module.html)
- [copy](https://docs.ansible.com/ansible/2.9/modules/copy_module.html)
- [service](https://docs.ansible.com/ansible/2.9/modules/service_module.html)

### Architacture
![alt text](https://ibb.co/wsndf2g/asg-rolling-ansible)

### Behind the Playbook
I just explaining some Important parts of the playbook here. For detaild code, please check .yml file.

```
# create ASG
- name: "create ASG"
       ec2_asg:
         name: "{{ name }}_asg"
         launch_config_name: "{{ lc_status.name }}" <--- lc creation check .yml file
         load_balancers: "{{ lb_status.elb.name }}" <---- elb creation, check .yml file
         health_check_period: 30
         health_check_type: EC2
         replace_all_instances: yes
         min_size: 3
         max_size: 3
         desired_capacity: 3
         region: "{{ region }}"
         tags:
           - Name: devops-web
             propagate_at_launch: true
       register: asg_status
     - name: "asg_status"
       debug:
         var: "asg_status.auto_scaling_group_name"
 
 # Dynamic Inventory creation
 - name: "collect ec2 instance details created by asg"
       ec2_instance_info:
         region: "{{ region }}"
         filters:
           "tag:aws:autoscaling:groupName": "{{ asg_status.auto_scaling_group_name }}"
           instance-state-name: [ "running"]
       register: instance_status
     - name:
       debug:
         var: instance_status

     - name: "Aws - Creating Dynamic Inventory"
       add_host:   <---------- Module for create Dynamic Inventory file
         name: "{{ item.public_ip_address }}" <--- public ip of instances by asg
         groups: "instances" <-- Group name in inventory file (hosts)
         ansible_host: "{{ item.public_ip_address }}" 
         ansible_port: 22
         ansible_user: "ec2-user"
         ansible_ssh_private_key_file: "/root/{{ key }}.pem"
         ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
       with_items:
         - "{{ instance_status.instances }}" <-- status of instances created by asg
         
# Rolling update
 - name: "Rolling update starting"
   hosts: instances <-- inventory file group (hosts )
   become: true
   serial: 1
   vars_files:
     - update.vars
   tasks:

     - name: "Cloning Git repository"
       git:
         repo: "{{ git_url }}"
         dest: /var/contents/
       register: repo_status
```
## Conclusion
The above playbook created for ASG rolling update without recreate instances.
Please see the asg.vars and update.vars for check variables and see ansible-asg.yml for playbook details.
