Virtualisation d'un serveur - VMWare Workstation
================================================

Objectif : pouvoir déployer en local un serveur sur une machine virtuelle.

Pour cet exemple, **Ubuntu Server** sera utilisé pour sa facilité, sa solidité, sa modularité et ses performances.

Dans cette documentation, nous allons mettre en place un serveur virtuel hébergeant un **Docker** et un **Git** local.

1. Création de la VM serveur
----------------------------

Voici les paramètres recommandés :

.. list-table::
   :widths: 30 70
   :header-rows: 0

   * - OS
     - Linux
   * - Version
     - Ubuntu Server 24.04 LTS
   * - CPU
     - 2 vCPU
   * - RAM
     - 2 Go
   * - Espace disque
     - 20 Go
   * - Réseau
     - Bridged


Nous utilisons l'interface *bridged* pour que la VM soit vue comme un vrai appareil sur le réseau avec sa propre IP,
à l'instar d'un serveur physique.


2. Configuration du serveur
---------------------------

2.1 Configuration générale
^^^^^^^^^^^^^^^^^^^^^^^^^^

Avant toute chose, mettre le système à jour est une bonne pratique à garder en tête.

Sur le serveur :

.. code-block:: bash

    sudo apt update
    sudo apt upgrade
    # confirmer avec y

On prend le temps de vérifier quelques informations importantes :

.. code-block:: bash

    hostnamectl # Pour vérifier le nom de la machine.
    ip a # Permet d'afficher l'IP.
    whoami # Affiche le username actuel.
    groups # Affiche les groupes (utile pour voir si sudo est bien configuré).

On vérifie ensuite l'état du pare-feu :

.. code-block:: bash

    sudo ufw status
    # Sur une machine "from scratch", dois afficher 'inactive'.

On autorise ensuite le SSH puis on active le pare-feu (notion de sécurité importante) :

.. code-block:: bash

    sudo ufw allow OpenSSH # Autorise l'accès SSH au travers du pare-feu.
    sudo ufw enable # Activation du pare-feu.
    sudo ufw status # Permet de vérifier l'état du pare-feu.
    # Pour info, UFW (Uncomplicated FireWall) est une surcouche simplifiée de nftables/iptables.

A ce stade, le serveur est opérationnel et prêt à recevoir la couche supplémentaire utilisée (Docker, OpenLDAP, etc).

2.2 Configuration Docker
^^^^^^^^^^^^^^^^^^^^^^^^

D'abord, on vérifie sur quelle version d'Ubuntu on se trouve :

.. code-block:: bash

    lsb_release -a # Affiche le nom de code (par exemple, 'noble' pour 24.04)
    uname -m # Affiche l'architecture (exemple : 'x86_64')

Ensuite, on met à jour la liste des paquets disponibles, puis on installe trois outils nécessaires pour la suite :

.. code-block:: bash

    sudo apt update # Récupération de la liste à jour des packages.
    sudo apt install -y ca-certificates curl gnupg
    # 'ca-certificates' permet de rendre HTTPS fiable, 'curl' est utilisé
    # pour télécharger la clé du repo Docker, et 'gnupg' pour gérer la clé cryptographique.

Sur **Ubuntu**, il est recommandé de stocker les clés de dépôts dans `/etc/apt/keyrings`, de manière à rendre la
configuration plus propre et sécurisée.

.. code-block:: bash

    sudo install -m 0755 -d /etc/apt/keyrings
    # Install permet de copier et/ou créer des fichiers/dossiers avec des permissions propres.
    # -d pour 'create directory' et l'argument '-m 0755' fixe les permissions :
    # 0755 : Propriétaire en lecture + ecriture + exécution (7), groupe en lecture + exécution (5),
    # pareil pour others.

On prend ensuite la clé publique de Docker pour la mettre dans le fichier `docker.asc` et finir en autorisant sa lecture :

.. code-block:: bash

    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo tee /etc/apt/keyrings/docker.asc > /dev/null
    # -f pour faire échouer curl au besoin.
    # -s pour une installation silencieuse, -S pour réafficher tout de même les erreurs au cas où.
    # -L pour suivre les redirections.
    # Le pipe '|' s'assure ensuite que ce que 'curl' télécharge soit donné à 'tee' pour écriture.

    sudo chmod a+r /etc/apt/keyrings/docker.asc
    # a+r = "all" + "read".

Ensuite, nous créons un fichier de dépôt APT ``/etc/apt/sources.list.d/docker/list`` et y ajoutons le dépôt Docker :

.. code-block:: bash

    echo \
        "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
        https://download.docker.com/linux/ubuntu \
        $(lsb_release -cs) stable" | \
        sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        # deb = dépôt de binaries packages.
        # stable = on privilégie toujours le canal stable.
        # $(dpkg --print-architecture) = Substitution de commandes,
        # l'exécute et remplace par le résultat.

Comme nous venons d'ajouter un nouveau dépôt, on remet à jour les dépôts APT puis on installe Docker + compose :

.. code-block:: bash

    sudo apt update
    sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    # docker-ce = Le moteur Docker, sans quoi le reste ne fonctionne pas.
    # docker-ce-cli = Pour permettre la commande `docker`.
    # containerd.io = Le runtime des conteneurs sur lequel Docker s'appuie.
    # docker-buildx-plugin = Le build moderne de Docker.
    # docker-compose-plugin = `docker compose` v2.

Docker est un ensemble de fonctionnalités qui s'exécutent ensemble ou de manière autonome les unes des autres :
Un daemon `dockerd`, une commande client `docker`, un runtime container `containerd` et des plugins.

3. Démarrage et configuration de Docker
---------------------------------------

On fait en sorte que Docker soit activé immédiatement et à chaque démarrage.

.. code-block:: bash

    sudo systemctl enable --now docker
    # --now permet de démarrer tout de suite et `enable` le démarrage automatique au boot.

On autorise les utilisateurs (généralement un) à utiliser Docker sans sudo. Pour cela, on l'ajoute au groupe `docker` :

.. code-block:: bash

    sudo usermod -aG docker "$USER"
    # = -a pour append, permet d'ajouter sans remplacer la liste des groupes existants.
    # En étant membre du groupe docker, l'utilisateur pourra exécuter les commandes suivantes :
    docker ps
    docker compose up -d

Une fois le serveur redémarré, on vérifie que Docker tourne et qu'on peut l'utiliser sans sudo :

.. code-block:: bash

    docker --version
    docker compose version
    docker ps

Si ``docker ps`` marche sans sudo, tout s'est passé comme prévu.




