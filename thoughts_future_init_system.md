# Introduction 

In dicussions on `#freebsd-labs`, we came to the conclusion that a new init system is something we should discuss in length.

Here's my take on this topic:

# Analysis

`rc.d` clearly is an init system.
`systemd` clearly is more than an init system. Some people use the term init system + *middleware*.

The discussion on `#freebsd-labs` continued, resulting in an intermediate conclusion:

> Systemd is great for out-of-the-box solutions for appliances (every `systemd.unit` defined in advance)
> or distributions like Ubuntu / Fedora which want to push Linux on the desktop in terms of ease of use
> (most `systemd.unit`s defined by packages).

Why are people opposed to systemd? (No claim for completeness, please mail me additional points)

- Exisiting work is obsoleted or has to be re-done systemd-style. (existing init scripts, etc.)
- Feature creep scares people off
- Huge learning curve to know about all features, new way of thinking -> time-consuming switch
- Because it uses Linux-specific Kernel APIs
- Fear of journald because it uses a non-plaintext logging format by default

One impression taken away from the discussion: Many systemd opponents look at the current usage of systemd, decide it's not for them and don't play with it further.
This leads to FUD - fear, uncertainty and doubt.

Let's collect the positive features of systemd that seem compelling for the BSDs: (no cleaim for competeness)
 
- General paradigm of **defining state** in contrast to **event-based scripting** for everything
- Watchdog for daemon processes
- Structured logging 
- lib-systemd and the D-bus interface as a higher-level interface for applications & desktop environments
	- libudev as part of an API for getting notifications about device state changes
- Networkd offering dynamism on mobile devices like laptops
- Isolation of daemons via systemd-nspawn: https://wiki.archlinux.org/index.php/Systemd-nspawn

Whether you agree on the items in the list being an actual upside or not, there is no problem in thinking about a more sustainable implementation.

# Architecture 

In the following, the term OS is meant to be Kernel + IPC + Userland. This leaves the exact implementation open. However, for the BSDs one would assume that everything described as OS is part of a BSD release.

We divide the boot process into two sequences: Early boot and late boot. Subsequent operation is referred to as 'operation'. Shutdown phases are the other way around.

##Early Boot
The point of early boot is to get the system into useful state. We assume an initramfs-style boot.
The kernel boots off the initramfs and starts a `setup process` whose sole purpose is to make the system usable.
This involes:

- Mounting the real root filesystems (local / network) to a directory
	- Disk Decryption
- Tell the kernel to start the `event dispatcher` on the real root filesystem (which becomes the new root fs) after `setup process` dies.

##Late Boot, Operation 
The `event dispatcher` is what holds the system together and is the other part of the init system.
The `event dispatcher` has several types of `subscribers`:

- `service manager`s that expose a list of services in a common format.
- `netowrking managers`s that react to changes in the network state
- `logging handler`s that can receive log output from services
- `triggers`s that can subscribe to system events, e.g. to run automated scripts.

At least one `service manager`, `netowrk manager` and `logging handler` must be registered with the event dispatcher. 
`trigger`s may be added or removed at any time.

### Service Manager

A service is something that runs in the background. A service may be running right on the system, in a jail or some other containerization technology. The kernel must have means to identify it, though.

Identification of a service is done via a reference to the same `service_info` object by all processes or threads that are part of the service.
Services processes are spawned by service managers which are privileged to set such `service_info` references. The service process itself may not do that.
The kernel has to make sure that child processes spawned by the service process inherit the `service_info` reference from their parent.
The kernel also has to provide means to list all processes that reference one `service_info`.

A Service Manager has to do the following:

- expose a list of `service_info` it manages to the OS (this is in turn checked by the OS)
- receives system events (which might influence the service management, dependencies, etc)
- start, stop, monitor services
	- because of configuration files like systemd.unit-files stored on disk
	- as a consequence of user interaction
	- as the result of a system event

**Important**: There can be multiple service managers as long as the sets of services they manage are disjoint.
Use the service manager of systemd, use shell scripts + PID files, roll your own. You only have to expose the list of `service_info`.

### Logging Handler 

A logging handler is fed with log output from services.
The `event dispatcher` provides the logging APIs and takes care of the correct mapping from process to service. The event dispatcher then dispatches the log messages to those `log handlers` that have subscribed to them. 

Possible implementations could:

- Store this output on disk
- Send it to a remote log aggregation server
- Anaylyze structured log data and expose metrics to a monitoring system
- Notify a management server about critical events

#### Logging from services

The init system supports both *standard C syslog* as well as *structured logging*.

In addition, one could consider a standardized API for escalation from daemons to get the users attention.

##### Stuctured Logging

Services may log structured data in addition to human-readable strings. This is useful for telemtry when combined with a *Remote Log Aggregation Handler*

The FLOSS community should strive to **standardize** some structured logging API to keep applications portable. As long as this has not happened, the init system shall expose the standard C syslog APIs.

##### Escalation

Daemons may be running on a server or a laptop. However, currently, logs have to be checked for error-level messages which describe the problem.
In particular daemons on dekstop machines should have a standardized way to send a message to the log system that is marked as `REQUIRES IMMEDIATE ATTENTION FROM A HUMAN` or something similar.

A *notification logging handler* may then subscribe to this kind of message and use the desktop environment's specific mechanism to present an alert.

### Networking Manager

This is not referencing some software used in several Linux distros.

The `event dispatcher`assigns network interfaces to a Networking Manager.
The Networking Manager configures the assigned networking device and reacts to changes in connectivity.

Interfaces not assigned to a Networking Manager are considered user-managed.

There might be need for interoperability / communication between Networking Managers, e.g. for a *Bridge Network Manager*.


### Triggers

Last but not least, Triggers provide a generic way to hook into system events, e.g. to run automation scripts, etc.

It should be easy to register & test triggers with the `event dispatcher`.

## Early Shutdown

Service Managers receive a termination event and would most likely stop all services managed by them.
Triggers may trigger.
Networking Managers receive a termination event, close VPN connections, etc.
`Logging Handlers` are stopeed.

Any dependencies on shutdown between triggers, logging handlers and networking must be resolved by sending notifications like "will close network connections", etc.

## Late Shutdown

Unmount filesystems.
Execute halt / reboot / ... command. 


# Implementation

Let's discuss the architecture first.
