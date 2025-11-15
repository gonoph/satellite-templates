# satellite-templates
a list of custom satellite templates to help with ImageMode provisioning and other custom things

## How to use these

At a high level:

1. Create a custom OS called ImageMode #major#.#minor# (example ImageMode 10.0)
2. Synchronize these templates into your Satellite.
3. Update the default templates for the OS.
4. Assign parameter `imagemode_image` to point to the container image in the
   registry. You can update this on a host by host basis, too.
5. Also configure some optional parameters:
    1. `admin_users` - space delimited list of users to create and add to sudoers
    2. `custom_adhoc_post_command` - shell type script to run as part of %post
    3. `host_registration_insights` - (true) register insights as part of registration
    4. `only_subscription_manager_repos` - (false) don't fiddle with repos
    5. `redhat_install_host_tools` - (false) don't install anything
    6. `redhat_install_host_tracer_tools` - (false) don't install anything
    7. `subscription_manager_auto_attach` - (false) don't attach any subs
6. Create a new host, select Image Mode as your OS, configure it.
7. Boot the host, and watch it provision as an Image Mode system!

Things working:

1. LVM and thin volumes
2. Satellite registration
3. Insights (Lightspeed) registration

Things not working:

* Good error messages when things try and install stuff during the kickstart.
* Rebuilding the system defaults back to RedHat OS, which is package mode.
* Embedding authentation for the image provisioning, nor for runtime. (I didn't use auth in my internal lab)

## Why do you need these templates?

Short answer, as of Satellite 6.18, the templates do not support Image Mode.

Long answer:

### It would be difficult to keep custom changes in sync with the official templates.

There are a few details on how the exisitng Satellite templates are made and
maintained that would interfere with editing them outright.  Additionally, it
wasn't enough to edit one template, things like partitions and PXELinux and
PXEGrub2 needed to be updated to pass information that is auto-detected in a
package mode kickstart, but are not detected in an Image Mode kickstart.

### Package Mode uses a different install path

Technically, Package Mode and Image Mode the same install path, but Image Mode
adds an additional install path to laydown the rpm-ostree.

Package Mode installs everything to `sysimage`.

Image Mode installs a bootstrap to `sysimage`, and then installs rpm-ostree to
`sysroot`.

### Network and DNS resolution

Due to the path issue above, This causes things like network resolution to not
work for the chroot %post sections, since by default the `/etc/resolv.conf` is
installed to `sysimage` like Package Mode, but it really should go inside
`sysroot`.

### Passing the stage2 url

Normally, during the install, you need to pass `inst.repo` or `inst.stage2` to
the kernel command line, so that the dracut based `initrd` ramdisk knows how to
obtain the stage2 installer, unpack it, and run it. However, the dracut
`initrd` has special sauce that auto detects the `inst.repo` setting from the
passed kickstart file. It parses the kickstart and looks at the `url` line to
figure out where to seek the stage2 installer.

In Image Mode, this entry doesn't exist in the kickstart, and the stage2 url
can't be deciphered from the container image. Therefore, you must pass this to
the kernel command line - which is what the PXELinux and PXEGrub2 templates do.

### Image Mode does not populate /home from the container

When you create an Image Mode system from Image Builder, the /home directory is
included in the raw, cloud, or qcow2 image. When you use the kickstart
`ostreecontainer` command, this is not the case. You have to add logic to
refresh / add those user dirs back.

### Credentials and Custom Certificate Authorities

In order to inject credentials and custom CA's into the image, you have to do
it in 2 parts:

1. custom creds and CAs need to be injected into installer image via %pre
2. custom creds and CAs need to be injected into the installed image via %post

### Can't install things during provisioning

Since the Image Mode system gets all of it's applications, packages, and most
files from the source container image, you're not able to install anything new
during provisioning. Any updates need to come from a new container image. 

## Example Ansible

### Define Variables

| variable              | meaning                                   | example                          |
| :-----------------    | :---------------------------------------- | :------------------------------- |
| `imagemode_image`     | will render the name of the container image | registry.lab.example.com/rhel10/workload-vm:10.1 |
| `os_versions`         | the specific OS versions that Image Mode supports | string: 10.1, 10.0, 9.6, 9.5 |
| `partition_tables`    | partition table template that work for Image Mode | Kickstart imagemode partition |
| `architectures`       | Image Mode works on x86_64 and ARM64      | architectures: [ x86_64 ]        |
| `template_parameters` | define some sane defaults that apply to hosts with this OS type | (see below) |

```yaml
imagemode_image: "registry.lab.example.com/rhel{{ os_item | split('.') | first}}/workload-vm:{{ os_item }}"
os_versions:
  - "10.1"
  - "10.0"
  - "9.6"
  - "9.5"
partition_tables:
  - Kickstart imagemode partition
architectures:
  - x86_64
template_parameters:
  - name: imagemode_image
    parameter_type: string
    value: "{{ imagemode_image }}"
  - name: subscription_manager_auto_attach
    parameter_type: boolean
    value: false
    hidden_value: false
  - name: redhat_install_host_tracer_tools
    parameter_type: boolean
    value: false
    hidden_value: false
  - name: redhat_install_host_tools
    parameter_type: boolean
    value: false
    hidden_value: false
  - name: only_subscription_manager_repos
    parameter_type: boolean
    value: false
    hidden_value: false
  - name: host_registration_insights
    parameter_type: boolean
    value: true
    hidden_value: false
```

## Playbook tasks

```yaml
- name: create ImageMode OS
  theforeman.foreman.operatingsystem:
    server_url: "{{ satellite_host }}"
    username: "{{ satellite_user }}"
    password: "{{ satellite_pass }}"
    name: ImageMode
    architectures: "{{ architectures }}"
    major: "{{ os_item | split('.') | first }}"
    minor: "{{ os_item | split('.') | last }}"
    os_family: Redhat
    state: present_with_defaults
  loop: "{{ os_versions }}"
    loop_var: os_item

# this should auto attach what it can
- name: import custom templates
  theforeman.foreman.templates_import:
    server_url: "{{ satellite_host }}"
    username: "{{ satellite_user }}"
    password: "{{ satellite_pass }}"
    repo: https://github.com/gonoph/satellite-templates.git
    branch: production
    associate: new
  register: import_templates

# this fixes what the import didn't auto attach
- name: update ImageMode OS
  theforeman.foreman.operatingsystem:
    server_url: "{{ satellite_host }}"
    username: "{{ satellite_user }}"
    password: "{{ satellite_pass }}"
    name: ImageMode
    architectures: "{{ architectures }}"
    major: "{{ os_item | split('.') | first }}"
    minor: "{{ os_item | split('.') | last }}"
    os_family: Redhat
    state: present
    parameters: "{{ template_parameters }}"
    ptables: "{{ partition_tables }}"
    provisioning_templates: "{{ provisioning_templates | map(attribute='provisioning_template') | list }}"
  loop: "{{ os_versions }}"
    loop_var: os_item

- name: assign default templates to OS
  theforeman.foreman.os_default_template:
    server_url: "{{ satellite_host }}"
    username: "{{ satellite_user }}"
    password: "{{ satellite_pass }}"
    operatingsystem: "ImageMode {{ os_template_item.0 }}"
    template_kind: "{{ os_template_item.1.template_kind }}"
    provisioning_template: "{{ os_template_item.1.provisioning_template }}"
  loop: "{{ os_versions | product(provisioning_templates) | list }}"
    loop_var: os_template_item
```

# Disclaimer

Last updated: November 14, 2025

The information contained on the Service is for general information purposes
only.

The Author assumes no responsibility for errors or omissions in the contents of
the Service.

## Errors and Omissions Disclaimer

The information given by the Service is for general guidance on matters of
interest only. Even if the Author takes every precaution to ensure that the
content of the Service is both current and accurate, errors can occur. Plus,
given the changing nature of laws, rules and regulations, there may be delays,
omissions or inaccuracies in the information contained on the Service.

The Author is not responsible for any errors or omissions, or for the results
obtained from the use of this information.

## No Responsibility Disclaimer

The information on the Service is provided with the understanding that the
Author is not herein engaged in rendering legal, accounting, tax, or other
professional advice and services. As such, it should not be used as a
substitute for consultation with professional accounting, tax, legal or other
competent advisers.

In no event shall the Author or its suppliers be liable for any special,
incidental, indirect, or consequential damages whatsoever arising out of or in
connection with your access or use or inability to access or use the Service.

## "Use at Your Own Risk" Disclaimer

All information in the Service is provided "as is", with no guarantee of
completeness, accuracy, timeliness or of the results obtained from the use of
this information, and without warranty of any kind, express or implied,
including, but not limited to warranties of performance, merchantability and
fitness for a particular purpose.

The Author will not be liable to You or anyone else for any decision made or
action taken in reliance on the information given by the Service or for any
consequential, special or similar damages, even if advised of the possibility
of such damages.

## Contact Us

If you have any questions or concerns, please open an Issue on the repository.
