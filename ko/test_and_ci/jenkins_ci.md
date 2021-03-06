# Jenkins CI

<div v-if="$themeConfig.px4_version != 'master'">
  <div class="custom-block tip"><p class="custom-block-title">TIP</p> <p>This page may be out of date. The latest version <a href="../test_and_ci/README.md">can be found here</a>.</p>
  </div>
</div>

Jenkins continuous integration server on [ci.px4.io](http://ci.px4.io/) is used to automatically run integration tests against PX4 SITL.


## Overview

  * Involved components: Jenkins, Docker, PX4 POSIX SITL
  * Tests run inside [Docker Containers](../test_and_ci/docker.md)
  * Jenkins executes 2 jobs: one to check each PR against master, and the other to check every push on master

## Test Execution

Jenkins uses [run_container.bash](https://github.com/PX4/Firmware/blob/master/integrationtests/run_container.bash) to start the container which in turn executes [run_tests.bash](https://github.com/PX4/Firmware/blob/master/integrationtests/run_tests.bash) to compile and run the tests.

If Docker is installed the same method can be used locally:

```sh
cd <directory_where_firmware_is_cloned>
sudo WORKSPACE=$(pwd) ./Firmware/integrationtests/run_container.bash
```

## Server Setup

### Installation

See setup [script/log](https://github.com/PX4/containers/tree/master/scripts/jenkins) for details on how Jenkins got installed and maintained.

### Configuration

  * Jenkins security enabled
  * Installed plugins
    * github
    * github pull request builder
    * embeddable build status plugin
    * s3 plugin
    * notification plugin
    * collapsing console sections
    * postbuildscript

