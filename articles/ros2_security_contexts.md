---
layout: default
title: ROS 2 Security Contexts
permalink: articles/ros2_access_control_policies.html
abstract:
  This article specifies the integration between security and contexts.
author:  >
  [Ruffin White](https://github.com/ruffsl),
  [Mikael Arguedas](https://github.com/mikaelarguedas)

published: false
categories: Security
---

{:toc}


# {{ page.title }}

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

Original Author: {{ page.author }}


TODO: Some concise overview introduction here

## Concepts

Before detailing the SROS 2 integration of the contexts, the following concepts are introduced.

### Namespaces

Namespaces are a fundamental design pattern in ROS and are widely used to organize and differentiate many types of resources as to be uniquely identifiable; i.e. for topics, services, actions, and node names. As such, the concept of namespaceing is well know and understood by current users, as well as strongly supported with the existing tooling. Namespaces are often configurable at runtime via command line arguments or statically/programmatically via launch file declarations.

Previously, the Fully Qualified Name (FQN) of a node was used directly by a selected security directory lookup strategy to load the necessary key material. However, with the advent of contexts, such a direct mapping of FQN to security artifacts may no longer suffice.

### Contexts

With the advent of ROS 2, multiple nodes may now be composed into one process for improved performance. Previously however, each node would retain it's one to one mapping to a separate middleware participant. Given the non-negligible overhead incurred of multiple participants per process, a concept of contexts was introduced. Contexts permit a many-to-one mapping of nodes to participant by grouping many nodes per context, and one participant per context.

Currently, DDS participants can only utilise a single security identity; consequently the access control permissions applicable to every node mapped to a given context must be consolidated and combined into a single set of security artifacts. As such, additional tooling and extensions to SROS 2 are necessary to support this new paradigm.


## Keystore

With the additional structure of contexts, it’s perhaps best to take the opportunity to restructure the keystore layout as well. Rather than a flat directory of namespaced node security directories, we can push all such security directories down into a designated `contexts` sub-folder. Similarly, private and public keystore materials can also be pushed down into their own respective sub-folders within the root keystore directory. This is reminiscent of the pattern used earlier Keymint [1].


```
$ tree keystore/
keystore
├── contexts
│   └── ...
│       └── ...
├── private
│   ├── ca.csr.pem
│   └── ca.key.pem
└── public
    ├── ca.cert.pem
    ├── identity_ca.cert.pem -> ca.cert.pem
    └── permissions_ca.cert.pem -> ca.cert.pem
```


### `private`

The `public` directory contains anything permissable as public, such as public certificates for the identity or permissions certificate authorities. As such, this can be given read access to all executables. Note that in the default case, both the identity_ca and permissions_ca points to the same CA certificate.

### `public`

The `private` directory contains anything permissable as private, such as private key material for aforementioned certificate authorities. This directory should be redacted before deploying the keystore onto the target device/robot.

### `contexts`

The `contexts` directory contains the security artifacts associated with individual contexts, and thus node directories are no longer relevant. Similar to node directories however, the `contexts` folder may still recursively nest sub-paths for organizing separate contexts.


## Runtime

TODO: Some transition paragraph here about ros launch

### Unqualified context path

For nodes with unqualified context paths, the context directory will subsequently default to the root level context.

``` xml
<launch>
  <node pkg="demo_nodes_cpp" exec="talker"/>
  <node pkg="demo_nodes_cpp" exec="listener"/>
</launch>
```

```
$ tree contexts/
contexts/
├── cert.pem
├── governance.p7s
├── identity_ca.cert.pem -> ../public/identity_ca.cert.pem
├── key.pem
├── permissions_ca.cert.pem -> ../public/permissions_ca.cert.pem
└── permissions.p7s
```

### Pushed unqualified context path

For nodes with unqualified context paths pushed by a namespace, the context directory will subsequently be pushed to the relative sub-folder.

``` xml
<launch>
  <node pkg="demo_nodes_cpp" exec="talker"/>
  <group>
    <push_ros_namespace namespace="foo"/>
    <node pkg="demo_nodes_cpp" exec="listener"/>
  </group>
</launch>
```

```
$ tree --dirsfirst contexts/
contexts/
├── foo
│   ├── cert.pem
│   ├── governance.p7s
│   ├── identity_ca.cert.pem
│   ├── key.pem
│   ├── permissions_ca.cert.pem
│   └── permissions.p7s
├── cert.pem
├── governance.p7s
├── identity_ca.cert.pem
├── key.pem
├── permissions_ca.cert.pem
└── permissions.p7s
```
> Symbolic links suppressed for readability

### Relatively pushed qualified context path

For nodes with qualified context paths pushed by a namespace, the qualified context directory will subsequently be pushed to the relative sub-folder.

``` xml
<launch>
  <group>
    <push_ros_namespace namespace="foo"/>
    <node pkg="demo_nodes_cpp" exec="listener" context="bar"/>
  </group>
</launch>
```

```
$ tree --dirsfirst contexts/
contexts/
└── foo
    └── bar
        ├── cert.pem
        ├── governance.p7s
        ├── identity_ca.cert.pem
        ├── key.pem
        ├── permissions_ca.cert.pem
        └── permissions.p7s
```

### Fully qualified context path

For nodes with fully qualified context paths, namespacs do not subsequently push the relative sub-folder.

``` xml
<launch>
  <group>
    <push_ros_namespace namespace="foo"/>
    <node pkg="demo_nodes_cpp" exec="listener" context="/bar"/>
  </group>
</launch>
```

```
$ tree --dirsfirst contexts/
contexts/
└── bar
    ├── cert.pem
    ├── governance.p7s
    ├── identity_ca.cert.pem
    ├── key.pem
    ├── permissions_ca.cert.pem
    └── permissions.p7s
```


## Alternatives

### Context path orthogonal to namespace

An alternative to reusing namespaces to hint the context path could be to completely disassociate the two entirely, treating the context path as it's own unique identifier. However, having to book keep both identifier spaces simulations may introduce to many degrees of freedom that a human could groc or easily introspect via tooling.

#### `<push_ros_namespace namespace="..." context="foo"/>`

TODO: Describe added `context` attribute to `push_ros_namespace` element.
Keeps pushing contexts close/readable to pushing of namespaces.

#### `<push_ros_context context="foo"/>`

TODO: Describe added `push_ros_context` element.
Keeps pushing context path independent/flexable from namespaces.


## Concerns

### Multiple namespaces per context

For circumstances where users may compose multiple nodes of dissimilar namespaces into a single context, the user must subsequently specify a common fully qualified context path for each node to compose, as the varying different namespaces would not push to a common context. For circumstances where the context path is orthogonal to node namespace, the use of fully qualifying all relevant nodes is could be tedious, but could perhaps could still be parametrized via the use of `<var/>`, and `<arg/>` substation and expansion.


### Multiple contexts per process

As before the use of contexts, multiple nodes composed into a single process where each mapped to a septate participant. Each participant subsequently load an security identity and access control credential prevalent to it's respective node. This composition however would inevitably mean that code compiled to node `foo` could access credentials/permissions only trusted only to node `bar`. This consequence of composition could unintendedly subvert the minimal spanning policy as architected by the designer or measured/generated via ROS 2 tooling/IDL.

With the introduction of contexts, it becomes possible to describe the union of access control permission by defining a collection of SROS 2 policy profiles as element within a specific context. This would allow for formal analysis tooling to check for potential violations in information flow control given the composing of nodes at runtime. However, should multiple contexts be used per process, then such security guaranties are again lost. Thus it should be asked whether if multiple contexts per process should even be supported.

## References

1. [Procedurally Provisioned Access Control for Robotic Systems](https://doi.org/10.1109/IROS.2018.8594462)

``` bibtex
@inproceedings{White2018,
	title     = {Procedurally Provisioned Access Control for Robotic Systems},
	author    = {White, Ruffin and Caiazza, Gianluca and Christensen, Henrik and Cortesi, Agostino},
	year      = 2018,
	booktitle = {2018 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS)},
	doi       = {10.1109/IROS.2018.8594462},
	issn      = {2153-0866},
	url       = {https://doi.org/10.1109/IROS.2018.8594462}}
```
