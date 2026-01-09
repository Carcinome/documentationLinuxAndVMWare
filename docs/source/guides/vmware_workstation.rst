VMWare Workstation
==================

Contenu du guide VMWare Workstation.


1. Création des VMs
-------------------

Création de deux VMs, une de construction et une de test.

**1. Machine de construction :**

- Oracle Linux 9.6
- GNOME
- Accès Internet

Elle a pour rôle d'écrire les Kickstart, de construire l'ISO custom et de documenter.

**2. Machine jetable :**

Elle sert à tester l'ISO custom.
Permet la validation de :

- L'installation est 100% automatique
- Les règles de sécurité sont appliquées
- Le branding est ok


2. Création de la VM Build
--------------------------

2.1 Création de la VM dans VMWare Workstation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Voici les paramètres recommandés :

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


2.2 Mise à jour et installation des outils pour le build
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Une fois connecté sur la VM, dans un terminal :

.. code-block:: bash

    sudo dnf -y update
    sudo reboot

    # Puis on installe les outils pour construire l'ISO :
    sudo dnf -y install \
    lorax \ # Fabrique et modifie les ISO (il contient mkksiso)
    pykickstart \ # Permet de valider les fichiers Kickstart
    anaconda \ # Le moteur d'installation Oracle Linux
    createrepo_c \ # Permettra plus tard de configurer les dépôts RPM directement dans l'OS crée
    genisoimage \ # Utilitaire de manipulation d'ISO
    xorriso \ # Utilitaire de manipulation d'ISO
    isomd5sum \
    git vim

2.3 Arborescence entreprise
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Dans le HOME, créer les dossiers suivants :

.. code-block:: bash

    mkdir -p ~/carcios/{iso, ks, post, branding, repo, docs, tests}


2.4 Création du Kickstart minimal
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Dans la VM build, dans un terminal :

.. code-block:: bash

    cd ~/carcios/ks
    touch carcios-minimal.ks

    vim carcios-minimal.ks

    # Dans le fichier .ks, écrire ceci :

    # ==================================
    # CarciOS - Kickstart minimal
    # Base Oracle Linux 9.6
    # ==================================

    # Text mod (more stable for tests)
    text

    # Langage / keyboard
    lang en_EN.UTF-8
    keyboard fr

    # Timezone
    timezone Europe/Paris --utc

    # Network (DHCP auto)
    network --bootproto=dhcp --device=link --activate
    network --hostname=carcios-test

    # Installation sources
    cdrom

    # License agreed
    eula --agreed

    # Security (simple for now)
    selinux --enforcing
    firewall --enabled --service=ssh

    # Accounts
    user --name=adminuser --groups=wheel --password=changeme --plaintext

    # Bootloader
    bootloader --location=mbr

    # Automatic partitions
    ignoredisk --only-use=nvme0n1 # Ligne importante,
    # pour éviter que le disque ne soit pas trouvé par anaconda.
    # pour savoir le modèle précis de disque, sur la machine on peut faire :
    # lsblk -d -o NAME,SIZE,MODEL,TYPE
    clearpart --all --initlabel --drives=nvme0n1 # Cette ligne permet d'autoriser anaconda
    # à formater le disque avant d'écrire dessus.
    autopart --type=lvm

    # Auto reboot
    reboot

    # Packages
    %packages
    @core
    @standard
    @gnome-desktop
    vim
    git
    curl
    wget
    %end

    # Post-install minimal
    %post --log=/root/ks-post.log
    # Mode graphique activé par defaut
    systemctl set-default graphical.target
    # S'assurer que GDM est bien actif
    systemctl enable gdm
    # Validation message
    echo "Welcome on CarciOS - minimal build validated" > /etc/motd

    %end

Avant d'aller plus loin, on valide le Kickstart pour être sûr qu'il n'y a pas d'erreur :

.. code-block:: bash

   ksvalidator ~/carcios/ks/carcios-minimal.ks

   # Résultat attendu : aucune sortie


2.5 Téléchargement de l'ISO officielle et mise à disposition
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Placer l'ISO officielle dans :

.. code-block:: bash

    ~/carcios/iso/

    # On vérifie avec :
    ls -lh ~/carcios/iso


2.6 Création de l'ISO custom
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Toujours dans la VM build, dans un terminal :

.. code-block:: bash

    cd ~/carcios
    sudo mkksiso \
    --ks ~/ks/carcios-minimal.ks \
    ~/iso/OracleLinux-R9-U6-x86_64-dvd.iso \
    ~/iso/OracleLinux-R9-U6-x86_64-dvd-carcios_v1.iso


3. Création de la VM test et premiers essais
--------------------------------------------

3.1 Création de la VM test dans VMWare Workstation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Voici les paramètres recommandés :

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
     - 4 Go
   * - Espace disque
     - 40 Go
   * - CD/DVD
     - ISO Custom carcios.iso
   * - Réseau
     - NAT

On procède ensuite à l'installation sur la VM de test de l'OS custom.