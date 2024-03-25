Repository to host AEGIR patchs for running on Debian11/12 with PHP 8.2
to deploy Drupal 9.4 / 9.5 / 10

All instructions are on https://wiki.evolix.org/HowtoAegir

Installation
Pré-requis : Debian 10 ou Debian 11 avec PHP par défaut, on passera en PHP 8.2 dans un second temps (pour Debian 12 voir installation sous Debian 12)
Préparer un service MySQL permettant ainsi au package Debian aegir3-hostmaster va notamment créer un utilisateur MySQL aegir_root. Si MySQL n’est pas utilisable sans mot de passe via l’utilisateur Unix root, le package aegir3-hostmaster va vous demander un identifiant MySQL “admin”.
# echo "deb http://pub.evolix.org/evolix aegir main" > /etc/apt/sources.list.d/aegir.list
# wget https://pub.evolix.org/evolix/pub.asc -O /etc/apt/trusted.gpg.d/pub_evolix.asc
# chmod 644 /etc/apt/trusted.gpg.d/pub_evolix.asc

# apt update && apt install aegir3


# ls -d /var/aegir/hostmaster-*
/var/aegir/hostmaster-7.x-3.192+nmu1

# /usr/local/bin/drush version
 Drush Version   :  8.4.8 
Note 1 : le dépôt APT Aegir n’existant plus, nous hébergeons les vieux paquets Aegir récupérés via Koumbit (merci!!)
Note 2 : sur certaines installations, il faut relancer une 2e fois l’installation ou lancer un hack en parallèle de l’installation du type screen -S aegir-dpkg-workaround -dm bash -c "while true; do systemctl daemon-reload; done" puis pkill -9 screen && screen -wipe à la fin
On peut ensuite activer différents modules via l’interface web d’Aegir : options pour Git, Let’s Encrypt, etc.
Si l’on utilise pas “Composer” pour déployer, nous conseillons de désactiver le module “Aegir Platform Composer” (sans cela on constate que le code Drupal n’était pas déployé sur les infra multi-serveurs).
Installation sous Debian 12
Il est intéressant d’installer Aegir sous Debian 12 afin d’avoir un système qui soit supporté au moins jusqu’en 2026.
Drush 8.4.8 ne fonctionne pas bien avec PHP 8.2, il faut donc installer Drush 8.4.12 :
# mv /usr/local/bin/drush /usr/local/bin/drush.bak
# su - aegir

$ wget -O setup-composer.php https://getcomposer.org/installer
$ mkdir bin
$ php setup-composer.php --install-dir=./bin --filename=composer
All settings correct for using Composer
Downloading...

Composer (version 2.7.1) successfully installed to: /var/aegir/bin/composer
Use it: php ./bin/composer

$ rm setup-composer.php

$ ./bin/composer global require drush/drush:8.4.12

$ .config/composer/vendor/bin/drush.php version
 Drush Version   :  8.4.12 

# ln -s /var/aegir/.config/composer/vendor/bin/drush.php /usr/local/bin/drush


Ensuite lancer :
# apt install aegir3

puis appliquer les patchs pour PHP 8.

Passer en PHP 8.2 pour Drupal 9.5 / 10
Pour la compatibilité avec Drupal 9.5 / 10, nous passons PHP 8.2 via Sury (PHP 8.1 fonctionne également) :
# mount -o remount,rw /usr
# curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
# dpkg -i /tmp/debsuryorg-archive-keyring.deb
# echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
# apt update

# apt install php8.2 php8.2-cli php8.2-mysql php8.2-gd php8.2-xml php8.2-mbstring

# php -v
PHP 8.2.16 (cli) (built: Feb 16 2024 15:53:11) (NTS)

# a2dismod php php7.3 php7.4
# a2enmod php8.2
# systemctl restart apache2
À ce stade, Aegir fonctionne presque bien en PHP 8, il faut appliquer quelques patches.
Patches pour PHP 8
Commençons par appliquer le patch de l’issue Provision 3374479 pour éviter l’affichage d’erreurs Deprecated :
# wget https://www.drupal.org/files/issues/2023-08-14/3374479-2.patch
# cd /usr/share/drush/commands/provision/Provision/Config/Drupal
# patch -p4 < /root/3374479-2.patch
# patch /var/aegir/hostmaster-7.x-3.192+nmu1/sites/aegir.example.com/settings.php < 3374479-2.patch
Puis un petit patch simple qui permet de réparer la page « Create Platform » (hostmaster/field.form.inc.patch):

# patch /var/aegir/hostmaster-7.x-3.192+nmu1/modules/field/field.form.inc < field.form.inc.patch
Un petit patch pour gérer l’authentification HTTP (hostmaster/http_basic_auth.drush.inc.patch):
# patch /var/aegir/hostmaster-7.x-3.192+nmu1/profiles/hostmaster/modules/aegir/hosting_tasks_extra/http_basic_auth/drush/http_basic_auth.drush.inc < http_basic_auth.drush.inc.patch
Enfin, il faut désactiver la vérification qu’une plateforme n’a pas le même chemin car cela échoue en PHP 8 (hostmaster/hosting_platform.module.patch) :

