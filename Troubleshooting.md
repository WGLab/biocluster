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

