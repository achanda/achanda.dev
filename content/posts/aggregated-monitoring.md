+++
title = 'Aggregated Monitoring for Systemic Problems'
date = 2024-05-30T21:08:17-05:00
draft = false
+++

# Introduction

We build software systems bottoms up: install linux, install postgres, put a proxy in front. However, our users see those
systems in the reverse direction, a user connection will first arrive at the proxy, then postgres and then on the host. There
is a lot of value in monitoring these systems from the user's point of view. However, such monitoring tends to hide the underlying
tiered nature of the system. As the system evolves and more components are added, practitioners often tend to forget how those
components interact. And then when there is an incident, lets say an user reports that their queries are running slow. The logical
approach is to follow the users path, from the proxy to postgres and then to Linux. What if the problem was a faulty disk driver
that caused disk IO to slow down? we have no notion of interdependence of these individual components.

Modern software systems are fractal in nature, a larger system if made up of a set of smaller systems and so on. For our discussion
a system is any software that is on interest. Each system at a given level might have special roles and in some cases, a system might 
be shared between multiple higher level systems. At this point the graphical nature of this is obvious. The graph might even have cycles
where a system is simultaneously at the same level as another system and also it's dependent.

To understand this better, let us look at Postgres. The whole server is our starting point in this case. The first level subsystems can be 
the processes that postmaster spawns: a connection handler, bgwriter, WAL writer etc. For bgwriter, the subsystems are the underlying kernel
and the connection handler.

Modern monitoring systems focus on the root's behaviour as seen by a user. This model makes it easy to define failures: if it does not work
for the end user, it must be broken. But this approach hides problems in underlying systems where the fix is typically applied. Hence such
systems often need experienced operators who are familiar with the failure modes. Let us look at a concrete example. Going back to Postgres'
again, assume our users are complaining that a given query is running slower than usual without any other changes. An experienced DBA who is
familiar with the schema and traffic volume might realize that this is not due to missing index, and look at hardware performance. But someone
not familiar with those details might look at indexing. 
