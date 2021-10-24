# Ansible Playbook: eccube

## 概要

* AnsibleでECCube用に Apache + PHP7 + MariaDB の環境を作成する
* 本番環境はAWSのAmazonLinux2で作成する
* ローカル開発環境はVagrantのCentOS7で作成する

## 本番環境: Amazon Linux 2を準備

### Ansibleをインストール

以下のようにして、ExtrasリポジトリからAnsible・PHP7を手動でインストールする

```bash
$ sudo su -
# amazon-linux-extras install ansible2 -y
# amazon-linux-extras install php7.4 -y
```

本番用の構築なら、以下なども作業する

* スワップ領域を設定
* SSHのポート番号を変更
* root宛メールを転送
* CloudWatch Logsを設定

Playbookは `/home/ec2-user/ansible` に配置するものとする

## ローカル開発環境: Vagrantを準備

VagrantでCentOS7を新規にインストールし、そこに開発環境（Apache 2.4.6 + PHP 7.4 + MariaDB 5.5）を構築する<br>
作業フォルダは、ここでは `C:\Users\refirio\Vagrant\eccube4` とする<br>
VagrantのIPアドレスは、ここでは `192.168.33.10` とする

### hostsを設定

Vagrant に `eccube4.local` でアクセスできるようにする

`C:\Windows\System32\drivers\etc\hosts`

```
192.168.33.10   eccube4.local
```

環境を構築すると、以下でアクセスできる想定
http://eccube4.local/

### ボックスの追加

以下のコマンドで公式のCentOS7を追加

```
>vagrant box add centos/7
```

以下のコマンドで確認すると `centos/7 (virtualbox, 1905.1)` が追加されている

```
>vagrant box list
```

### Vagrantfile

```
Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.box_check_update = false
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.synced_folder "./code", "/var/www"
end
```

Vagrantfile と同階層に、同期用フォルダとして `code` を作成しておく<br>
Playbook は `code/ansible-develop` に配置するものとする（つまりサーバ内の `/var/www/ansible-develop` に配置される）

### 起動

```
>cd C:\Users\refirio\Vagrant\eccube4
>vagrant up
```

### 終了する場合

```
>cd C:\Users\refirio\Vagrant\eccube4
>vagrant halt
```

### 破棄する場合

```
>cd C:\Users\refirio\Vagrant\eccube4
>vagrant destroy
```

### 初期起動時にエラーになった場合

Guest Additions を自動でアップデートしてくれるプラグインを導入する

```
>vagrant plugin install vagrant-vbguest
>vagrant halt
>vagrant up
```

解消されなければ、さらにサーバ内でカーネルのアップデートを行う

```
>vagrant ssh
$ sudo yum install -y kernel kernel-devel gcc
$ exit
>vagrant reload
```

### Ansibleをインストール

```bash
$ sudo su -
# yum -y install epel-release
# yum -y install ansible
```

## Ansibleで環境構築

※以降の解説では、URLを `http://eccube4.local/` としている

### Ansibleを実行

Ansibleのhostsファイルの最後に設定を追加

```bash
# vi /etc/ansible/hosts
- - - - - - - - - - - - - - - - - - - - - - - - -
[localhost]
127.0.0.1
- - - - - - - - - - - - - - - - - - - - - - - - -
# exit
```

接続をテスト

```bash
$ ansible localhost -m ping --connection=local
```

Playbookの場所へ移動してから以下を実行

```bash
$ ansible-playbook site.yml --connection=local
```

ブラウザから以下にアクセスして確認<br>
http://eccube4.local/

### データベースと接続ユーザを作成

データベースと接続ユーザを作成

```bash
$ mysql -u root -p
（パスワードなし）
mysql> GRANT ALL PRIVILEGES ON main.* TO webmaster@localhost IDENTIFIED BY '1234';
mysql> FLUSH PRIVILEGES;
mysql> CREATE DATABASE main DEFAULT CHARACTER SET utf8mb4;
mysql> QUIT;
```

### データベースへの接続をテスト

```bash
$ mysql -u webmaster -p
1234
mysql> QUIT;
```

