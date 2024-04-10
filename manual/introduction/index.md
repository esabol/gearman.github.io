---
layout: default
title: Introduction
---

Introduction
============

Gearman provides a generic application framework to farm out work to other machines or processes that are better suited to do the work. It allows you to do work in parallel, to load balance processing, and to call functions between languages. It can be used in a variety of applications, from high-availability web sites to the transport of database replication events. In other words, it provides a distributed nervous system to build applications on. It is entirely open source, is fast, flexible, and multi-language. Gearman is designed to be deployed in architectures that have no single point of failure as well.

History
=======

Gearman was originally created at [Danga Interactive](http://www.danga.com/), the company responsible for creating [LiveJournal.com](http://www.livejournal.com/). This company was led by Brad Fitzpatrick, and is responsible for creating other projects such as memcached, MogileFS, and Perlbal. The first blog post on Gearman can be found [here in Brad's journal](http://brad.livejournal.com/2106943.html) in 2005. It is called Gearman because it is an anagram for Manager. Gearman, like a manager, distributes the worker to be done but does not actually do any itself. Gearman is used for a number of purposes at LiveJournal, including being the coordination mechanism for MogileFS, the distributed file storage system.

Implementations
===============

The original version was written in Perl, but when other companies such as Digg and Yahoo! began to use Gearman, they contributed APIs in other languages such as PHP. In early 2008, a port of the job server and libraries in C was started, and by the end of 2008 the new server was ready for the first release. The C server (gearmand) is actively maintained and supported. This also spurred a new set of client APIs to be written based on the C library, including PHP, Perl, Drizzle, and MySQL. A pure Java implementation has also been completed. See the download page for a complete list of APIs available.

One of the powerful features of Gearman is that it doesn't matter which API is used, they all speak the same protocol. There should be no issues running a client, worker, and job server each written in a different language. This can be extremely useful for heterogeneous development environments, and also provides for fast prototyping and upgrade paths. For example, if an initial implementation is written in PHP, parts can later be ported to C as bottlenecks are found.
