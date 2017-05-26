![Logo](images/bosh-splitter-logo.png)

# BOSH Splitter

## Overview

BOSH Splitter is a tool to help with large BOSH deployments which have a high instance count for a subset of jobs.  Cloud Foundry DEA Runners and Diego Cells are jobs which in large deployments scale above 100 instances.  BOSH Splitter works by carving a single large deployment into multiple smaller deployments.

## Reasons why it is needed

Updating large deployments of Cloud Foundry (500+ DEAs) will result in 36+ hour BOSH deployments, especially if stemcells also need to be upgraded.  Breaking a single deployment into multiple smaller deployments is easier for Ops to manage.

The tool also prevents the need for making and maintaining multiple deployment repos and preserves creation of `manifest.yml` to allow for rollback.

## Prerequisites

You need to have the following:
 - [Spruce 1.8.1](https://github.com/geofffranks/spruce) or newer
 - A [Genesis](https://github.com/starkandwayne/genesis) style deployment architecture

## Example usage

The Cloud Foundry deployment in `useast1-sb` was modified to extract each group of runners into its own deployment.  Implementation was done in the following order:
 1. Creation of the `split` bash script
 2. Modification of the environment level Genesis `Makefile`
 3. Modification of the site level `networking.yml`
 4. Create the core and runner manifests
 5. Deployment of the core manifest
 6. Deployment of each of the runner manifests

### Step 1 - Creation of `split` script

This is a one time activity and has already been done in this repo.  If BOSH Splitter is desired in other deployment repos simply copy it and place it into the `bin/` folder and make it executable.

A few notes about its execution:
 - It requires command line parameters to be passed to it.  See the `Makefile` as an example.
 - A temporary folder and yml files will be created in `<site>/<env>/scripts` and is removed at the end of the script execution.  If the script errors out these files may be left behind and can be ignored as they will be removed on the next successful execution of the script.
 - `manifest.yml` is used as the source of the split and should be updated before running BOSH Splitter.
 - The script takes the list of jobs to split out of the manifest.yml as command line arguments.  Each job specified will have its own deployment manifest file created.
 - When BOSH Splitter is executed the following files are created in the `manifests/` folder:
   - core.yml - Is the `manifest.yml` with the jobs specified on the command line removed.  It preserves the deployment name in manifest.yml
   - split_{job_name}.yml - This is the `manifest.yml` with all but a single job removed.  The job name is appended to the deployment name in manifest.yml so when BOSH deployed a new deployment is created.

### Step 2 - Modify Makefile

This step needs to be performed for each environment where BOSH Splitter will be used.  Modify the Makefile and add a space delimited list of job names you would like stripped out of manifest.yml into their own deployment manifest files.  A `split` task also needs to be added.  In the `useast1/prod` folder a small Makefile already exists which can be modified to your own needs.

```
...
SPLIT_JOBS := "runner_z1 runner_z2"
...
split:
	@../../bin/split "$(BUILD_SITE)" "$(BUILD_ENV)" "$(TO)" "$(SPLIT_JOBS)"
...
```


### Step 3 - Site Level Networking Changes

The networking groups need to be split.  One new networking group needs to be added for each job which is being split out.  Each network cannot overlap the available range of floating ips.

In the example for `useast1/prod` additional networks were added.  One additional network was added for `runner_z1` and `runner_z2` with a different IP range than `cf1` and `cf2`.

Leveraging BOSH v2 may get around this restriction.


### Step 4 - Create the core and runner manifests

Run the BOSH Splitter from the `<site>/<env>` folder in the deployment:
```
make split
```

Note that `split` should be run AFTER creating the manifest as the split script works by parsing `manifest.yml`

### Step 5 - Deployment of the core manifest

Do deploy Cloud Foundry but without `runner_z1` and `runner_z2` jobs deploy `manifests/core.yml`:

```
bosh deployment manifests/core.yml
bosh deploy
```


### Step 6 - Deployment of each of the runner manifests

Now we can deploy the new runner deployments:
```
bosh deployment manifests/split_runner_z1.yml
bosh deploy
bosh deployment manifests/split_runner_z2.yml
bosh deploy
```

Once deployed you will see the 2 new deployments for the runners:
```
cweibel@sw-jumpbox:~/projects/cloud-foundry-deployments/useast1/sandbox/manifests$ bosh deployments

+-----------------------------------------+------------------------+-------------------------------------------------+
| Name                                    | Release(s)             | Stemcell(s)                                     |
+-----------------------------------------+------------------------+-------------------------------------------------+
| useast1-sb-cloudfoundry                 | buildpacks/2           | bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.14 |
|                                         | cf/251                 |                                                 |
|                                         | shield/6.3.3           |                                                 |
|                                         | toolbelt/3.2.10        |                                                 |
+-----------------------------------------+------------------------+-------------------------------------------------+
| useast1-sb-cloudfoundry-runner_z1       | buildpacks/2           | bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.14 |
|                                         | cf/251                 |                                                 |
|                                         | shield/6.3.3           |                                                 |
|                                         | toolbelt/3.2.10        |                                                 |
+-----------------------------------------+------------------------+-------------------------------------------------+
| useast1-sb-cloudfoundry-runner_z2       | buildpacks/2           | bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.14 |
|                                         | cf/251                 |                                                 |
|                                         | shield/6.3.3           |                                                 |
|                                         | toolbelt/3.2.10        |                                                 |
+-----------------------------------------+------------------------+-------------------------------------------------+
```

Looking at one of the new deployments you will see the runners:
```
cweibel@sw-jumpbox:~/projects/cloud-foundry-deployments/useast1/sandbox/manifests$ bosh vms useast1-sb-cloudfoundry-runner_small_z1

+----------------------------------------------------+---------+-----+-----------+------------+
| VM                                                 | State   | AZ  | VM Type   | IPs        |
+----------------------------------------------------+---------+-----+-----------+------------+
| runner_z1/0 (ce5607c9-df72-4ce0-8aaa-438ee1012a53) | running | n/a | runner_z1 | 10.50.69.0 |
| runner_z1/1 (5910e60f-33f3-43f5-85c0-035b13f52170) | running | n/a | runner_z1 | 10.50.69.1 |
+----------------------------------------------------+---------+-----+-----------+------------+

VMs total: 2
```



### Additional tasks which would need to be done to productionalize this deployment
- Modify any existing concourse pipelines to deploy `core.yml` instead of `manifest.yml`
- Modify existing concourse pipelines to add new tasks to deploy each runner group

### Rollback

To revert these changes and go back to a single deployment and manifest:
 - Undo the network changes done in Step 3
 - Perform `bosh delete deployment` for each of the runner manifests
 - Run `bosh deployment manifest/manifest.yml; bosh deploy` to redeploy the original runners


## Diego deployment recommendations

It may be impractical to deploy the splitter against the existing runners or cells of a large deployment, the following is your easiest path forward:
 - Perform the split *before* scaling out Diego cells or DEA runners
 - Allocate a new /23 or similar networks for each new cell grouping and preserve the existing range for the CF core, database, brain, cc_bridge, route_emitter and access jobs
 - Explore converting to BOSH v2 so the networking does not need to be carved into groups to prevent deployments from having overlapping IP ranges.

## Gotchas

The following pain points were identified while creating the PoC with useast1-sb:
 - The existing two networks (`cf1` and `cf2`) networks needed to be split into 6, leveraging the `reserved:` to carve out groups of ip ranges which did not overlap.  This is needed because otherwise each deployment will try and use the same ip ranges and fail.  If there are ips overlapping any of the split groups the vms will get recreated.
 - Before removing the existing jobs (runners or otherwise) the new deployments of runners have to be created.  Otherwise you may wind up in a situation where there are not enough resources to run apps.
 - For some fixed sized infrastructures there may not be enough physical resources or IPs to have double the number of runners required after creating all the runner deployments but before deploying `core.yml` removing the runners form the original deployment.
