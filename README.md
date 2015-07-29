# Graphite-Web

[![Build Status](https://travis-ci.org/graphite-project/graphite-web.png?branch=master)](https://travis-ci.org/graphite-project/graphite-web)

## Overview

Graphite consists of three major components:

1. Graphite-Web, a Django-based web application that renders graphs and dashboards
2. The [Carbon](https://github.com/graphite-server/carbon) metric processing daemons
3. The [Whisper](https://github.com/graphite-server/whisper) time-series database library

![Graphite Components](https://github.com/graphite-server/graphite-web/raw/master/webapp/content/img/overview.png "Graphite Components")

## About graphite-server organization

The graphite-server is a fork of all major components of [graphite-project](https://github.com/graphite-server).
It is inteded to be a custom made version centered in centos and other specific technologies. 
The plan is to constantly update these projects base on [graphite-project](https://github.com/graphite-server).
So it is highly recommended to contribute to that project.  

## Installation, Configuration and Usage

Please refer to the instructions at [readthedocs](http://graphite.readthedocs.org/) for general installation.

For particular instructions in centos refer [centos-install](https://github.com/graphite-server/graphite-web/blob/master/centos-install.md).

## License

Graphite-Web is licensed under version 2.0 of the Apache License. See the [LICENSE](https://github.com/graphite-project/graphite-web/blob/master/LICENSE) file for details.
