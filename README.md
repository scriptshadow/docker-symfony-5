# Mon environnement de développement Symfony 5 avec Docker

# Prérequis
Docker : [https://www.docker.com/get-started](https://www.docker.com/get-started)

Docker-compose : [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

C’est tout

# Création du fichier docker-compose.yml
Je crée un fichier docker-compose.yml dans un répertoire qui contiendra mon environnement.
```bash
$ mkdir docker_symfony
$ cd docker_symfony
$ touch docker-compose.yml
```

Pour savoir dans quel format je dois écrire le docker-compose.yml, je vérifie la version de Docker disponible sur mon poste. Dans mon cas actuel, je suis en version 20.10.6.

```bash
$ docker -v
Docker version 20.10.6, build 370c289
```

Dans [la documentation de Docker](https://docs.docker.com/compose/compose-file/) le format du docker-compose.yml qui correspond à ma version de Docker (c’est rétro-compatible) est la version 3.8.
Je débute donc l’écriture du docker-compose.yml en spécifiant le numéro de version.
```yml
version: "3.8"
services:
```
# Conteneur MySQL
Il est temps désormais de spécifier nos “services” au sens docker-compose du terme. Commençons par le plus simple, la base de donnée MySQL.

```yml
version: "3.8"
services:
    db:
        image: mysql
        container_name: db_docker_symfony
        restart: always
        volumes:
            - db-data:/var/lib/mysql
        environment:
            MYSQL_ROOT_PASSWORD: 'root'
            MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
        networks:
            - dev
    networks:
        dev:
    volumes:
        db-data:
```
Vous aurez noté l’utilisation de l’option **MYSQL_ROOT_PASSWORD: 'root'** et **MYSQL_ALLOW_EMPTY_PASSWORD: ‘yes’**, qui nous permet d’indiquer que nous autorisons les comptes sans mot de passe (Nous ne somme pas en production !). Cela vous simplifiera l’utilisation avec le compte root lorsque vous customiserez vos fichiers .env de Symfony.

Pour que la configuration ci-dessus fonctionne, il faut ajouter à la fin du docker-compose.yml les références au “networks” et mettre en place un volume pour le stockage des fichiers de la base de données.

# Conteneur phpMyAdmin
Qui dit base de donnée MySQL dit phpMyAdmin non ? Au moins en environnement de développement.

```yml
    phpmyadmin:
        image: phpmyadmin
        container_name: phpmyadmin_docker_symfony
        restart: always
        depends_on:
            - db
        ports:
            - 8080:80
        environment:
            PMA_HOST: db
        networks:
            - dev
```
Pas de surprise à cette étape, vous noterez juste l’utilisation de **“depends_on”** qui permet au conteneur phpMyAdmin d’attendre le conteneur MySQL avant de démarrer.

Comme pour le conteneur MySQL, nous utilisons l’option **“restart: always”** qui permet d’indiquer au deamon Docker de redémarrer ce conteneur en cas d’arrêt de celui-ci.

# Conteneur Maildev

Pour de très nombreux projets j’ai besoin d’envoyer des mails, alors je vous propose d’inclure la solution [maildev](https://github.com/maildev/maildev) à votre environnement Docker.
En gros maildev va intercepter vos mails (il joue le rôle d’un serveur SMTP) et vous les présenter dans une petite interface graphique. Indispensable !

```yml
    maildev:
        image: maildev/maildev
        container_name: maildev_docker_symfony
        command: bin/maildev --web 80 --smtp 25 --hide-extensions STARTTLS
        ports:
          - "8081:80"
        restart: always
        networks:
            - dev
```

L’utilisation de la commande **“bin/maildev –web 80 –smtp 25 –hide-extensions STARTTLS”** permet d’éviter des messages d’erreur plus tard avec Symfony, ne l’oubliez pas 😉

# Conteneur Apache et PHP
Attaquons-nous au plus gros morceau, le conteneur Apache/Php. Pour cette partie allons construire nous même notre image à l’aide d’un Dockerfile. Commençons pas créer un fichier Dockerfile dans un sous-répertoire “php”.

```bash
$ mkdir php
$ cd php
$ touch Dockerfile
```
Nous n’allons pas partir de zéro, et allons simplement compléter une image déjà existante de php 7.4 avec Apache. Pour cela nous débutons l’écriture de notre Dockerfile en spécifiant l’image depuis laquelle on débute
```Dockerfile
FROM php:7.4-apache
```

Nous y ajoutons ensuite un certain nombre de librairies et d’outil (composer notamment). C’est dans ce fichier qu’il faudra intervenir si vous devez ajouter des libraires ou des extensions php (j’ai mis quelques exemples dans ce fichier).

J’utilise [cette liste d’extensions docker-php](https://gist.github.com/chronon/95911d21928cff786e306c23e7d1d3f3) pour m’aider (il y a peut-être des listes plus récente, mais celle donne une bonne base).

```Dockerfile
FROM php:7.4-apache

RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

RUN apt-get update \
    && apt-get install -y --no-install-recommends locales apt-utils git libicu-dev g++ libpng-dev libxml2-dev libzip-dev libonig-dev libxslt-dev;

RUN echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    echo "fr_FR.UTF-8 UTF-8" >> /etc/locale.gen && \
    locale-gen

RUN curl -sSk https://getcomposer.org/installer | php -- --disable-tls && \
   mv composer.phar /usr/local/bin/composer

RUN docker-php-ext-configure intl
RUN docker-php-ext-install pdo pdo_mysql gd opcache intl zip calendar dom mbstring zip gd xsl
RUN pecl install apcu && docker-php-ext-enable apcu

WORKDIR /var/www/
```

Une fois cette étape de création du Dockerfile, il ne nous reste qu’à l’ajouter au docker-compose.

Créons donc uns ervice “www” dans notre docker-compose.yml.

```yml
    www:
      build: php
      container_name: www_docker_symfony
      ports:
        - "8741:80"
      volumes:
        - ./php/vhosts:/etc/apache2/sites-enabled
        - ./:/var/www
      restart: always
      networks:
        - dev
```
Les plus observateurs aurons noté que nous n’utilisons plus le mot clé “image”, mais “build” dans ce service. Vous l’aurez compris, cele permet au docker-compose de comprend que pourceservice, il doit utiliser un build d’un Dockerfile.

Pour info, il est possible de lancer le build avec la commande suivante, (si vous ne le faites, cette opération sera réalisée au premier lancement de votre docker-compose).

```docker
$ docker-compose build
```
Vous aurez également noté, que dans les volumes nous mappons un répertoire “/php/vhosts” dans le conteneur. Ce fichier n’existe pas, il est temps de s’en occuper. L’objectif de laisser ce fichier en dehors du conteneur et de pouvoir facilement le modifier sans avoir à re-builder l’image à chaque modification.

```bash
$ mkdir vhosts
$ cd vhosts
$ touch vhosts.conf
```

Pour le contenu de ce fichier, nous nous baserons (quelques toutes petites modifications visibles ci-dessous) sur la version proposer dans [la documentation de Symfony](https://symfony.com/doc/current/setup/web_server_configuration.html).

```prettier
<VirtualHost *:80>
    ServerName localhost
 
    DocumentRoot /var/www/project/public
    DirectoryIndex /index.php
 
    <Directory /var/www/project/public>
        AllowOverride None
        Order Allow,Deny
        Allow from All
 
        FallbackResource /index.php
    </Directory>
 
    # uncomment the following lines if you install assets as symlinks
    # or run into problems when compiling LESS/Sass/CoffeeScript assets
    # <Directory /var/www/project>
    #     Options FollowSymlinks
    # </Directory>
 
    # optionally disable the fallback resource for the asset directories
    # which will allow Apache to return a 404 error when files are
    # not found instead of passing the request to Symfony
    <Directory /var/www/project/public/bundles>
        FallbackResource disabled
    </Directory>
    ErrorLog /var/log/apache2/project_error.log
    CustomLog /var/log/apache2/project_access.log combined
 
    # optionally set the value of the environment variables used in the application
    #SetEnv APP_ENV prod
    #SetEnv APP_SECRET <app-secret-id>
    #SetEnv DATABASE_URL "mysql://db_user:db_pass@host:3306/db_name"
</VirtualHost>
```
#On teste le bon fonctionnement ?

À ce stade vous devriez avoir envie de savoir comment on va se servir de cette configuration.
Commençons par lancer la stack docker-compose.

Le 1er lancement va être bien plus long que les autres car docker-compose va faire le build de votre image **“php”**.

```shell
$ docker-compose up-d
Starting db_docker_symfony         ... done
Starting www_docker_symfony        ... done
Starting maildev_docker_symfony    ... done
Starting phpmyadmin_docker_symfony ... done
```
Lançons maintenant la création d’un nouveau projet Symfony (et changeons le propriétaire du fichier par l’utilisateur courant pour pouvoir les modifier facilement par la suite).

```shell
$ docker exec www_docker_symfony composer create-project symfony/website-skeleton project
$ sudo chown -R $USER ./
```

Une fois l’installation terminée, en vous rendant à l’adresse http://127.0.0.1:8741/, vous devriez voir la page standard d’un nouveau projet Symfony ! **Félicitation**, mais c’est pas encore terminé ne vous emballez pas ;-).


Histoire de poursuivre notre test jusqu’au bout, testons l’utilisation de la base de données depuis le projet Symfony. Pour cela modifions le fichier .env pour y ajouter la référence à la base de données de notre conteneur MySQL.

```dotenv
DATABASE_URL=mysql://root:@db_docker_symfony:3306/db_name?serverVersion=5.7
```
Créons la base de donnée depuis la CLI de Symfony. Pour nous simplifier les prochaines commandes, nous allons entrer dans le shell du conteneur **“www”**.
```shell
$ docker exec -it www_docker_symfony bash
/var/www# cd project
/var/www/project# php bin/console doctrine:database:create
Created database `db_name` for connection named default
```

Mettons en place une **entité “Test”** avec un champ **“test” de type string** (et toutes les valeurs par défaut proposés par le Maker Bundle de Symfony), créons la migration associée puis **exécutons la migration**.

(Rappel : les commandes sont exécutées dans le shell du conteneur **“www”**)

```shell
/var/www/project# php bin/console make:entity
/var/www/project# php bin/console make:migration
/var/www/project# php bin/console doctrine:migrations:migrate
```
Nous pouvons vérifier dans phpMyAdmin que la création est effective à l’adresse http://127.0.0.1:8080/.

Bon, on arrive à la fin de ce bien trop long article, il ne nous reste plus qu’a testé l’envoie de mails depuis Symfony.

Commençons pas indiquer l’adresse de notre serveur SMTP maildev dans le .env.

```dotenv
MAILER_DSN=smtp://maildev_docker_symfony:25
```
Créons un contrôleur “MailController” en charge de l’envoie d’un mail de test avec le Maker Bundle de Symfony. (Toujours dans le shell du conteneur “www”).
```shell
/var/www/project# php bin/console make:controller
```
Modifions le fichier `/src/MailController.php` nouvellement créé pour lui ajouté l’envoie d’un mail de test à chaque chargement de la page.

```prettier
<?php
 
namespace App\Controller;
 
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;
use Symfony\Component\Routing\Annotation\Route;
 
class MailController extends AbstractController
{
    /**
     * @Route("/mail", name="mail")
     **/
    public function index(MailerInterface $mailer): Response
    {
        $email = (new Email())
            ->from('hello@example.com')
            ->to('you@example.com')
            ->subject('Test de MailDev')
            ->text('Ceci est un mail de test');
        $mailer->send($email);
 
        return $this->render('mail/index.html.twig', [
            'controller_name' => 'MailController',
        ]);
    }
}
```

Chargeons la page http://127.0.0.1:8741/mail plusieurs fois d’affilée et vérifions la bonne réception des mails dans maildev à l’adresse http://127.0.0.1:8081/

# Conclusions et dépôt

L’idée, plutôt que de vous fournir une solution prête à l’emploie, était d’expliquer chaque étape de la construction de notre stack de développement avec Docker, cela vous permettra de la faire évoluer selon vos besoins.
