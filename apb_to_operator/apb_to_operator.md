Ansible Operator and Ansible Playbook Bundles
=============================================

[Ansible Operator](https://github.com/operator-framework/operator-sdk/) makes it
easy for you to write [Ansible](https://www.ansible.com/) to package operational
knowledge into a Kubernetes application. [Ansible Playbook Bundles
(APBs)](https://github.com/ansibleplaybookbundle/) are lightweight application
definitions designed to make cloud services available to the [Kubernetes Service
Catalog](https://github.com/kubernetes-incubator/service-catalog) through the
[Automation Broker](https://github.com/openshift/ansible-service-broker). This
post will demonstrate a method for taking an existing investment in Ansible, in
the form of an APB, and using it to develop a functional operator using the
[operator-sdk](https://github.com/operator-framework/operator-sdk/).

# Introduction

First, if you are reading this and are not aware of the `k8s` [Ansible
Module](https://docs.ansible.com/ansible/latest/modules/k8s_module.html) you
should take a look. This module, introduced in Ansible 2.6, changes the game
with respect to managing Kubernetes objects in Ansible. [This post on the
Ansible
blog](https://www.ansible.com/blog/dynamic-kubernetes-client-for-ansible)
introduces the `k8s` module and the [dynamic
client](https://github.com/openshift/openshift-restclient-python). Simply put,
in my opinion, if you are interacting with Kubernetes objects in Ansible and are
not using the `k8s` module, you are doing it wrong.

The Automation Broker, often referred to as the Openshift Ansible Broker, and the
APBs it relies on came from a desire to make exposing an application to the
Service Catalog easier for application developers. Instead of a developer having
to make your own Open Service Broker API compatible broker, you wrote some
Ansible, added some metadata to your container image and the Automation Broker
took care of the hard parts. In many of the same ways the Ansible Operator
allows you to write some Ansible, tell the Ansible Operator what [Custom
Resources](https://kubernetes.io/docs/concepts/api-extension/custom-resources/)
to watch, and allow the Ansible Operator to worry about reconciliation and
garbage collection.

Developing a functional operator from an existing APB project is straightforward
and this post will show you how. While the method shown here will not work out
of the box for every possible APB project layout, it will put on display what Ansible
Operator requires to run properly in a way that could be adapted to any APB when
making an operator. This post I will:

1. Clone a simple APB project,
   [hello-world-apb](https://github.com/ansibleplaybookbundle/hello-world-apb)
   and use `operator-sdk new --type=ansible` to get the essential pieces for an
   Ansible Operator.
1. Take the essential generated Ansible Operator components and incorporate
   them into the hello-world-apb project.
1. Demonstrate that operator running and provisioning our hello-world
   application in a Kubernetes cluster.

# Hello World

Here, I will start with an APB project and use the `operator-sdk` to generate
the necessary parts for a functional Ansible Operator. If you haven't noticed
by now, this post is really just a clever attempt to get you to try
`operator-sdk`; maybe not all that clever.

**Pre-Requisites**

1. Operator SDK
  * You can grab the binary on [the project's releases
   page](https://github.com/operator-framework/operator-sdk/releases) or follow the
   [quick start](https://github.com/operator-framework/operator-sdk#quick-start) to
   install the latest from source.
1. Ansible
  * See their [installation
    guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
    if you do not already have Ansible installed.

Clone the hello-world-apb:

```shell
$ git clone https://github.com/ansibleplaybookbundle/hello-world-apb.git
```

Then run `operator-sdk new`:

```shell
# From inside the hello-world-apb project is best
$ operator-sdk new --type=ansible hello-world-operator \
                  --api-version=examples.djzager.io/v1alpha1 \
                  --kind=HelloWorld \
                  --skip-git-init
INFO[0000] Creating new Ansible operator 'hello-world-operator'.
INFO[0000] Create build/Dockerfile
INFO[0000] Create watches.yaml
INFO[0000] Create /tmp/osdk583910079/galaxy_init.sh
INFO[0000] Create deploy/service_account.yaml
INFO[0000] Create deploy/role.yaml
INFO[0000] Create deploy/role_binding.yaml
INFO[0000] Create deploy/operator.yaml
INFO[0000] Create deploy/crds/examples_v1alpha1_helloworld_crd.yaml
INFO[0000] Create deploy/crds/examples_v1alpha1_helloworld_cr.yaml
INFO[0000] Running galaxy-init.
Initializing role skeleton...
- HelloWorld was created successfully
- INFO[0000] Project creation complete.
```

# The Essentials for Ansible Operator

Now that I have all the essential pieces for Ansible Operator, I will
incorporate them into my project. Before I jump in, I want to take a moment to
call out the important files generated by the `operator-sdk new --type=ansible`
command that I ran:

* build/Dockerfile - how to build the operator container image
* deploy/operator.yaml - operator deployment, the operator itself run as a
  deployment in Kubernetes
* deploy/(role|service_account|role_binding).yaml - RBAC for the operator
  deployment
* deploy/crds/. - the custom resource definition the operator will be watching
  and an example custom resource
* watches.yaml - tell the operator what to watch and what Ansible to run

## Incorporate Ansible Operator into APB Project

This is my project after running `operator-sdk new`:

```shell
$ tree -L 2
.
├── apb.yml
├── defaults
│   └── main.yml
├── Dockerfile
├── hello-world-operator
│   ├── build
│   ├── deploy
│   ├── roles
│   └── watches.yaml
├── Makefile
├── meta
│   └── main.yml
├── playbooks
│   ├── deprovision.yml
│   ├── provision.yml
│   └── test.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
│   ├── deployment.yml.j2
│   ├── route.yml.j2
│   └── service.yml.j2
└── vars
    └── main.yml
```

Move the operator files into the project, throwing away the rest:

```shell
# First the Dockerfile
$ mv hello-world-operator/build .

# Then the deploy directory
$ mv hello-world-operator/deploy .

# Then the watches file
$ mv hello-world-operator/watches.yaml .

# Throw away the rest
$ rm -r hello-world-operator
```

The project now looks like:

```shell
$ tree -L 2
.
├── apb.yml
├── build
│   └── Dockerfile
├── defaults
│   └── main.yml
├── deploy
│   ├── crds
│   ├── operator.yaml
│   ├── role_binding.yaml
│   ├── role.yaml
│   └── service_account.yaml
├── Dockerfile
├── Makefile
├── meta
│   └── main.yml
├── playbooks
│   ├── deprovision.yml
│   ├── provision.yml
│   └── test.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
│   ├── deployment.yml.j2
│   ├── route.yml.j2
│   └── service.yml.j2
├── vars
│   └── main.yml
└── watches.yaml
```

## Make the APB an Ansible Operator

With the generated files from `operator-sdk` in place, it is time to make
a functional operator. This will have the ancillary benefit providing a deep
dive of sorts into the important pieces of Ansible Operator.

### Ansible Operator is Watching

From the [Ansible Operator User
Guide](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md#watches-file):

> The Watches file contains a list of mappings from custom resources,
> identified by it's Group, Version, and Kind, to an Ansible Role or Playbook.

When I initially ran `operator-sdk new`, I said nothing about the arguments
provided.

* `api-version` is a combination of the Group and Version
  (`examples.djzager.io/v1alpha1`)
* `kind` is just that, `HelloWorld` in my case

Since I am working with an Ansible **Playbook** Bundle, I will map the
custom resource to an Ansible Playbook in my watches file:

```
---
- version: v1alpha1
  group: examples.djzager.io
  kind: HelloWorld
  playbook: /opt/ansible/operator.yml
```

### The Operator Playbook

Here is my operator playbook:

```yaml
---

- name: hello-world-apb operator
  hosts: localhost
  gather_facts: false
  connection: local
  vars:
    apb_action: provision
    app_name: "{{ meta.name }}"
    namespace: "{{ meta.namespace }}"
  roles:
  - role: ansibleplaybookbundle.asb-modules
  - role: hello-world-apb
```

Where do `meta.name` and `meta.namespace` come from? Extravars given to the
`ansible-playbook` during reconciliation come from the Custom Resource.
For exmaple a HelloWorld resource like:

```yaml
apiVersion: examples.djzager.io/v1alpha1
kind: HelloWorld
metadata:
  name: example-helloworld
  namespace: default
spec:
  size: 3
```

The structure of extravars would be:

```json
{
  "meta": {
    "name": "example-helloworld",
    "namespace": "default"
  },
  "size": 3,
  "_examples_djzager_io_helloworld": {
     <Full CR>
   }
}
```

I need to update my Ansible to make use of `size`:

```shell
# Add a default size
$ echo 'size: 1' >> defaults/main.yml

# Update the deployment(config) replica count
$ sed 's|replicas: 1|replicas: "{{ size }}"|' templates/deployment.yml.j2
```

Awesome work. Now all I need to do is configure my operator container image.

### Ansible Operator Container Image

Here is my Dockerfile:

```
FROM quay.io/water-hole/ansible-operator

# This APB has a dependency
RUN ansible-galaxy install ansibleplaybookbundle.asb-modules

# The operator playbook is expecting a hello-world-apb role
COPY . ${HOME}/roles/hello-world-apb

# Put the operator playbook where it belongs in the container image
COPY playbooks ${HOME}/

# The watches file goes in the home directory
COPY watches.yaml ${HOME}/watches.yaml
```

# Conclusion

That's it. I have made a functional Ansible Operator using the `operator-sdk`
with an existing APB project. To recap:

1. With `operator-sdk new --type=ansible` I was able to generate the necessary
   pieces for an Ansible Operator.
1. I moved those pieces into my APB project in such a way that `operator-sdk`
   can be used when building the operator (more on this next) or adding more
   Custom Resources.
1. I configured `watches.yaml`, added an `operator.yml` playbook, and made the
   number of replicas configurable via `size`.
1. I updated the operator Dockerfile based on the structure of my project.

# Next Steps

* Use the `operator-sdk build` command to build an operator image.
* Ansible Playbook Bundles have parameter validation via `apb.yml` and the
  Service Catalog. There is nothing preventing a user from trying to create a
  HelloWorld resource with `size: lol`. It would be a good idea to add
  validation in our Ansible.
* In this incredibly simple example there was no need for multiple custom
  resources. In your project, you may want to use `operator-sdk new crd` and
  the watches file to allow your operator to manage more Custom Resources.
* Check out the [Operator Lifecycle
  Manager](https://github.com/operator-framework/operator-lifecycle-manager/).

I hope that you will give `operator-sdk` and the Ansible Operator a try. To
learn more, head over to the Operator SDK documentation and work through the
[Getting Started Guide](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md)
for Ansible-based Operators. Then join us on the [Operator Framework mailing
list](https://groups.google.com/forum/#!forum/operator-framework) and let us
know what you think.
