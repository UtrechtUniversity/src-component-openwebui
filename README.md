# Template repository for SURF ResearchCloud Components

This repository provides you with the boilerplate needed to get started with Ansible-based components for SURF ResearchCloud. It contains:

* An example playbook file (`playbook.yml`)
* An example dependency file (`requirements.yml`)
* Directory structure for separating Ansible concerns:
  * `vars` for declaring variables
  * `roles` for roles you want to use that cannot be imported as a collection in the requirements file
* `component_vars.yml` can be used to simulate variables defined in the ResearchCloud portal, in conjunction with [run_component.sh](#manually-running-the-component-on-a-container)
* Test configuration in the `molecule` directory.
  * A molecule configuration file (`.env.yml`)
* GitHub Actions workflows (see [CI](#ci)).

## Manually running the component on a container

You can use Docker or Podman to test your playbook on a container that mimics a ResearchCloud workspace!

Test containers are available [here](https://github.com/UtrechtUniversity/SRC-test-workspace). The containers container a `run_component.sh` script that mock the process of deploying a component on ResearchCloud.

Should you use Docker or Podman? In order to make it possible to use `sytemd` services on the container, we will start it using `/sbin/init`. This is easier to achieve using Podman, which uses a sane configuration for this by default. Using Docker, it may be necessary to add `--privileged` to your `docker run` command.

1. First determine which test image you want to use. Have a look at the options [here](https://github.com/UtrechtUniversity/SRC-test-workspace#src-test-workspace).
1. In order to be able to pull the test image, login to the GitHub container registry:
    * `podman login ghcr.io`
    * Note: you will need to have a GitHub personal access token configured to login
1. Start a test container with the image you have picked: `podman run -d --name src_component_test -v $(pwd):/etc/rsc/my_component ghcr.io/utrechtuniversity/src-test-workspace:ubuntu_jammy /sbin/init`
    * `-d`: run in the background
    * `--name src_component_test`: give your container a name
    * `-v $(pwd):/etc/rsc/my_component`: make your component directory available on the container
    * start with `/sbin/init` to allow testing of `systemd` services
1. If your component expects variables to be defined in the ResearchCloud portal, mock these variables by defining them in the file `component_vars.yml`.
1. Execute your component on the container using the provided `run_component` script:
  * `podman exec src_component_test run_component.sh /etc/rsc/my_component/playbook.yml`
1. Observe the output, make changes to your component, and run it again!
  * Simply destroy and recreate your container as needed.

## Molecule

[Molecule](https://github.com/ansible/molecule) is a test framework for Ansible. In contrast to the [method above](#manually-running-the-component-on-a-container), Molecule tests allow you to automate performing an entire test cycle, including:

* Creating a test container
* Running certain preparations
* Executing the playbook
* Checking for [idempotence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html#desired-state-and-idempotency)
* Verifying certain assertions
* Destroying the test container

Additionally, you can create multiple test *scenarios*, in order to test the execution of your playbook in different circumstances. While `run_component.sh` can be used for basic development, Molecule is thus more suited for actual integration tests, including in [CI](#ci).

This repository contains some Molecule configuration files in `molecule/ext/molecule-src`. This setup is geared towards testing ResearchCloud components. It configures default container images, and [mimics certain other features of ResearchCloud workspaces](https://github.com/UtrechtUniversity/SRC-molecule#scenarios).

**Note: for `molecule` to run correctly, the path to the component playbook should be correctly set in `molecule/default/molecule.yml! It defaults to the default `playbook.yml`, but if you rename this file, be sure to change in the `molecule.yml`, too.**

To run `molecule`:

1. Install the Python dependencies:
    * `pip install -r molecule/ext/molecule-src/requirements.txt`
1. Install the Ansible dependencies:
    * `ansible-galaxy install -r requirements.yml`
1. Run: `molecule -c molecule/ext/molecule-src/molecule.yml test`

This will run the scenario in the `molecule/default` directory.

To add and run a new scenario, simply:

1. Copy `molecule/default` into `molecule/<yourscenarioname>`.
1. Make your desired changes.
1. Run: `molecule -c molecule/ext/molecule-src/molecule.yml test -s <yourscenarioname>`

## CI

GitHub Actions workflows are added for:

 * Running `molecule` tests
 * Running `ansible-lint`

A configuration file for `ansible-lint` is also provided in `.ansible-lint`.