以下でPHPからアクセスできる
```php
<?php

try {
    $pdo = new PDO(
        'mysql:dbname=main;host=localhost',
        'webmaster',
        '1234'
    );

    $stmt = $pdo->query('SELECT NOW() AS now;');
    $data = $stmt->fetch(PDO::FETCH_ASSOC);
    echo "<p>" . $data['now'] . "</p>\n";

    $pdo = null;
} catch (PDOException $e) {
    exit($e->getMessage());
}
```

### メモ

Amazon Linux 2でPlaybookを実行すると以下の警告が表示される

```bash
 [WARNING]: Platform linux on host 127.0.0.1 is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python
interpreter could change this. See https://docs.ansible.com/ansible/2.8/reference_appendices/interpreter_discovery.html for more information.
```

「Pythonが /usr/bin/python にあるが、将来別のPythonがインストールされる予定です」とのこと<br>
ひとまずこのままでも問題無さそうだが、そのうち詳細を調べたい

## 本番環境: バーチャルホストの設定

上の手順で構築すると、ドメインでのアクセスでもIPアドレスのアクセスでも同じ場所を参照する<br>
一つのサーバで複数サイトを管理したい場合や、SEOなどの理由でIPアドレスでアクセスされたくない場合や、「IPアドレスでアクセスされたときは別の画面を表示したい」
のような場合、別途バーチャルホストを設定する

ただしドメイン割り当てまでは、アクセスするにはhostsの設定が必要になるので注意が必要<br>
またAWSでロードバランサー使う場合、ロードバランサーには固定IPアドレスが無いので注意（hostsの定期的な変更が必要になる可能性がある）

## ローカル開発環境: Vagrantfileの調整

### Apacheを自動で起動させる

環境構築が完了した後は、Vagrantfile 内の最後にある `end` の前の行に、以下を追加しておく<br>
これが無いと、Vagrant起動時にApacheが自動で起動されないので注意

