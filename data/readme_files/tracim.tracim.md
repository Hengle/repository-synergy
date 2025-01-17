# Tracim

![logo_tracim](logo_tracim.png)

develop branch status:
[![Build Status](https://travis-ci.org/tracim/tracim.svg?branch=develop)](https://travis-ci.org/tracim/tracim)
[![Coverage Status](https://coveralls.io/repos/github/tracim/tracim/badge.svg?branch=develop)](https://coveralls.io/github/tracim/tracim?branch=develop)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/tracim/tracim/badges/quality-score.png?b=develop)](https://scrutinizer-ci.com/g/tracim/tracim/?branch=develop)

## What is Tracim?

Tracim is a collaborative platform software intended for (not only technical) team collaboration. It is simple to use, offers a user-friendly interface and runs on every computer. It is very valuable for R&D teams, assocations, remote collaboration.

More information on the website: https://www.tracim.fr (in French)

## Quick start (using Docker)

Test Tracim on your computer with Docker:

```
mkdir -p ~/tracim/etc
mkdir -p ~/tracim/var
docker run -e DATABASE_TYPE=sqlite -p 8080:80 -v ~/tracim/etc:/etc/tracim -v ~/tracim/var:/var/tracim algoo/tracim
```

Then visit http://localhost:8080 and log in:

- email: `admin@admin.admin`
- password: `admin@admin.admin`

For advanced docker-based usage, look at the full [Tracim Docker documentation](https://github.com/tracim/tracim/tree/develop/tools_docker),

## License

Tracim is distributed under the terms of 3 distinct licenses:

- AGPLv3 for the Agenda application (`frontend_app_agenda` folder)
- LGPL for other frontend parts (`frontend` and `frontend_xxx` folders)
- MIT License for the rest (backend stuff, functionnal tests, docker recipe, documentation, etc).

## Contribute

In order to contribute to the Tracim source code, please read [CONTRIBUTING.md](./CONTRIBUTING.md) file

## Advanced - Install Tracim from Source

OS compatibility (tested with Python >= 3.5):

- Debian:
  - Jessie (8)
  - Stretch (9)
  - Buster (10)
- Ubuntu:
  - Trusty (14.04) - need manual modification to install Tracim, see https://github.com/tracim/tracim/issues/2514
  - Xenia (16.04)
  - Bionic (18.04)

### Get the source code

Get the source code from GitHub (you need git):

    git clone https://github.com/tracim/tracim.git
    cd tracim/

### Install the backend

#### Option 1: Install the backend manually

See the [Backend README](backend/README.md).

#### Option2: Install backend: Automated script for easy setup

This script runs the backend with a simple default configuration: development.ini conf file. It uses
the default config file, sqlite database, etc.

    ./setup_default_backend.sh

This script uses sudo. Make sure it is installed and configured.
Alternatively, under root:

    ./setup_default_backend.sh root

For each missing configuration file, this script will generate them from the default configuration.
If the default SQLite database is missing, the script will generate it.
This script may also be used for updates. To update a script
generated by the Tracim installation, update the source code with git pull and
rerun the same script to update the database model, the system and Python dependencies.

For more information about configuring the backend, see [Backend README](backend/README.md).
For more information about the configuration files, see development.ini.sample and the [backend setting documentation](backend/doc/setting.md).


### Install the frontend: easy setup

    ./install_frontend_dependencies.sh
    ./build_full_frontend.sh

This script uses sudo. Make sure it is installed and configured.
Alternatively, under root:

    ./install_frontend_dependencies.sh root

You can add "-d" to build_full_frontend.sh to disable obfuscation and reduce build time.

### Run Tracim

Tracim is composed of multiples services, some are web wsgi applications and others
are daemons (servers not web-related to do some task like sending email).

#### Easy start (with pserve and pastedeploy)

An easy way to run Tracim WSGI apps with pastedeploy (config in development.ini):

    cd backend/
    source env/bin/activate

    # running web server
    pserve development.ini

You can run some other WSGI services with pastedeploy using the `tracimcli` command:

    # running webdav server
    tracimcli webdav start

    # running caldav server
    tracimcli caldav start
    tracimcli caldav sync  # sync Tracim data with radicale caldav server

You can run some Tracim daemons too if you want those features:

    # set tracim_conf_file path
    export TRACIM_CONF_PATH="$(pwd)/development.ini"

    ## DAEMONS SERVICES
    # email notifier (if async email notification is enabled)
    python3 daemons/mail_notifier.py &

    # email fetcher (if email reply is enabled)
    python3 daemons/mail_fetcher.py &

You can now head to
[http://127.0.0.1:6543](http://127.0.0.1:6543) and login with the admin account:

 * user: `admin@admin.admin`
 * password: `admin@admin.admin`


#### More documentation about running Tracim and Tracim services


Full documentation about running Tracim services with uWSGI and supervisor
is available in the [Backend README](backend/README.md), sections `Running Tracim Backend Daemon`
and `Running Tracim Backend WSGI APP`.


### Running tests with Cypress

#### Installation of Cypress: Automated script for easy setup

This script check if nodejs is installed (npm is necessary to install Cypress), if file package.json and cypress.json exist in 'functionnal_tests' folder. if not the script install necessary file and install Cypress and his dependency's.

    ./setup_functionnal_tests.sh

This script uses sudo, make sure it is installed and configured.
Alternatively, under root:

    ./setup_functionnal_tests.sh root

If you need to run Cypress with external server of Tracim, modify "baseurl" in cypress.json (for more details, see: https://docs.cypress.io/guides/references/configuration.html#Options ).

#### Prerequisit for running Cypress tests

⚠ To run Cypress tests, you need a running Tracim with a specific configuration:

    cd backend/
    source env/bin/activate
    pserve cypress_test.ini

#### If you are running tests in a development environment

You must change the apiUrl property in `frontend/configEnv.json` to:

    http://localhost:1337/api/v2

Then rebuild the frontend:

    cd frontend/
    npm run build

#### Run tests from the command line ##

To runs all the tests in the 'functionnal_tests/cypress/integration' folder:

    cd functionnal_tests/
    npm run cypress-run

#### Run tests with Cypress GUI ##

You can watch the tests running directly from a (graphical) web interface:

    cd functionnal_tests/
    npm run cypress-open

### Running frontend unit tests

#### Prerequisite for unit tests

To run unit tests, you need to install frontend dependencies and build the full frontend:

    ./install_frontend_dependencies.sh
    ./build_full_frontend.sh

This first script uses sudo, make sure it is installed and configured.
Alternatively, under root:

    ./install_frontend_dependencies.sh root

#### Run tests

To run all the unit tests:

    ./run_frontend_unit_test.sh

To run the unit tests of a specific frontend app or of frontend_lib:

    cd frontend_app_file # (or any frontend app)
    npm run test

Note: to retrieve all frontend apps, run this command:

    ls -d frontend*
