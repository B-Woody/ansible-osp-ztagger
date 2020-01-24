Ansible OSP Z-Tagger
=========

Ansible OSP Z-Tagger is an Ansible playbook and role which can help map Red Hat Openstack container image tags to official Red Hat Z releases of Openstack and then tag the container images in a local container registry with a label naming the Z release that it belongs to.

Doing this can assist in cases where a previous Z release of OpenStack needs to be used or tested. 

The play essentially automates something like the following for each container:
```
skopeo copy docker://registry.access.redhat.com/rhosp13/openstack-heat-engine:13.0-91 docker://local-registry.example.com/rhosp13/openstack-heat-engine:Z9
```

This was initially written for use in a larger set of Ansible plays before being repurposed to run independently. The idea was loosely based on the ["m4r1k/z-mapper" BASH scripts](https://github.com/m4r1k/z-mapper). If you're looking for a simple list of mappings without the requirement to automate the tagging then the BASH scripts will be more suitable.

Included in the roles `files/` directory is a sample list of common Openstack container images and version tags current as of the Z10 release. A list can be recreated for your Openstack environment by using the output of an `openstack container image prepare` command that is run on the undercloud if the `{{ gather_list }}` variable is set to true and other role variables are modified to allow access to your undercloud. Alternatively you can supply your own list in the same format and each container image will be inspected then tagged with a Z release. 

The list is in the following format (example):
```
registry.access.redhat.com/rhosp13/openstack-aodh-api:13.0-86
registry.access.redhat.com/rhosp13/openstack-aodh-api:13.0-94
registry.access.redhat.com/rhosp13/openstack-aodh-api:13.0-98
registry.access.redhat.com/rhosp13/openstack-aodh-evaluator:13.0
registry.access.redhat.com/rhosp13/openstack-aodh-evaluator:latest
```

For container inspection the shell module will run a skopeo command instead of using the `podman_container_facts` or `docker_container_facts` Ansible modules. This is because according to the modules man pages *"If an image does not exist locally, it will not appear in the results."* In these use cases we may want to inspect and then tag a bunch of remote container images before having to pull them so skopeo is used via shell module.


Requirements
------------

- System must have `skopeo inspect` installed for image inspection and tagging
- System must have OpenStack command line tools installed if new container image list is generated.
- Variables must be modified to allow access to your undercloud if new container image list is generated.

Variables
--------------

Vars will need to be modified in `roles/container-z-release/vars/main.yml` to suit your environment particularly if a new container image list is generated.

The `{{ gather_list }}` variable is set to false by default.

The registry to inspect and destination registry will need to be set in `roles/container-z-release/vars/main.yml`

Variables containing the dates of each Openstack Z release are kept in the bottom section of `roles/container-z-release/vars/main.yml`

Z Release Dates
---------------
The dates used in the abovementioned var file are based on official release notes at https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html/release_notes/index

There is no documentation stating which versions of container images shipped with each Z release of OpenStack so a modified version of the logic in [m4r1k/z-mapper](https://github.com/m4r1k/z-mapper) was used after some testing.

The original [m4r1k/z-mapper](https://github.com/m4r1k/z-mapper) checks that the `Created` timestamp is 7 days before or 7 days after the release date and anything outside those days was called an “async release”. The lists of container images that are output by the script (and this play) were sometimes incomplete when using that logic. It is assumed each Z release would require all images be updated because the underlying RHEL container image layers would be updated with security errata and bugfixes.

By checking a wider scope backwards for container image `Created` dates from the official Z release date (20 days instead of 7) I was able to notice some patterns where majority of containers were built 10-14 days prior to a Z release with occasional builds within 3-5 days after the release date. Using this method appears to give more complete container image lists in the testing that I've done.

Caveats and Known Issues
----------------

- This was originally written for use in a particular environment and may take some tinkering to get working elsewhere if you plan on generating a container image list from your undercloud.

- Doing some of the string, data and datetime manipulation stuff in Ansible got out of hand and the code looks quite ugly and hard to read. If anyone knows of ways to simplify it then a PR is welcomed and encouraged.

- The mapping of Z release to container image tags is a guess based on my testing using the best available information and may not always be accurate.

- Iterating through each item in the container image list to do a `skopeo inspect` can take a **LONG** time. If you're reaching out to an external registry it might be worth copying that variable to a local file for re-use in case something goes wrong and you need to start again. 