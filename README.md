#  Open WebUI on SURF ResearchCloud

This repo provides the Ansible playbook and roles for an [Open WebUI](https://openwebui.com/) server component on SURF ResearchCloud (SRC) 🏄‍♀️. Ollama is installed as a backend, allowing for fully self-hosted AI workflows.

You can request access to a catalog item that utilises this component [here](https://portal.live.surfresearchcloud.nl/catalog/catalogItem/d1cb1a87-ec86-41c7-9ceb-4d845821bc9a).

### Prerequisites

* Assumes Nginx is already installed on the workspace via the SURF [Nginx component](https://gitlab.com/rsc-surf-nl/plugins/plugin-nginx).
* This repo uses roles from Utrecht University's general [ResearchCloud collection](https://github.com/UtrechtUniversity/researchcloud-items/), `uusrc.general`.
 
## Overview

### Architecture

* [Ollama](https://ollama.com/) runs as a model backend on `http://localhost:3000`.
* Open WebUI runs on `http://localhost:3000`.
* Nginx reverse proxies inbound https to Open WebUI.
  * If desired, the API is also reverse proxied under `http://<workspace_name>/ext/api`, allowing you to bypass SRAM authentication and use API keys to connect directly to the API.

### Authentication

The server is configured to let the webserver (Nginx) handle authentication. The webserver uses OpenID Connect to authenticate against [SRAM](https://utrechtuniversity.github.io/vre-docs/docs/glossary.html#sram).

Any members of the workspace's Collaborative Organisation (CO) will be able to authenticate using the authentication mechanism of their institution (Single Sign-On). At the moment all members of the collaboration will be considered admins for Open WebUI. Support for finer-grained access management via SRAM is tracked [here](https://github.com/UtrechtUniversity/src-component-openwebui/issues/19).

If desired, the Open WebUI [API](https://docs.openwebui.com/reference/monitoring/) is exposed by Nginx, without SRAM authentication (see [parameters](#ResearchCloud-parameters) below). You'll need to set API keys for your user in Open WebUI, and use them when making requests.

## ResearchCloud parameters

The component takes the following parameters:

* `model`: String. A default Ollama model to pull at workspace creation (you can pull more models yourself after the workspace is running). Leave empty for none.
* `ollama_version`: String. Version of Ollama to install, e.g. `0.17.7`. Leave empty for latest version.
* `ollama_use_external_storage`: Boolean (default: `true`). In case this is true, and in case an SRC Storage Unit is attached to the workspace, will use that storage unit to store Ollama data (including models). This allows pulling larger models than fit on the workspace's internal storage.
* `expose_api`: Boolean (default: `true`). Whether the reverse proxy should serve Open WebUI's API at the route `http://<workspace_name>/ext/api`. This will let through traffic to Open WebUI's API without SRAM authentication, allowing you to use the API from outside of the workspace for automated worklows.

## Maintenance

You can stop/restart Ollama Open WebUI using systemctl, e.g.: `<sudo> systemctl restart openwebui`.

## Testing

This repo contains [Molecule](https://ansible.readthedocs.io/projects/molecule/) tests for the playbook. To run them, you'll need:

1. [Molecule](https://github.com/ansible/molecule/) installed
1. [Podman](https://podman.io/docs/installation) installed
2. Access to the `ghcr.io/utrechtuniversity/src-test-workspace:ubuntu_jammy-nginx` container image
  * see: https://github.com/UtrechtUniversity/SRC-test-workspace/pkgs/container/src-test-workspace

Run the molecule tests with `molecule -c molecule/ext/molecule-src/molecule.yml test --all`.

See [here](https://github.com/UtrechtUniversity/SRC-molecule) for more information on configuring Molecule to test SRC components.

### Manually running the component on a container

The Molecule tests try their best to be as close to actual deployment on SRC as possible. However, this means the setup is a bit complicated and doesn't perform super fast. If you want an easy way to test/develop, locally, you can also use the following method:

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

### Linting

`ansible-lint` is configured for this repository. Just install `ansible-lint` and run: `ansible-lint . playbook.yml`

## CI

GitHub Actions workflows are added for:

 * Running `molecule` tests
 * Running `ansible-lint`

A configuration file for `ansible-lint` is also provided in `.ansible-lint`.

# License

[MIT](LICENSE). Copyright 2025-2026, Utrecht University and the orignal contributors:

- Dawa Ometto (UU)
- Jelle Treep (UU)
- Stefan Verhoeven (eScience NL)
- Slawa Loev (SURF)
- Stef de Groot (HU)
- Berend Wijers (UvA)
- Farshad Radman (EUR)