* 参考: [Vagrantのup時、httpdが自動起動しないとき - Qiita](https://qiita.com/ooba1192/items/96b7ab25d2bda1676aaa)

```
  config.vm.provision :shell, run: "always", :inline => <<-EOT
    sudo service httpd restart
  EOT
```

### 共有フォルダをNFSにする

VagrantでECCubeを使うと非常に重いが、これは共有フォルダの同期に時間がかかっている可能性がある<br>
`vagrant-winnfsd` プラグインを導入して共有フォルダをNFSに指定すると、動作が早くなる可能性がある（Vagrantを起動している場合は、あらかじめ終了させておく）

```
>vagrant halt
>vagrant plugin install vagrant-winnfsd
```

`Vagrantfile` で以下のように設定する

```
  config.vm.synced_folder "./code", "/var/www"
↓
  config.vm.synced_folder "./code", "/var/www", type: "nfs"
```

```
>vagrant up
```

起動時にWindowsのネットワークの警告が表示された<br>
動作が早くなり、起動自体も早くなった

* [Vagrant(VirtualBox)でディスクアクセスが遅い問題の対処法](https://masshiro.blog/vagrant-laravel-slow/)
* [EC-CUBE 開発コミュニティ - フォーラム](https://xoops.ec-cube.net/modules/newbb/viewtopic.php?topic_id=20574&forum=2)

## ECCubeを配置（公式サイト版を設置する場合）

### プログラムを配置

* [EC-CUBEダウンロード | ECサイト構築・リニューアルは「ECオープンプラットフォームEC-CUBE」](https://www.ec-cube.net/download/)

上記からダウンロードしたファイルを、公開ディレクトリ内に丸ごと配置する

### ECCubeの初期設定

ブラウザから以下にアクセス<br>
http://eccube4.local/

## ECCubeを配置（コマンドで新規作成する場合）

### プログラムを配置

* [EC-CUBE/ec-cube: EC-CUBE is the most popular e-commerce solution in Japan](https://github.com/EC-CUBE/ec-cube)

上記のGitHub版を使う（と言ってもGitHubからダウンロードするわけではなく、`create-project` で作成する）

`main/html` を `main/html_backup` に変更する

```bash
$ sudo su -s /bin/bash - apache
$ cd /var/www/main
```

以下のように `create-project` を実行すると `html` 内にプログラムが作成され、ここが公開ディレクトリになる

```bash
$ composer create-project --no-scripts ec-cube/ec-cube html "4.0.x-dev" --keep-vcs
Creating a "ec-cube/ec-cube" project at "./html"
Installing ec-cube/ec-cube (4.0.x-dev 4feb7da4ad86251c234ebb6508f42079ff5803f2)
  - Installing ec-cube/ec-cube (4.0.x-dev 4feb7da): Cloning 4feb7da4ad
Created project in /var/www/main/html
Loading composer repositories with package information
Installing dependencies (including require-dev) from lock file
Package operations: 164 installs, 0 updates, 0 removals
  - Installing ocramius/package-versions (1.4.2): Downloading (100%)
  - Installing ec-cube/plugin-installer (0.0.8): Downloading (100%)
  ～中略～
  - Installing php-coveralls/php-coveralls (v2.2.0): Downloading (100%)
  - Installing symfony/phpunit-bridge (v3.4.42): Downloading (100%)
Package zendframework/zend-eventmanager is abandoned, you should avoid using it. Use laminas/laminas-eventmanager instead.
Package zendframework/zend-code is abandoned, you should avoid using it. Use laminas/laminas-code instead.
Package sensio/generator-bundle is abandoned, you should avoid using it. Use symfony/maker-bundle instead.
Package setasign/fpdi-tcpdf is abandoned, you should avoid using it. No replacement was suggested.
Package facebook/webdriver is abandoned, you should avoid using it. Use php-webdriver/webdriver instead.
Package phpunit/phpunit-mock-objects is abandoned, you should avoid using it. No replacement was suggested.
Package easycorp/easy-log-handler is abandoned, you should avoid using it. No replacement was suggested.
Generating optimized autoload files
Deprecation Notice: Class Plugin\EntityExtension\Entity\CustomerSortNoTrait located in ./app/Plugin/EntityExtension/Entity/CustomerRankTrait.php does not comply with psr-4 autoloading standard. It will not autoload anymore in Composer v2.0. in phar:///usr/local/bin/composer/src/Composer/Autoload/ClassMapGenerator.php:201
55 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
```

### トラブル: create-project時にエラー

`create-project` の際、以下のエラーになることがあった

```bash
  [ErrorException]
  file_put_contents(/usr/share/httpd/.config/composer/config.json): failed to open stream: No such file or directory
```

以下のようにして `composer` ディレクトリを作成した

```bash
# cd /usr/share/httpd/
# mkdir .config
# chown apache. .config/
# cd .config/
# mkdir composer
# chown apache. composer/
```

これで再度インストールすると進めた

### トラブル: create-project時にトークンを求められる

※GitHubから何度かCloneしていると、不正アクセス対策かトークンを求められることがある？もしくは `/usr/share/httpd/.cache/composer/` に書き込み権限が無い場合に求められる？

```bash
$ php composer.phar create-project --no-scripts ec-cube/ec-cube html "4.0.x-dev" --keep-vcs
Cannot create cache directory /usr/share/httpd/.cache/composer/repo/https---repo.packagist.org/, or directory is not writable. Proceeding without cache
Creating a "ec-cube/ec-cube" project at "./html"
Installing ec-cube/ec-cube (4.0.x-dev 4feb7da4ad86251c234ebb6508f42079ff5803f2)
Cannot create cache directory /usr/share/httpd/.cache/composer/files/, or directory is not writable. Proceeding without cache
  - Installing ec-cube/ec-cube (4.0.x-dev 4feb7da): Cloning failed using an ssh key for authentication, enter your GitHub credentials to access private repos
Head to https://github.com/settings/tokens/new?scopes=repo&description=Composer+on+ip-10-0-1-231.ap-northeast-1.compute.internal+2020-07-08+1512
to retrieve a token. It will be stored in "/usr/share/httpd/.config/composer/auth.json" for future use by Composer.
Token (hidden):
```

上記に表示されているURLにアクセスし、GitHubのパスワードでログイン<br>
「New personal access token」という画面が表示されるので、そのまま「Generate token」をクリックしてみる

```
670ecf020f4e84ccb69d9aa15fde453b727f1f80
```

が表示されたので入力してみると、インストールが続行された

```bash
Token stored successfully.
Cloning 4feb7da4ad
Created project in /var/www/main/html
Cannot create cache directory /usr/share/httpd/.cache/composer/repo/https---repo.packagist.org/, or directory is not writable. Proceeding without cache
Cannot create cache directory /usr/share/httpd/.cache/composer/files/, or directory is not writable. Proceeding without cache
Loading composer repositories with package information
Installing dependencies (including require-dev) from lock file
Your requirements could not be resolved to an installable set of packages.

  Problem 1
    - The requested PHP extension ext-intl * is missing from your system. Install or enable PHP's intl extension.
```

### ECCubeの初期設定

ブラウザから以下にアクセス<br>
http://eccube4.local/

しばらく待ったあと以下が表示された
```
Warning: SessionHandler::read(): Session data file is not created by your uid
```

セッションの保存場所が特殊なので、一般的なものにしておく<br>
`main/html/app/config/eccube/packages/framework.yaml` を編集する

```
12:-        save_path: '%kernel.project_dir%/var/sessions/%kernel.environment%'
12:+        save_path: '/tmp/%kernel.environment%'
```

これでインストール画面が表示された<br>
「推奨」が表示されているが、「apc拡張モジュール」はPHP5の機能みたいなので無視して良さそう

## ローカル開発環境: ECCubeを配置（リポジトリからPULLして作成する場合）

### プログラムを配置

`main/html` を `main/html_backup` に変更する<br>
新規にカラのフォルダ `main/html` を作成し、この中にプログラムを配置する（GitからPULLし、`develop` ブランチに切り替える）<br>
SSHで接続して以下を実行する

```bash
$ sudo su -s /bin/bash - apache
$ cd /var/www/main/html
```

※プラグインを導入済みの場合、「プラグインを導入済みのECCubeをセットアップする場合」を参照

```bash
$ composer install
Loading composer repositories with package information
Installing dependencies (including require-dev) from lock file
Warning: The lock file is not up to date with the latest changes in composer.json. You may be getting outdated dependencies. It is recommended that you run `composer update` or `composer update <package name>`.
Package operations: 164 installs, 0 updates, 0 removals
  - Installing ocramius/package-versions (1.4.2): Downloading (100%)
  - Installing ec-cube/plugin-installer (0.0.8): Downloading (100%)
  - Installing kylekatarnls/update-helper (1.2.0): Downloading (100%)
  - Installing symfony/flex (v1.8.4): Downloading (100%)

Prefetching 160 packages
   Downloading (100%)

  - Installing symfony/process (v3.4.42): Loading from cache
  - Installing symfony/finder (v3.4.42): Loading from cache
  ～中略～
  - Installing php-coveralls/php-coveralls (v2.2.0): Loading from cache
  - Installing symfony/phpunit-bridge (v3.4.42): Loading from cache
Package zendframework/zend-eventmanager is abandoned, you should avoid using it. Use laminas/laminas-eventmanager instead.
Package zendframework/zend-code is abandoned, you should avoid using it. Use laminas/laminas-code instead.
Package sensio/generator-bundle is abandoned, you should avoid using it. Use symfony/maker-bundle instead.
Package setasign/fpdi-tcpdf is abandoned, you should avoid using it. No replacement was suggested.
Package facebook/webdriver is abandoned, you should avoid using it. Use php-webdriver/webdriver instead.
Package phpunit/phpunit-mock-objects is abandoned, you should avoid using it. No replacement was suggested.
Package easycorp/easy-log-handler is abandoned, you should avoid using it. No replacement was suggested.
Generating optimized autoload files
Carbon 1 is deprecated, see how to migrate to Carbon 2.
https://carbon.nesbot.com/docs/#api-carbon-2
    You can run './vendor/bin/upgrade-carbon' to get help in updating carbon and other frameworks and libraries that depend on it.
55 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
ocramius/package-versions:  Generating version class...
ocramius/package-versions: ...done generating version class
Executing script cache:clear --no-warmup [OK]
Executing script cache:warmup --no-optional-warmers [OK]
Executing script assets:install --symlink --relative html [OK]
```

ダミーの商品データは作成されるが、商品画像は存在しない<br>
`.env` が自動作成されているが、このファイルが存在するとインストーラーを起動できない。インストーラからテーブルを作成するために削除しておく（テーブルをインポートする場合、`.env` の内容を編集して使えばいいはず）

### ECCubeの初期設定

ブラウザから以下にアクセス<br>
http://eccube4.local/

## 本番環境: ECCubeを配置（リポジトリからPULLして作成する場合）

### 鍵を作成

```bash
# mkdir /usr/share/httpd/.ssh
# chown -R apache:apache /usr/share/httpd/.ssh
```

```bash
$ sudo su -s /bin/bash - apache
$ ssh-keygen -t rsa
$ cat /usr/share/httpd/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEApyEkei/Or9yISacO2MjrlwEzrCq+5I0MrvD29UWYAPktY3Hu
vXoDwMFm5RY0DciGPt/N7l+ELtD3+pPqlrsUEWHAV5DlP3E0gU8QYLROdh+Xs+bV
ditLKXVy8JCePwjJCcdHzycDtFMFDJIcyYsM+5IQqMBuZ59XzMp/Qz8y5g/Q9AI7
ZPeY6/UaUsJu49F5KHt6eAqLeDKsyd1TdAi+pj46PyZAmN8GEXVK/jpx0Dwf7bJ/
VGmNCA2MqlXzeXghwBmEnNTuDa3MpF+tMsFp0XxKXRYxqobIHtFPPf0kXUwqkpDF
～略～
whVkVc0nF0TZ5gC9mQVt+ntm64kSvmbcfCelvUKy2CVprOM6xIW9cJ5xGwvoQr9s
0QR2G6dk4DFwyj6n5m6U4H86+SYS9UYm0Od7BbiL47hwM0jP8YK4v3/NJOICmoAh
OnkrAoGABRoLeQBOd2GgJGKSxQzL4n5rt+buKRGfNFY798EOLQ/kpJbPHnAqyQTz
7Uhwfzj0zDemVwcw2+qbJArQPvp8n0duXhylfXegKKj9S9/gr89bR8AQw/dmrCMB
HWRdQ2lwf+U7mvs2d07FTSQ3+wWV7hImprzikfT2Mpo+n393Hks=
-----END RSA PRIVATE KEY-----
$ cat /usr/share/httpd/.ssh/id_rsa.pub
ssh-rsa AAAAB ～略～ t0zvJ apache@ip-10-0-1-188.ap-northeast-1.compute.internal
```

### 鍵を登録（Bitbucketの場合）

PULLしたいリポジトリの「Repository settings → アクセスキー → Add Key」に `id_rsa.pub` の内容を追加する

```
Label: Test
Key: id_rsa.pubの内容
```

### 鍵を登録（GitHubの場合）

PULLしたいリポジトリの「Settings → Deploy keys → Add deploy key」に `id_rsa.pub` の内容を追加する。特に理由が無ければ「Allow write access」のチェックは不要

```
Title: Test
Key: id_rsa.pubの内容
```

### アプリケーションを配置

※リポジトリはBitbucketの例。GitHubならClone時に `$ git clone git@github.com:refirio/eccube-test.git /var/www/main/test` のようになる

```bash
$ sudo su -s /bin/bash - apache
$ mv /var/www/main/html /var/www/main/html_backup
$ mkdir /var/www/main/html
$ cd /var/www/main/html
$ git clone git@bitbucket.org:refirio/eccube-test.git /var/www/main/html
$ git pull
$ git checkout develop
```

※プラグインを導入済みの場合、「プラグインを導入済みのECCubeをセットアップする場合」を参照

```bash
$ composer install
Loading composer repositories with package information
Installing dependencies (including require-dev) from lock file
Warning: The lock file is not up to date with the latest changes in composer.json. You may be getting outdated dependencies. It is recommended that you run `composer update` or `composer update <package name>`.
Package operations: 164 installs, 0 updates, 0 removals
  - Installing ocramius/package-versions (1.4.2): Downloading (100%)
  - Installing ec-cube/plugin-installer (0.0.8): Downloading (100%)
  - Installing kylekatarnls/update-helper (1.2.0): Downloading (100%)
  - Installing symfony/flex (v1.8.4): Downloading (100%)

Prefetching 160 packages
   Downloading (100%)

  - Installing symfony/process (v3.4.42): Loading from cache
  - Installing symfony/finder (v3.4.42): Loading from cache
  ～略～
  - Installing php-coveralls/php-coveralls (v2.2.0): Loading from cache
  - Installing symfony/phpunit-bridge (v3.4.42): Loading from cache
Package zendframework/zend-eventmanager is abandoned, you should avoid using it. Use laminas/laminas-eventmanager instead.
Package zendframework/zend-code is abandoned, you should avoid using it. Use laminas/laminas-code instead.
Package sensio/generator-bundle is abandoned, you should avoid using it. Use symfony/maker-bundle instead.
Package setasign/fpdi-tcpdf is abandoned, you should avoid using it. No replacement was suggested.
Package facebook/webdriver is abandoned, you should avoid using it. Use php-webdriver/webdriver instead.
Package phpunit/phpunit-mock-objects is abandoned, you should avoid using it. No replacement was suggested.
Package easycorp/easy-log-handler is abandoned, you should avoid using it. No replacement was suggested.
Generating optimized autoload files
Carbon 1 is deprecated, see how to migrate to Carbon 2.
https://carbon.nesbot.com/docs/#api-carbon-2
    You can run './vendor/bin/upgrade-carbon' to get help in updating carbon and other frameworks and libraries that depend on it.
55 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
ocramius/package-versions:  Generating version class...
ocramius/package-versions: ...done generating version class
Executing script cache:clear --no-warmup [OK]
Executing script cache:warmup --no-optional-warmers [OK]
Executing script assets:install --symlink --relative html [OK]
```

### 設定ファイルの移動

* [インストール方法 - < for EC-CUBE 4.0 Developers />](https://doc4.ec-cube.net/quickstart_install)

自動作成された `.env` は削除したうえで、`httpd.conf` や `.htaccess` で設定することが推奨される
例えばインストール完了後に `/var/www/main/html/.env` で

```
APP_ENV=prod
APP_DEBUG=0
DATABASE_URL=mysql://webmaster:1234@localhost/main
DATABASE_SERVER_VERSION=5.5.64-MariaDB
MAILER_URL=smtp://smtp.gmail.com:465?encryption=ssl&auth_mode=login&username=example@gmail.com&password=xxxxxxxxxxxxxxxx

ECCUBE_AUTH_MAGIC=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ECCUBE_ADMIN_ALLOW_HOSTS=[]
ECCUBE_FORCE_SSL=false
ECCUBE_ADMIN_ROUTE=system
ECCUBE_COOKIE_PATH=/
ECCUBE_TEMPLATE_CODE=default
ECCUBE_LOCALE=ja
```

と設定されている場合、`/var/www/main/.htaccess` を以下のようにする

```
SetEnv APP_ENV prod
SetEnv APP_DEBUG 0
SetEnv DATABASE_URL mysql://webmaster:1234@localhost/main
SetEnv DATABASE_SERVER_VERSION 5.5.64-MariaDB
SetEnv MAILER_URL smtp://smtp.gmail.com:465?encryption=ssl&auth_mode=login&username=example@gmail.com&password=xxxxxxxxxxxxxxxx

SetEnv ECCUBE_AUTH_MAGIC xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SetEnv ECCUBE_ADMIN_ALLOW_HOSTS []
SetEnv ECCUBE_FORCE_SSL false
SetEnv ECCUBE_ADMIN_ROUTE system
SetEnv ECCUBE_COOKIE_PATH /
SetEnv ECCUBE_TEMPLATE_CODE default
SetEnv ECCUBE_LOCALE ja
```

ブラウザでアクセスして、`.env` があるときと同じように表示されるか確認する

## プラグインを導入済みのECCubeをセットアップする場合

### 概要

`composer.json` には、追加プラグインの情報が含まれている（プラグインをインストールすると、`composer.json` と `composer.lock` が更新される）
そのため何らかのプラグインを追加した後は、通常の `composer install` では新規にインストールできなくなる
具体的には、以下のようなエラーになる

```bash
  [RuntimeException]
  You can not install the EC-CUBE plugin via `composer` command.
  Please use the `bin/console eccube:composer:require ec-cube/veritrans4g` instead.
```

`bin/console` で上記指定のコマンドを実行すれば良さそうだが、
初期状態はそもそも `vendor` が無いので以下のエラーになる

```bash
PHP Warning:  require(/var/www/main/html/bin/../vendor/autoload.php): failed to open stream: No such file or directory in /var/www/main/html/bin/console on line 13

Warning: require(/var/www/main/html/bin/../vendor/autoload.php): failed to open stream: No such file or directory in /var/www/main/html/bin/console on line 13
PHP Fatal error:  require(): Failed opening required '/var/www/main/html/bin/../vendor/autoload.php' (include_path='.:/usr/share/pear:/usr/share/php') in /var/www/main/html/bin/console on line 13

Fatal error: require(): Failed opening required '/var/www/main/html/bin/../vendor/autoload.php' (include_path='.:/usr/share/pear:/usr/share/php') in /var/www/main/html/bin/console on line 13
```

### 対応手順

`--no-plugins --no-scripts` を付けてComposerでのインストールを行う<br>
これにより、上記プラグイン関連のエラーを回避できる

まずは作業ディレクトリに移動する

```bash
$ sudo su -s /bin/bash - apache
$ cd /var/www/main/html
```

`--no-plugins --no-scripts` を付けてComposerをインストール

```bash
$ composer install --no-plugins --no-scripts
Deprecation warning: require.ec-cube/VeriTrans4G is invalid, it should not contain uppercase characters. Please use ec-cube/veritrans4g instead. Make sure you fix this as Composer 2.0 will error.
Loading composer repositories with package information
Installing dependencies (including require-dev) from lock file
Package operations: 165 installs, 0 updates, 0 removals
  - Installing ocramius/package-versions (1.4.2): Loading from cache
  - Installing ec-cube/plugin-installer (0.0.8): Loading from cache
  - Installing kylekatarnls/update-helper (1.2.0): Loading from cache
  - Installing symfony/flex (v1.8.4): Loading from cache
  - Installing symfony/process (v3.4.42): Loading from cache
  - Installing symfony/finder (v3.4.42): Loading from cache
  - Installing symfony/polyfill-ctype (v1.17.1): Loading from cache
  - Installing symfony/filesystem (v3.4.42): Loading from cache
  - Installing symfony/polyfill-mbstring (v1.17.1): Loading from cache
  - Installing psr/log (1.1.2): Loading from cache
  ～略～
codeception/codeception suggests installing league/factory-muffin-faker (For Faker support in DataFactory module)
codeception/codeception suggests installing phpseclib/phpseclib (for SFTP option in FTP Module)
codeception/codeception suggests installing stecman/symfony-console-completion (For BASH autocompletion)
Package zendframework/zend-eventmanager is abandoned, you should avoid using it. Use laminas/laminas-eventmanager instead.
Package zendframework/zend-code is abandoned, you should avoid using it. Use laminas/laminas-code instead.
Package sensio/generator-bundle is abandoned, you should avoid using it. Use symfony/maker-bundle instead.
Package setasign/fpdi-tcpdf is abandoned, you should avoid using it. No replacement was suggested.
Package facebook/webdriver is abandoned, you should avoid using it. Use php-webdriver/webdriver instead.
Package phpunit/phpunit-mock-objects is abandoned, you should avoid using it. No replacement was suggested.
Package easycorp/easy-log-handler is abandoned, you should avoid using it. No replacement was suggested.
Generating optimized autoload files
```

完了すると、プラグイン無しの状態でインストールされている

この状態で引き続き `php bin/console eccube:composer:install` を実行しても、データベースに接続できないのでエラーになる（データベースへの接続情報を登録していないため）<br>
いったんECCube自体のセットアップを行う。ブラウザから以下にアクセスしてインストールする（後述の「ECCubeをインストール」と同じ手順でインストールできる）<br>
http://eccube4.local/

インストールできたらマイグレーションを実行する

```bash
$ php bin/console doctrine:migrations:migrate
```

続いて、管理画面から認証キーを設定する（管理画面 → オーナーズストア → 認証キー設定）<br>
以前の環境で設定された認証キーは、`composer.json` の最後の方にある `X-ECCUBE-KEY` という箇所に記録されている。よってこの値を設定する<br>
設定すると改めて認証キーが `composer.json` に書き込まれるが、同じ認証キーを設定しているのでファイルの差分が発生しないことを確認する

以下のコマンドでプラグインをダウンロードする。これで管理画面からもプラグインを確認できる

```bash
$ php bin/console eccube:composer:install
```

最後に、以下でオートローダーを生成する

```bash
$ composer install
```

あとは、必要に応じて管理画面からプラグインを有効化＆設定する

## ECCubeをインストール

※インストール画面は非常に重いので、気長に待つ。実行環境の性能によっては、`php.ini` で `max_execution_time` を `600` などに伸ばしておく必要がある

「ようこそ」が表示されているので「次へ進む」をクリック<br>
「権限チェック」で「アクセス権限は正常です」と表示されていることを確認して「次へ進む」をクリック<br>
「サイトの設定」で以下を入力して「次へ進む」をクリック

```
あなたの店名: テスト店 （一例）
メールアドレス: example@gmail.com （一例。自身のメールアドレス）
管理画面ログインID: admin （一例）
管理画面パスワード: abcd1234 （一例）
管理画面のディレクトリ名: system （一例）
```

「データベースの設定」で以下を入力して「次へ進む」をクリック

```
データベースのホスト名: localhost
データベース名: main
ユーザ名: webmaster
パスワード: 1234
```

「データベースの初期化」が表示されるので「次へ進む」をクリック<br>
「インストールが完了しました！」が表示されたら「管理画面を表示」をクリック
上で設定した管理画面情報でログインする

http://eccube4.local/<br>
http://eccube4.local/system/

### トラブル: インストール画面にアクセスしたらタイムアウトが表示される

以下が表示された
```
Fatal error: Maximum execution time of 30 seconds exceeded
```

プログラムが重いようなので、PHPの実行時間を伸ばしてみる

```bash
# vi /etc/php.ini
- - - - - - - - - - - - - - - - - - - - - - - - -
max_execution_time = 30
↓
max_execution_time = 600
- - - - - - - - - - - - - - - - - - - - - - - - -
```

### トラブル: インストール画面にアクセスしたらSessionエラーが表示される

以下が表示された

```
Warning: SessionHandler::read(): Session data file is not created by your uid
```

セッションの保存場所が特殊みたいなので、一般的なものにしてみる<br>
`app\config\eccube\packages\framework.yaml` を編集する

```
12:-        save_path: '%kernel.project_dir%/var/sessions/%kernel.environment%'
12:+        save_path: '/tmp/%kernel.environment%'
```

## インストール後の設定

### SMTPの設定

メールを送信する場合は `.env` を編集する<br>
以下はGmailのSMTPを使う例

```
25:- MAILER_URL=smtp://localhost:25
25:+ MAILER_URL=smtp://smtp.gmail.com:465?encryption=ssl&auth_mode=login&username=example@gmail.com&password=XXXXXXXXXXXXXXXX
```

### コマンドラインインターフェイスの確認

* [コマンドラインインターフェイス - < for EC-CUBE 4.0 Developers />](https://doc4.ec-cube.net/quickstart_cli)

以下で実行できる

```bash
$ sudo su -s /bin/bash - apache
$ cd /var/www/main/html
$ php bin/console list
```

XAMPP環境なら以下のようにする

```
>C:\xampp\php\php.exe bin/console list
```

### 認証キー

管理画面 → オーナーズストア → 認証キー設定

から、認証キーの新規発行と登録ができる<br>
すでに案件用に発行したキーがあるなら、その値を登録する

### 開発用モードに切り替え

`.env` を以下のように変更する

```
APP_ENV=prod
APP_DEBUG=0
↓
APP_ENV=dev
APP_DEBUG=1
```

変更後、以下のコマンドでキャッシュを削除しておく

```bash
$ php bin/console cache:clear --no-warmup
```

ただしこの設定にすると、キャッシュされない代わりに動作が重くなるので注意
