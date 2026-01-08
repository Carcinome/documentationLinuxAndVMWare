Pense-bête
==========


Pour créer de la documentation en RST (recommandé), déployer via sphinx le package complet dans un environnement Python.

Penser à se mettre en environnement virtuel Python :

.. code-block:: powershell

    .\env_work\Scripts\activate.ps1

Une fois ceci fait, on installe et configure le package Sphinx :

.. code-block:: Python

    python -m pip install --upgrade pip
    python -m pip install sphinx

    # On se déplace dans le dossiers docs nouvellement crée :
    cd docs
    sphinx-quickstart

    # Thèmes :
    pip install sphinx-rtd-theme
    pip install sphinx-autodoc-typehints

    # On vérifie avec :
    sphinx-build --version


Voici un tableau type sans entête en RST :

.. list-table::
   :widths: 30 70
   :header-rows: 0

   * - OS
     - Linux
   * - Version
     - Oracle Linux 9.6
   * - CPU
     - 2 vCPU
   * - RAM
     - 8 Go
   * - Espace disque
     - 50 Go
   * - Réseau
     - NAT


Pour monter un dossier partagé sur VMWare (une fois le paramétrage de la VM fait), utiliser cette commande :

.. code-block:: bash

    sudo mount -t fuse.vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other

Rajouter les commandes pour lire les disques en tty anaconda, etc