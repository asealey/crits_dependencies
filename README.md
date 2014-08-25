CRITs Dependencies
==================

This repository contains some of the dependencies for CRITs. This includes any
software and libraries that cannot be acquired via normal OS package management.

# Acquiring this software

The two options to download the dependencies are to either clone the repository
or download the repo as a zipfile. If you choose to git clone it will be much
easier to update the dependencies in the future via git pull.

If you are looking for the dependencies for a specific version of CRITs, be sure
to download the proper tagged commit of the repo.

# Installing this software

Once you have downloaded or cloned the repository, you will want to run the
install_dependencies.sh script which comes with it.

`sudo -E ./install_dependencies.sh`

This will take care of using the OS's package management system to download some
basic packages before installing the local content.
