+++
title = "A brief introduction to the expose and manage PCI device reset project"
date = 2021-05-25
in_search_index = true
draft = false
template = "blog_post.html"
aliases = [""]
+++

## Background
PCI and PCIe devices may support a number of possible reset mechanisms, for example Function Level Reset (FLR)
provided via Advanced Feature or PCIe capabilities, Power Management reset, bus reset, or device specific reset.
Currently the PCI subsystem creates a policy prioritizing these reset methods which provides neither visibility
nor control to userspace. This project would work to expose the reset methods available per device to userspace,
likely via sysfs, and allow a administrative user or device owner to have some ability to manage per device reset
method priorities or exclusions.

This feature aims to allow greater control of a device for use cases as device assignment, where specific device or
platform issues may interact poorly with a given reset method, and for which device specific quirks have not been
developed.

<!-- more -->


## Current Mechanism
The current policy of prioritizing reset methods is,
Device specific reset -> PCIe FLR(Function level reset) -> PCI AF(Advanced Feature) FLR -> Power Management reset -> Slot reset -> Bus reset. </br>
The supported reset mechanisms are probed when device is added:
```
pci_init_capabilities()
	pci_probe_reset_function()
```
Currently there is no way of knowing which reset methods are supported by the device for the userspace.
The reset can be initiated by writing 1 to the `/sys/bus/pci/device/.../reset` sysfs attribute which will
try resetting the device by using reset methods in an order mentioned in above policy.
```
pci_reset_function()
	__pci_reset_function_locked()
		pci_dev_specific_reset()
		pcie_flr()
		pci_af_flr()
		pci_pm_reset()
		pci_dev_reset_slot_function()
		pci_parent_bus_reset()
```

## Proposed Change
The [patch series](https://lore.kernel.org/linux-pci/20210529192527.2708-1-ameynarkhede03@gmail.com/) introduces a new byte array in `struct pci_dev`
to keep track of ordering of device supported reset methods.
Initially when user has not set any preferred reset method or ordering of multiple reset methods,
device supported reset methods are ordered by exisiting policy.
```
 ➜  ~ cat /sys/bus/pci/devices/0000:02:00.0/reset_method
flr,bus
 ```
 Writing any of the supported device reset method enables that method
 ```
➜  ~ echo bus,flr > /sys/bus/pci/devices/0000:02:00.0/reset_method
➜  ~ cat /sys/bus/pci/devices/0000:02:00.0/reset_method
bus,flr                                                                                                                                                                                  ➜  ~
```
Writing empty string to `reset_method` attribute will disable device from being reset while
writing `default` will return to original value. To actually reset the device write
`1` to exisiting `reset` sysfs attribute.
 ```
➜  ~ echo 1 > /sys/bus/pci/devices/0000:02:00.0/reset
```
</br>

## Acknowledgements
During this LFX mentorship period I learned a lot from my mentor Alex who patiently
explained me all the details including small and basic things while encouraging me to try different
approaches. I would also like to thank Raphael for his time to review the patches.