# patch /var/aegir/hostmaster-7.x-3.192+nmu1/profiles/hostmaster/modules/aegir/hosting/platform/hosting_platform.module < hosting_platform.module.patch
Sinon au moment de la création d’une nouvelle plateforme vous obtiendrez :
'PDOException', '!message' => 'SQLSTATE[HY093]: Invalid parameter number: number of bound variables does not match number of tokens: SELECT n.title AS name FROM {hosting_platform} AS h INNER JOIN {node} AS n ON n.nid = h.nid WHERE publish_path = !publish_path AND h.status >= !h.status; Array ( [!publish_path] => /var/aegir/platforms/PLATFORM [!h.status] => 0 ) ', '%function' => 'hosting_platform_path_exists()', '%file' => '/var/aegir/hostmaster-7.x-3.192+nmu1/profiles/hostmaster/modules/aegir/hosting/platform/hosting_platform.module', '%line' => 669, 'severity_level' => 3, ) 
TODO : un patch plus gros est fourni par BOA, à voir l’intérêt

Patches pour Drupal 10
Pour gérer du Drupal 10, il faut ajouter les fichiers suivants du repretoire provision  dans /usr/share/drush/commands/provision/platform/drupal/ :
    • clear_10.inc
    • cron_key_10.inc
    • deploy_10.inc
    • import_10.inc
    • install_10.inc
    • packages_10.inc
TODO: fournir la source de ces fichiers, voir notamment l’issue Provision 3406925
Il faut supprimer le Drush 12 inclus dans les sources :
$ rm platforms/DRUPAL10/vendor/bin/drush
$ rm -rf platforms/DRUPAL10/vendor/drush
Il faut aussi appliquer ce patch drush-8-symfony-console-compat.patch et en partie ce patch 3353492-symfony-console-4-update_1.patch sur tous les fichiers Symfony (TODO: à relister) de l’issue Provision 3353492 :
# patch /var/aegir/.config/composer/vendor/drush/drush/lib/Drush/Command/DrushInputAdapter.php < drush-8-symfony-console-compat.patch
# cd /usr/share/drush/commands/provision/vendor/symfony/console
# patch -p1 < 3353492-symfony-console-4-update_1.patch
Attention, ce patch nécessite alors de patcher aussi le code de Drupal 9.5 pour qu’il reste « installable »  (drush8/drush-8-symfony-console-compat.patch) :
# patch /var/aegir/platforms/PLATEFORM/vendor/symfony/console/Input/InputInterface.php < drush-8-symfony-console-compat-drupal95.patch
Note : le patch sur Input/InputInterface.php nécessite un coup de main (TODO ;)
On crée le fichier Sql10.php :
$ cat /var/aegir/.config/composer/vendor/drush/drush/lib/Drush/Sql/Sql10.php
<?php
namespace Drush\Sql;

use Drupal\Core\Database\Database;

class Sql10 extends Sql9 {
}
Sinon on aura des erreurs lors de l’installation d’un Drupal 10 :
Drush\Sql\SqlException: Unable to find a matching SQL Class. Drush cannot find your database connection details...
ET idem pour DrupalBoot10.php, StatusInfoDrupal10.php, User10.php et UserSingle10.php :
/var/aegir/.config/composer/vendor/drush/drush/lib/Drush/Boot/DrupalBoot10.php /var/aegir/.config/composer/vendor/drush/drush/lib/Drush/UpdateService/StatusInfoDrupal10.php /var/aegir/.config/composer/vendor/drush/drush/lib/Drush/User/User10.php /var/aegir/.config/composer/vendor/drush/drush/lib/Drush/User/UserSingle10.php
TODO: fichiers à fournir
Il faut aussi patcher dans le code de Drupal 10 :
    1. supprimer “string|” dans les fichiers suivants :
    • vendor/psr/log/src/LoggerTrait.php (18 fois)
    • core/lib/Drupal/Core/Logger/LoggerChannel.php (1 fois)
    • core/lib/Drupal/Core/Logger/RfcLoggerTrait.php (9 fois)
    • core/modules/dblog/src/Logger/DbLog.php (1 fois)
    2. supprimer “: void” dans les fichiers suivants :
    • core/lib/Drupal/Core/Logger/RfcLoggerTrait.php (9 fois)
    • vendor/psr/log/src/LoggerInterface.php (9 fois)
    • core/modules/dblog/src/Logger/DbLog.php (1 fois)
    3. Supprimer “: void” uniquement dans “function log(…)” dans les fichiers suivants :
    • vendor/psr/log/src/LoggerTrait.php
    • core/lib/Drupal/Core/Logger/LoggerChannel.php
Enfin, il faut appliquer le patch suivant sur le fichier /var/aegir/.config/composer/vendor/drush/drush/lib/Drush/Boot/DrupalBoot.php  (drush8/DrupalBoot.php.patch):

# patch /var/aegir/.config/composer/vendor/drush/drush/lib/Drush/Boot/DrupalBoot.php < DrupalBoot.php.patch
