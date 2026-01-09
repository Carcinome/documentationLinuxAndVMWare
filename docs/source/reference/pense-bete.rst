Pense-bête
==========

Ici un condensé en vrac de tips et astuces.

1. Création de documentation RST
--------------------------------

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

2. Tableau sans entête RST
--------------------------

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


3. Monter un dossier partagé sur VMWare
---------------------------------------

Pour monter un dossier partagé sur VMWare (une fois le paramétrage de la VM fait), utiliser cette commande :

.. code-block:: bash

    sudo mount -t fuse.vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other

4. Connaître le type de disque sur une machine Linux
----------------------------------------------------

Pour connaître le type de disque, se connecter en tty sur un terminal et entrer la commande suivante :

.. code-block:: bash

    lsblk -d -o NAME,SIZE,MODEL,TYPE
    # si lsblk n'est pas dispo :
    cat /proc/partitions

5. Contrôler le bon déroulé de l'installation de l'OS
-----------------------------------------------------

Pour contrôler le bon déroulé de l'installation de l'OS personnalisé, on utilise cette commande :

.. code-block:: bash

    cat /etc/motd

