# The examples do not yet match the contents of the readme, I will work on this when I have time.

# BOSH Splitter

## Overview

BOSH Splitter is a tool to help with large BOSH deployments which have a high instance count for a subset of jobs.  Cloud Foundry DEA runners and Diego Cells are jobs which in large deployments scale above 100 instances.  BOSH Splitter works by carving a single large deployment into multiple smaller deployments.

## Reasons why it is needed

Updating large deployments of Cloud Foundry (400+ DEAs) will result in 36+ hour BOSH deployments, especially if stemcells also need to be upgraded.  Breaking a single deployment into multiple smaller deployments is easier for Ops to manage.

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
 5. Deployment of manifest.yml with the new reserved range of ips
 6. Deployment of each of the runner manifests
 7. Deployment of the core manifest

### Step 1 - Creation of `split` script

This is a one time activity and has already been done in this repo.  If BOSH Splitter is desired in other deployment repos simply copy it and place it into the `bin/` folder and make it executable.

A few notes about its execution:
 - It requires command line parameters to be passed to it.  See the `Makefile` as an example.  It should likely only be run from a `make refresh manifest split` command.
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

#TODO: Make this match the example in this repo  
The networking groups need to be split.  One new networking group needs to be added for each job which is being split out.  Each network cannot overlap the available range of floating ips.

In `useast1-sb` networks cf1 and cf2 were modified to only use the first 256 ips in the original ranges (added to `reserved:`).  Each had a /23 network assigned as the range.  `cf1_runner`, and `cf2_runner` were created by wholesale copying the corresponding `cf1` and `cf2` and adding to `reserved:` so `cf*_runner` used ips 257 - 385 and `cf*_runner_small` used ips 386 - 512.

Leveraging BOSH v2 may get around this restriction.


### Step 4 - Create the core and runner manifests

Run the BOSH Splitter from the `<site>/<env>` folder in the deployment:
```
make refresh manifest split
```

Note that `split` should be run AFTER `manifest` as the split script works by parsing `manifest.yml`

### Step 5 - Deployment of manifest.yml

In the example of useast1-sb networks cf1 and cf2 were modified to reuse IPs within the /23 network.  To force all deployments into this IP range run:
```
bosh deployment manifest/manifest.yml
bosh deploy
```

On useast1-sb this just happened to be a no-op as no IPs had been used outside of the new IP range.  If this had not been the case any offending instances would have been recreated.

### Step 6 - Deployment of each of the runner manifests

Now we can deploy the new runner deployments.  Because `useast1-sb` is small and there are plenty of resources and IPs available we were able to deploy:
 - split_runner_small_z1.yml - manually modify yml and change the network from cf1 to cf1_runner_small
 - split_runner_small_z2.yml - manually modify yml and change the network from cf2 to cf2_runner_small
 - split_runner_z1.yml - manually modify yml and change the network from cf1 to cf1_runner
 - split_runner_z2.yml - manually modify yml and change the network from cf2 to cf2_runner

Once the files were edited each was deployed:
```
bosh deployment manifests/split_runner_small_z1.yml
bosh deploy
bosh deployment manifests/split_runner_small_z2.yml
bosh deploy
bosh deployment manifests/split_runner_z1.yml
bosh deploy
bosh deployment manifests/split_runner_z2.yml
bosh deploy
```

Once deployed you will see the 4 new deployments for the runners:
```
cweibel@sw-jumpbox:~/projects/cloud-foundry-deployments/useast1/sandbox/manifests$ bosh deployments

+-----------------------------------------+------------------------+-------------------------------------------------+
| Name                                    | Release(s)             | Stemcell(s)                                     |
+-----------------------------------------+------------------------+-------------------------------------------------+
| useast1-sb-cloudfoundry                 | buildpacks/2           | bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.14 |
|                                         | cf-sensu-client/1.7.12 |                                                 |
|                                         | cf/243                 |                                                 |
|                                         | shield/6.3.3           |                                                 |
|                                         | toolbelt/3.2.10        |                                                 |
+-----------------------------------------+------------------------+-------------------------------------------------+
| useast1-sb-cloudfoundry-runner_small_z1 | buildpacks/2           | bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.14 |
|                                         | cf-sensu-client/1.7.12 |                                                 |
|                                         | cf/243                 |                                                 |
|                                         | shield/6.3.3           |                                                 |
|                                         | toolbelt/3.2.10        |                                                 |
+-----------------------------------------+------------------------+-------------------------------------------------+
| useast1-sb-cloudfoundry-runner_small_z2 | buildpacks/2           | bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.14 |
|                                         | cf-sensu-client/1.7.12 |                                                 |
|                                         | cf/243                 |                                                 |
|                                         | shield/6.3.3           |                                                 |
|                                         | toolbelt/3.2.10        |                                                 |
+-----------------------------------------+------------------------+-------------------------------------------------+
| useast1-sb-cloudfoundry-runner_z1       | buildpacks/2           | bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.14 |
|                                         | cf-sensu-client/1.7.12 |                                                 |
|                                         | cf/243                 |                                                 |
|                                         | shield/6.3.3           |                                                 |
|                                         | toolbelt/3.2.10        |                                                 |
+-----------------------------------------+------------------------+-------------------------------------------------+
| useast1-sb-cloudfoundry-runner_z2       | buildpacks/2           | bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.14 |
|                                         | cf-sensu-client/1.7.12 |                                                 |
|                                         | cf/243                 |                                                 |
|                                         | shield/6.3.3           |                                                 |
|                                         | toolbelt/3.2.10        |                                                 |
+-----------------------------------------+------------------------+-------------------------------------------------+
```

