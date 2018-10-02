### Documentation structure
[**About this repository**](README.md#about-this-repository)  
[**Introduction**](README.md#introduction)  
[**Various 'edit' options available**](README.md#various-edit-options-available)  
[**"configure exclusive" workflow with automation**](README.md#configure-exclusive-workflow-with-automation)  
[**"configure private" workflow with automation**](README.md#configure-private-workflow-with-automation)  
[**Conclusion**](README.md#conclusion)  

### About this repository
This repository has no code. It is about 'edit exclusive' and 'edit private' usage with Network automation.  
So, should we use 'edit exclusive' or 'edit private' with Network automation?  

### Introduction  
Junos supports various ‘edit’ options. Should we use 'edit exclusive' or 'edit private' with Network automation? There is no "one size fits all" solution to this question. However, here are the recommendations.  

### Various 'edit' options available 
 
Junos CLI supports 'configure exclusive' and 'configure private' and 'configure'. We recommend to never use the shared configuration database (configure) as there is a risk of committing incorrect configuration changes when two users are editing the configuration at the same time.  
 
PyEZ (junos-eznc python library) supports 'configure exclusive' and 'configure private'  

SaltStack modules for Junos currently only uses 'configure exclusive'  

Junos space currently only uses 'configure exclusive'  

Ansible core modules for Junos use currently only 'configure exclusive'.    

Ansible galaxy module [**juniper_junos_config**](http://junos-ansible-modules.readthedocs.io/en/2.0.1/juniper_junos_config.html) supports both 'configure exclusive' and 'configure private'.    
 
### "configure exclusive" workflow with automation

Most of our customers are using 'configure exclusive' with automation tools/processes.  

A "configure exclusive" workflow ensures there will never be conflicting changes committed into the network.  

The drawback is that the configuration task may "fail" because the configuration is currently locked. This is not specific to network automation.  

The same arises when configuration changes are manual.  
In general, an automated procedure is going to have the configuration locked for a much shorter time than an equivalent configuration change from a human, and having the configuration locked for shorter periods will decrease the likelihood of a "conflict".  

If there are also humans involved in making configuration changes then there should be a process/procedure to ensure that “humans” use “configure private” and avoid blocking any automation that takes place as a result of using “configure exclusive” from an automation tools perspective.  

That said, conflicts may still occur, and they must have a process for dealing with them. There must be a process which defines how to handle this situation (Do you retry automatically or manually? If automatically, how frequently do you retry? How many times do you retry? What happens if it still fails after max retries?) 

Here's [**how to retry ansible playbooks manually and automatically for the devices that failed**](https://github.com/ksator/EVPN_DCI_automation/wiki/how-to-retry-a-playbook-for-the-devices-that-failed)  
Here's the doc about [**Error handling with ansible**](http://docs.ansible.com/ansible/latest/playbooks_error_handling.html)   

PyEZ also has exception handling support:  
[**Here's the doc**](https://www.juniper.net/documentation/en_US/junos-pyez/topics/example/junos-pyez-program-configuration-data-loading-from-file.html)  
[**Here's examples**](https://github.com/ksator/python-training-for-network-engineers/tree/master/exceptions)    

### "configure private" workflow with automation

"configure private" workflow can be used with automation as well.   

In that case, the configuration is not locked. So, no configuration lock error can happen.      

However, using edit private, if the same section of the configuration is changed "simultaneously" (i.e 2 different 'edit private' happens before the first commit), then, depending on the configuration changes, a configuration conflict might happen. In that case, the second commit will fail.  
This is not specific to network automation. The same arises when configuration changes are manual.  

A human using junos cli can use the command "update" if, based on his judgment, he decides to commit the second change, but this will discard the previous commit.  

Automation executes tasks (opens a private configuration database, loads changes to their private configuration database, commit changes) quicker than humans.  
That makes it difficult for some other human/process to edit the configuration in between. 
However, it is certainly still possible.  
Therefore, we need to ensure a configuration conflict does not occur.  

"configure private" is a valid workflow if a given section of the configuration only has one "single source of authority".  

For example, you might have thing1 maintaining BGP neighbor configuration and policy, thing2 maintaining OSPF, and humans manually configuring interfaces. In this situation, thing1 would be the "single source of authority" for BGP, thing2 would be the "single source of authority" for OSPF, and the humans would be the single source of authority for interfaces configuration. thing1 and thing2 and humans can configure the network simultaneously as they 'own' different sections of the configuration.  
For each section of the device configuration (think of this as a list of one or more "edit" hierarchies in Junos), we recommend that only a single "thing" should "control" that section of the hierarchy.  
Let's use another example: VLANs on a network comprised of EX switches. In this case, the tool would touch both the [edit vlans] and [edit interfaces] hierarchies. I am simply suggesting that this tool "owns" those two configuration hierarchies and that no other tool (or humans) should be modifying those configuration hierarchies.  
In this particular example, we might choose to further refine the portions of the configuration controlled by the tool: this tool controls VLANS in the range 1000-3000, but another tool (or humans) control VLANS outside that range. We might also define that interfaces ge-0/0/0 to ge-0/0/23 are controlled by this tool while interfaces outside that range are controlled by another tool.  
The idea is very simple. "Don't have two things changing the same config"  
With a workflow that ensures a "single source of authority" for each section of their configuration, then a "configure private" workflow is the preferred workflow.  

Two processes/people can safely configure a network device at the same time using "configure private" as long as they are not both trying to change the same section of the configuration.  

Even if we should have a "single source of authority" for each section of the configuration with an ‘edit private’ workflow, a robust automation tool which uses the "configure private" workflow really should retry and/or fail in a predictable way in the event of conflict.  

However, if the same "single source of authority" do changes "simultaneously" (i.e 2 different 'edit private' happens before the first commit) on the same section of the configuration, then, depending on the configuration changes, a conflict might happen, and in that case, the second commit will fail. Whether it is ansible or python or cli.  
If it is ansible juniper_junos_config module, the second commit would fail and the config changes from that task would be discarded.  
If it is cli, the second commit will fail and the human using junos cli can use the command "update" if he decides based on his judgment to commit the second change, but this will discard the previous commit.  

Also, if two processes/people do change the same section of the configuration at roughly the same time, then you get an unpredictable final configuration depending on which commit happened first.  

If the process does not ensure a "single source of authority" for each section of their configuration, I would recommend against this workflow.  

### Conclusion

With a workflow that ensures a "single source of authority" for each section of their configuration, then a "configure private" workflow is the preferred workflow.  

Without a workflow that ensures a "single source of authority" for each section of their configuration, the "configure exclusive" workflow is preferred.   

In both scenarios, the automation should handle the conflicts.   
