# azure-k3s-cluster (WIP)

A (WIP) dynamically resizable [`k3s`][k3s] cluster for Azure, based on my [`azure-docker-swarm-cluster`][adsc] project.

## What

This is an Azure Resource Manager template that automatically deploys a [`k3s`][k3s] cluster atop Ubuntu 18.04. This cluster has a single master VMs and a VM scaleset for workers/agents, plus the required network infrastructure.

The template defaults to deploying B-Series VMs (`B1ls`) with the smallest possible managed disk size (S4, 32GB). It also deploys (and mounts) an Azure File Share on all machines with (very) permissive access.

The key aspect of this template is that _you can add and remove agents at will_ simply by resizing the VM scaleset - the cluster comes with a few (very simple) helper scripts that allow nodes to join and leave the swarm as they are created/destroyed. 

## Why

This was originally built as [a Docker Swarm template][adsc], and even though Azure has a perfectly serviceable Kubernetes managed service, I enjoy the challenge of building my own stuff and fine-tuning it.

[`k3s`][k3s] is a breath of fresh air, and an opportunity to play around with a simpler, slimmer version of Kubernetes--and break it to see what happens.

## Roadmap

* [ ] remove scale set load balancer
* [ ] trivial ingress through master node
* [ ] clean `kubernetes-dashboard` deployment
* [ ] sample deployments/pods/charts
* [ ] install `k3s` via`cloud-config`
* [ ] change `cloud-config` to expose `k3s` token to agents
* [ ] remove unused packages from `cloud-config`
* [x] remove unnecessary commands from `Makefile`
* [x] remove unnecessary files from repo and trim history
* [x] fork, new `README`

## `Makefile` commands

* `make keys` - generates an SSH key for provisioning
* `make deploy-storage` - deploys shared storage
* `make params` - generates ARM template parameters
* `make deploy-compute` - deploys cluster resources and pre-provisions Docker on all machines
* `make view-deployment` - view deployment progress
* `make list-agents` - lists all agent VMs
* `make scale-agents-<number>` - scales the agent VM scale set to `<number>` instances, i.e., `make scale-10` will resize it (up or down) to 10 VMs
* `make stop-agents` - stops all agents
* `make start-agents` - starts all agents
* `make reimage-agents-parallel` - nukes and paves all agents
* `make reimage-agents-serial` - reimages all agents in sequence
* `make chaos-monkey` - restarts all agents in random order
* `make proxy` - opens an SSH session to `master0` and sets up TCP forwarding to `localhost`
* `make tail-helper` - opens an SSH session to `master0` and tails the `k3s-helper` log
* `make list-endpoints` - list DNS aliases
* `make destroy-cluster` - destroys the entire cluster

## Recommended Sequence

    az login
    make keys
    make deploy-storage
    make params
    make deploy-compute
    make view-deployment
    # Go to the Azure portal and check the deployment progress
    
    # Clean up after we're done working
    make destroy-cluster


## Requirements

* [Python][p]
* The [Azure CLI][az] (`pip install -U -r requirements.txt` will install it)
* GNU `make` (you can just read through the `Makefile` and type the commands yourself)

## Internals

`master0` runs a very simple HTTP server (only accessible inside the cluster) that provides tokens for new VMs to join the swarm and an endpoint for them to signal that they're leaving. That server also cleans up the node table once agents are gone.

Upon provisioning, all agents try to obtain a worker token and join the swarm. Upon rebooting, they signal they're leaving the swarm and re-join it again.

This is done in the simplest possible way, by using `cloud-init` to bootstrap a few helper scripts that are invoked upon shutdown and (re)boot. Check the YAML files for details.

## Provisioning Flow

To avoid using VM extensions (which are nice, but opaque to most people used to using `cloud-init`) and to ensure each fresh deployment runs the latest Docker version, VMs are provisioned using `customData` in their respective ARM templates. 

`cloud-init` files and SSH keys are then packed into the JSON parameters file and submitted as a single provisioning transaction, and upon first boot Ubuntu takes the `cloud-init` file and provisions the machine accordingly.

If instantiation speed is a concern, this can be done once for each role and baked into a `.vhd` file - which then has to be pre-uploaded, and its location supplied to the cluster template. However, it might be much more efficient to just pause unneeded instances and restart them again when necessary.

## Improvements

There are several things that can be done to improve upon this:

* Use the instance metadata endpoint at http://169.254.169.254 to assess instance state and handle [scheduled events](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/scheduled-events)
* Ensure this works with multiple masters (cursory testing suggests it works just fine, although it can be fiddly for agents to re-try connecting to up to 5 possible masters, etc.)
* Strengthen the token exchange mechanism (adding SSL and/or a shared `nonce` to the registration/draining URLs is left as an exercise to the reader)
* Find ways to nudge Swarm into re-balancing the load between nodes (there are already multiple approaches for this in the [Docker][d] issue list - (re)tagging might be the simplest)
* Stop instances and leave their resources allocated instead of destroying them completely upon rescaling, for faster scale-ups
* <strike>Turn on CPU-based auto-scaling in Azure (again, it's off largely because this is simpler to understand)</strike>

## Disclaimers

Keep in mind that this was written for conciseness and ease of understanding -- you can use this as the basis for rolling out a production environment, but _only_ after adding some error-checking.

The [Docker][d] way of achieving cluster self-provisioning relies on service-specific containers baked into their cloud images (and does not seem to allow for dynamically adding or removing nodes), so the approach in this repo is not canon - but it might be more interesting (and easier to expand upon) for people learning how to use [Docker][d] Swarm. 

Also, keep in mind that the load-balancer configuration does _not_ include TCP port probing or proactive failure detection.

[k3s]: https://k3s.io
[adsc]: https://github.com/rcarmo/azure-docker-swarm-cluster
[d]: http://docker.com
[p]: http://python.org
[dh]: https://hub.docker.com/r/rcarmo/demo-frontend-stateless/
[az]: https://github.com/Azure/azure-cli