Looking at one of the new deployments you will see the runners:
```
cweibel@sw-jumpbox:~/projects/cloud-foundry-deployments/useast1/sandbox/manifests$ bosh vms useast1-sb-cloudfoundry-runner_small_z1

+----------------------------------------------------------+---------+-----+-----------------+------------+
| VM                                                       | State   | AZ  | VM Type         | IPs        |
+----------------------------------------------------------+---------+-----+-----------------+------------+
| runner_small_z1/0 (ce5607c9-df72-4ce0-8aaa-438ee1012a53) | running | n/a | runner_small_z1 | 10.72.69.0 |
| runner_small_z1/1 (5910e60f-33f3-43f5-85c0-035b13f52170) | running | n/a | runner_small_z1 | 10.72.69.1 |
+----------------------------------------------------------+---------+-----+-----------------+------------+

VMs total: 2
```
At this point you will have twice the runner capacity as compared to Step 1, hence step 7 below.

### Step 7 - Deployment of the core manifest

Now that there is new runner capacity added via the 4 new deployments, the original deployment `useast1-sb-cloudfoundry` can have the runners removed from it.  This is accomplished by deploying the `core.yml` which is the original `manifest.yml` with the runners removed from it:

```
cweibel@sw-jumpbox:~/projects/cloud-foundry-deployments/useast1/sandbox$ bosh deployment manifests/core.yml

Deploying
---------
Are you sure you want to deploy? (type 'yes' to continue): yes

Director task 273449
  Started preparing deployment > Preparing deployment. Done (00:00:01)

  Started preparing package compilation > Finding packages to compile. Done (00:00:00)

  Started deleting unneeded instances
  Started deleting unneeded instances > runner_small_z1/0 (c13c59be-0295-4099-9db8-caa640600504)
  Started deleting unneeded instances > runner_z2/4 (0f5c04b1-cdb6-4d59-a8dd-786e9247179b)
  Started deleting unneeded instances > runner_small_z2/0 (784eb1a6-cf4a-43b7-a269-76d918f46f61)
  Started deleting unneeded instances > runner_small_z1/1 (cf320579-79f3-4a12-871a-689ee63f866c). Done (00:04:36)
  Started deleting unneeded instances > runner_z2/3 (666786d8-08d1-42d9-8b00-5f28d457870e)
     Done deleting unneeded instances > runner_small_z2/0 (784eb1a6-cf4a-43b7-a269-76d918f46f61) (00:06:44)
  Started deleting unneeded instances > runner_z1/0 (6a38fa4e-5a22-4514-a759-d37103227316)
     Done deleting unneeded instances > runner_small_z1/0 (c13c59be-0295-4099-9db8-caa640600504) (00:06:44)
  Started deleting unneeded instances > runner_z1/1 (30abc1ed-7f5c-4598-8f5c-4d4720476483)
     Done deleting unneeded instances > runner_z2/4 (0f5c04b1-cdb6-4d59-a8dd-786e9247179b) (00:06:44)
  Started deleting unneeded instances > runner_z1/2 (0c7dce0e-2306-4400-8529-c81bf2cc9fdf)
     Done deleting unneeded instances > runner_z2/3 (666786d8-08d1-42d9-8b00-5f28d457870e) (00:04:56)
  Started deleting unneeded instances > runner_z1/3 (773fd180-53f7-4fb6-a7ba-a3d2b5c75eaa)
     Done deleting unneeded instances > runner_z1/0 (6a38fa4e-5a22-4514-a759-d37103227316) (00:04:54)
  Started deleting unneeded instances > runner_z2/0 (be6626d4-26f8-4f49-93a1-b50d1e41a83d)
     Done deleting unneeded instances > runner_z1/1 (30abc1ed-7f5c-4598-8f5c-4d4720476483) (00:04:55)
  Started deleting unneeded instances > runner_small_z2/1 (3a0a2db1-e650-49a9-947f-ab2345973b83)
     Done deleting unneeded instances > runner_z1/2 (0c7dce0e-2306-4400-8529-c81bf2cc9fdf) (00:06:57)
  Started deleting unneeded instances > runner_z2/2 (41ed63d7-1390-4e29-b2cf-f2200106d502)
     Done deleting unneeded instances > runner_z1/3 (773fd180-53f7-4fb6-a7ba-a3d2b5c75eaa) (00:08:10)
  Started deleting unneeded instances > runner_z1/4 (3c9d9557-99dc-400c-bae0-72dc379481b6)
     Done deleting unneeded instances > runner_z2/0 (be6626d4-26f8-4f49-93a1-b50d1e41a83d) (00:06:56)
  Started deleting unneeded instances > runner_z2/1 (d001089c-2533-4d6c-8bda-e0e0b32a4a44)
     Done deleting unneeded instances > runner_small_z2/1 (3a0a2db1-e650-49a9-947f-ab2345973b83) (00:08:59)
     Done deleting unneeded instances > runner_z2/2 (41ed63d7-1390-4e29-b2cf-f2200106d502) (00:06:59)
     Done deleting unneeded instances > runner_z2/1 (d001089c-2533-4d6c-8bda-e0e0b32a4a44) (00:07:00)
     Done deleting unneeded instances > runner_z1/4 (3c9d9557-99dc-400c-bae0-72dc379481b6) (00:08:18)
     Done deleting unneeded instances (00:26:00)

Task 273449 done

Started		2016-11-01 13:32:20 UTC
Finished	2016-11-01 13:58:30 UTC
Duration	00:26:10
```

