## Re-install nodes

When nodes are in error state, it is time to re-install.

Use 

```
rocks list host boot
```

to check the reinstall status if restarting the node. Generally, it should be "os" for all nodes.

Then use

```
rocks set host boot compute-0-0 action=install
```

to change the action to re-installation of Rocks system in the node.

Then if the machine is accessible, log into the machine and type `reboot`. If not accessible, just restart the machine via IPMI or manually by hand.

Alternatively, if the machine is accessible, you can just login and use `/boot/kickstart/cluster-kickstart` to reboot and re-install the node.

## When re-installation does not work

Sometimes a node fails, and you specify to re-install the node and forced to restart the computer. However, the computer displays various error messages, such as "DHCP -/|\" hanging, or "hard drive parameter not found", or "no hard drive found". Sometimes, during Rocks install, the message "loading vmlinz...................." shows up and then freezes, or the "waiting for hardware to initialize" message that freezes.

I found that all these problems were due to hard drive issues. After switching hard drive, all the issues were resolved.

