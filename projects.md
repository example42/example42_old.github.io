---
layout: page
title: Example42 Projects
subtitle: Open Source Projectst. Mostly Puppet related

---

# Example42 Projects

All Example42 Open Source projects are on [GitHub](https://github.com/example42), so what matters is there.

## Puppet Modules
A large (more than 100 modules) set of Puppet Modules with focus on reusability and standardization.

### Their major features:

- Support for multiple OS, easily extendable to new ones
- A common baseline of reusability oriented parameters
- Support for alternative data sources (Hiera 
- Optional integration with Example42 monitor and firewall modules
- Optional integration with Example42 puppi module
- Attention to interoperability (limited dependencies for not optional features)


### Issues

The current module set contains 2 generation of modules:

- Version 2.x (also called 'NextGen') which are well tested, widely used but ageing (they are developed to be compatible for Puppet version from 2.6).
- Version 3.x which adheres to [StdMod](https://github.com/stdmod) naming standards and are based on more recent patterns.

A full conversion to Version 3.x modules is currently in a development limbo:

- We have not the need to write / convert the current modules 2.x: they work
- We are on hold, considering how the latest Puppet developments can improve modules design patterns ( on Puppet Data in Modules developments 
- We are considering the opportunity to rely on Puppet Labs's most common modules for core functionalities (apache, mysql...)

## Puppi

## Puppet Playground 

## Puppet Architectures

## Puppet Tutorials