An important note is you can see the runners gracefully shut down and apps are moved to the new deployments' runners.


### Additional tasks which would need to be done to productionalize this deployment
- Modify `useast1/site/jobs.yml` for each runner_* job set the new network group so the changes are available to all deployments within the site
- Modify `deploy-useast1-sandbox.yml` to call `bin/split` and create 4 new tasks to deploy each of the runner manifests
- Modify each new runner task to run only after successful running of `deployment-useast1-sandbox`
- Modify `deploy-useast1-sandbox.yml` to call `bin/split` and deploy `core.yml` instead of `manifest.yml`


### Rollback

To revert these changes and go back to a single deployment and manifest:
 - Undo the network changes done in Step 3
 - Run `make refresh manifest` to create a new manifest with the original reserved ip ranges
 - Run `bosh deployment manifest/manifest.yml; bosh deploy` to redeploy the original runners
 - Once the previous step is done, perform `bosh delete deployment` for each of the runner manifests


## Diego deployment recommendations

It may be impractical to deploy the splitter against the existing DEAs, however as the Diego deployment is not fully utilized:
 - Perform the split *before* scaling out Diego cells
 - Allocate a new /23 network for each new cell grouping and preserve the existing range for the database, brain, cc_bridge, route_emitter and access jobs
 - Merge `cloud-foundry-deployments` with `diego-deployments` repos as more of the components, such as etcd, are shared and DEAs are deprecated.
 - Explore converting conversion to BOSH v2 so the networking does not need to be carved into groups to prevent deployments from having overlapping IP ranges.

## Gotchas

The following pain points were identified while creating the PoC with useast1-sb:
 - The existing two networks (`cf1` and `cf2`) networks needed to be split into 6, leveraging the `reserved:` to carve out groups of ip ranges which did not overlap.  This is needed because otherwise each deployment will try and use the same ip ranges and fail.  If there are ips overlapping any of the split groups the vms will get recreated.
 - Before removing the existing jobs (runners or otherwise) the new deployments of runners have to be created.  Otherwise you may wind up in a situation where there are not enough resources to run apps.
 - For some fixed sized infrastructures there may not be enough physical resources or IPs to have double the number of runners required after creating all the runner deployments but before deploying `core.yml` removing the runners form the original deployment.
