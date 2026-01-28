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

    mkdir -p ~/carcios/{iso, ks, post, branding, repo, docs, tests, carcios}

Il est treès important de créer un dossier `carcios` (nom de votre os) dans le dossier `carcios`,
parce que le kickstart va venir chercher dans ce dossier des informations natives à l'OS (la sécurité, le SSH, etc).

L'arborescence ressemble donc à cela :

.. code-block:: bash

        ~/carcios/
        ├── iso/
        │   ├── OracleLinux-R9-U6-x86_64-dvd.iso
        │   └── OracleLinux-R9-U6-x86_64-dvd-carcios_v1.iso
        ├── ks/
        │   └── carcios-minimal.ks
        ├── carcios/
        │   └── security/
        │       └── ssh/
        │           └── 10-carcios-baseline.conf


2.4 Création du Kickstart minimal
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Dans la VM build, dans un terminal :

.. code-block:: bash

    # ==============================
    # CarciOS - Kickstart minimal
    # Base Oracle Linux 9.6
    # ==============================

    # Mode texte (plus stable pour les tests)
    text

    # Langue / clavier
    lang en_EN.utf-8
    keyboard fr

    # Timezone
    timezone Europe/Paris --utc

    # Network (DHCP auto for now)
    network --bootproto=dhcp --device=link --activate
    network --hostname=carcios-test

    # Installation sources
    cdrom

    # Acceptation license
    eula --agreed

    # Security (simple for now)
    selinux --enforcing
    firewall --enabled --service=ssh

    # Accounts
    user --name=naval --groups=wheel --password=navalpwd --plaintext
    rootpw --lock

    #Bootloader
    bootloader --location=mbr

    # Automatic partitionment (LVM)
    ignoredisk --only-use=nvme0n1
    clearpart --all --initlabel --drives=nvme0n1
    autopart --type=lvm

    # Reboot
    reboot

    # Packages
    %packages
    # --- System base ---
    @core
    @standard
    @gnome-desktop

    # --- Admin tools and debug ---
    vim
    git
    curl
    wget
    tree
    rsync
    jq
    gettext
    xterm

    # --- Network ---
    net-tools
    bind-utils
    wireshark

    # --- Security ---
    policycoreutils-python-utils

    # --- Java ---
    java-1.8.0-openjdk

    %end

    # Post-install minimal
    %post --log=/root/ks-post.log

    # --- Sudo, root and wheel group ---
    dnf -y install sudo

    echo '%wheel ALL=(ALL) ALL' > /etc/sudoers.d/00-wheel
    chmod 440 /etc/sudoers.d/00-wheel

    passwd -l root || true

    echo "[POST] GUI activation"
    systemctl set-default graphical.target
    systemctl enable gdm

    echo "[POST] EPEL repository activation"
    dnf -y install oracle-epel-release-el9

    echo "[POST] DNF cache update"
    dnf makecache

    echo "[POST] No-ISO packages installation"
    dnf -y install htop inotify-tools

    echo "[POST] Post-install ended"
    echo "Welcome to CarciOS - minimal valid build" > /etc/motd

    # --- GNOME branding ---
    echo "[POST] Branding GNOME"

    %end

    # ---------------------------------------------------
    # Copy assets from ISO to installed system (NOCHROOT)
    # ---------------------------------------------------
    %post --nochroot --log=/mnt/sysroot/root/ks-post-nochroot.log

    # DEBUG: Check if the file exist in installer
    ls -l /run/install/repo/carcios/security/ssh || true

    # Create directory in installed system
    install -d -m 755 /mnt/sysroot/etc/ssh/sshd_config.d

    # Copy file in installed system
    install -m 644 \
      /run/install/repo/carcios/security/ssh/10-carcios-baseline.conf \
      /mnt/sysroot/etc/ssh/sshd_config.d/10-carcios-baseline.conf

    %end


    # ------------------------------
    # Validate ssh config (chroot)
    # ------------------------------
    %post --log=/root/ks-post-sshd.log
    sshd -t
    %end


Noter que dans le Kickstart, il faut bien séparer le `%package` du `%post`.\
En effet, `%package` gère les paquets ISO only, le reste (comme par exemples les applications tièrces)
est déployé via `%post`.

Avant d'aller plus loin, on valide le Kickstart pour être sûr qu'il n'y a pas d'erreur :

.. code-block:: bash

   ksvalidator ~/carcios/ks/carcios-minimal.ks

   # Résultat attendu : aucune sortie


2.5 Création d'un fichier conf baseline "basique"
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Le fichier baseline.conf permet de centraliser les règles de sécurité de manière à les déployer via le kickstart (dans
le %post) sans surcharger ce dernier. En effet, il est recommandé d'appeler des fichiers depuis le kickstart et non de
tout écrire à l'intérieur pour des raisons de relecture et de maintenabilité.

Voici un exemple d'un fichier baseline.conf de base :

.. code-block:: bash

    # CarciOS SSH baseline

    # --- Access control ---
    PermitRootLogin no
    MaxAuthTries 3
    LoginGraceTime 30

    # --- Authentication ---
    PasswordAuthentication yes
    PubkeyAuthentication yes

    # --- Hardening basics ---
    X11Forwarding no
    AllowTcpForwarding no
    ClientAliveInterval 300
    ClientAliveCountMax 2

    # --- Logging ---
    LogLevel VERBOSE


2.6 Téléchargement de l'ISO officielle et mise à disposition
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Placer l'ISO officielle dans :

.. code-block:: bash

    ~/carcios/iso/

    # On vérifie avec :
    ls -lh ~/carcios/iso


2.7 Création de l'ISO custom
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Toujours dans la VM build, dans un terminal :

.. code-block:: bash

    cd ~/carcios
    sudo mkksiso \
    --ks ~/ks/carcios-minimal.ks \
    ~/iso/OracleLinux-R9-U6-x86_64-dvd.iso \
    ~/iso/OracleLinux-R9-U6-x86_64-dvd-carcios_v1.iso


2.8 Mise en place des paramètres du Kernel Linux (sysctl)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Les paramètres du noyau Linux vont être modifiés directement depuis un fichier `sysctl`. Les modifications concerneront :

- La sécurité réseau
- La gestion mémoire
- La protection des processus
- La résistance aux attaques
- La performance

On se base sur la même logique que pour les paramètres SSH, à savoir que le fichier va servir de policy et que le système
va l'appliquer automatiquement à l'installation.

















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
