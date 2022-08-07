# Ansible을 이용한 Apache Web Server 구성관리 Playbook

---

해당 파일을 사용하면  Apache Web Server 설정에 필요한 패키지 설치, 서비스 활성화, 설정 파일 수정 등을 빠른 속도로 자동화 할 수 있다. <br>

  

# 1. Roles

---

```yaml

.

├── roles

│   ├── apache

│   │   ├── tasks

│   │   │   └── main.yaml

│   │   └── vars

│   │       └── main.yaml

│   └── wordpress

│       ├── tasks

│       │   └── main.yaml

│       ├── templates

│       │   └── wp-config.php.j2

│       └── vars

│           └── main.yaml

└── wordpress.yaml

```

  

| Directory       | Description |
| --------------- |--------------|
| tasks | 지정한 명령 실행 |
| vars | 작업 수행 시 사용할 변수 선언 |
| templates | 설정 파일 변경을 용이하게 해주는 Wordpress 설정파일 템플릿 |

<br>

## 1-1. Apache

---

Apache 서버 구성을 위한 패키지 및 서비스 시작 설정을 하기 위한 태스크 디렉토리이다.

<br>

## Tasks

##### Setting PHP 7.4 remi repo block

Remi repository 패키지를 설치한 후 Remi-Safe를 비활성화 시키고 PHP 7.4를 설치한다.

  
```yaml

- name: Setting PHP 7.4 remi repo

 block:

 - name: Install Remi Repo package from URL

 yum:

 name: "{{remi_repo['url']}}"

 state: present

 validate_certs: no

 - name: Remi-Safe OFF

 yum_repository:

 name: "{{remi_safe['name']}}"

 file: "{{remi_safe['name']}}"

 mirrorlist: "{{remi_safe['mirror']}}"

 description: "{{remi_safe['name']}}"

 enabled: no

 - name: Install PHP74

 yum_repository:

 name: "{{php74['name']}}"

 file: "{{php74['name']}}"

 mirrorlist: "{{php74['mirror']}}"

 description: "{{php74['name']}}"

 enabled: yes

 gpgcheck: yes

 gpgkey: "{{php74['gpgkey']}}"

```

  

#####  Download and Start Packages for Apache

Apache 관련 패키지를 다운 후 실행하고 부팅 시 자동 시작될 수 있도록 설정한다.

  
```yaml

- name: Download and Start Packages for Apache

 block:

 - name: Install Packages

 yum:

 name: "{{apache_pkg['pkg']}}"

 state: installed

 - name: Start Services

 service:

 name: "{{ item }}"

 state: started

 enabled: yes

 loop:

 - httpd

 - mariadb

```

<br>

## Vars

```yaml

---

remi_repo:

 url: https://rpms.remirepo.net/enterprise/remi-release-7.rpm

  

remi_safe:

 name: remi-safe

 mirror: http://cdn.remirepo.net/enterprise/7/safe/mirror

php74:

 name: remi-php74

 mirror: http://cdn.remirepo.net/enterprise/7/php74/mirror

 gpgkey: file:///etc/pki/rpm-gpg/RPM-GPG-KEY-remi

  

apache_pkg:

 pkg: httpd,php,php-mysqlnd,mariadb,mariadb-server,python2-PyMySQL

```

  

| Variable      |   Sub   | Description |
| ------------- |--------------|---------|
| remi_repo | url | Remi Repository를 설치하기 위한 url |
| remi_safe | name | Remi Safe 이름 |
|  | mirror | Remi Safe를 위한 mirror list |
| php74 | name | PHP 7.4 이름 |
|  | mirror | PHP 7.4 설치를 위한 mirror list |
|  | gpgkey | PHP 74 설치에 사용되는 암호화 키 파일 |
| apache_pkg | pkg | APache Server를 PHP, MairaDB와 연동시키기 위한 패키지 |

<br>

# 1-2. wordpress

---

## Tasks

##### Download Wordpress From Internet

URL을 사용하여 Wordpress 파일 압축 파일을 다운로드한 후 압축 해제하여 서버에 Wordpress 관련 설정 파일을 설치한다.

  

```yaml

- name: Download Wordpress From Internet

 block:

 - name: Install Compressed Wordpress

 get_url:

 url: "{{wp_url}}"

 dest: "{{remote_dest}}"

 - name: Unarchive Wordpress

 unarchive:

 src: "{{remote_dest}}/{{wp_filename}}"

 remote_src: yes

 dest: /var/www/html

 owner: apache

 group: apache

```

  

-> Install Compressed Wordpress

Wordpress 사이트에서 압축 파일을 다운로드한다.

  
> 이 때, `https://wordpress.org/latest.tar.gz` 는 오류가 발생할 수 있으니
> 최신 파일이 아닌 원하는 버전의 파일을 정확히 다운로드 받아야한다.

  

-> Unarchive Wordpress

해당 파일을 웹에서 사용할 수 있도록 `/var/www/html` 아래에 압축 해제한다.
소유자와 소유그룹을 `apache` 로 설정하여 Apache  Server에서 해당 파일을 사용할 수 있도록 한다.

<br>

#####  Setup Wordpress DB

Wordpress에서 사용할 데이터베이스 `wordpress` 를 생성한다.
해당 데이터베이스를 사용할 수 있는 사용자를 생성하고 데이터 사용을 위한 모든 권한과 패스워드를 부여한다.

  

```yaml

- name: Setup Wordpress DB

 block:

 - name: Make Database

 mysql_db:

 name: wordpress

 state: present

 login_user: root

 - name: Create user and Set Privileges

 mysql_user:

 name: "{{database['user']}}"

 password: "{{database['pwd']}}"

 state: present

 login_user: root

 priv: "{{database['name']}}.*:ALL"

```

<br>

##### Configure Database for Wordpress

미리 생성한 탬플릿을 사용하여 Wordpress php 설정 파일을 덮어쓴다.
해당 파일 또한 Apache Server에서 실행할 수 있어야하므로 소유자와 소유그룹을 Apache로 변경한다.

  

```yaml

- name: Configure Database for Wordpress

  block:

  - name: Copy Database Configure File for Wordpress

    template:

      src: ~/[실행 디렉토리명]/wp/roles/wordpress/templates/wp-config.php.j2

      dest: /var/www/html/wordpress/wp-config.php

      owner: apache

      group: apache
      
```

<br>

#####  Restart Apache

앞서 진행한 설정 관련 수정사항이 모두 적용될 수 있도록 Apache 데몬을 재시작한다.

  

```yaml

- name: Restart Apache

 service:

 name: httpd

 state: restarted

```

<br>

## Templates

Wordpress 설정 파일을 수정하기 쉽도록 기존의 `wp-config.php` 설정 파일이 변수를 참조하여 변경될 수 있도록 jinja 파일로 탬플릿화 한다.

  

``` yaml

...

  

// ** Database settings - You can get this info from your web host ** //

/** The name of the database for WordPress */

define( 'DB_NAME', '{{ database["name"] }}' );

  

/** Database username */

define( 'DB_USER', '{{ database["user"] }}' );

  

/** Database password */

define( 'DB_PASSWORD', '{{ database["pwd"] }}' );

/** Database hostname */

define( 'DB_HOST', '{{ database["host"] }}' );

/** Database charset to use in creating database tables. */

define( 'DB_CHARSET', 'utf8' );

  

/** The database collate type. Don't change this if in doubt. */

define( 'DB_COLLATE', '{{ database["utf"] }}' );

  

...

```

<br>

## Vars

```yaml

---

# Wordpress Package

wp_version: 5.9.3

wp_filename: "wordpress-{{ wp_version }}.tar.gz"

wp_url: "https://wordpress.org/{{wp_filename}}"

  

# Wordpress Destination

remote_dest: "/home/vagrant"

  

# Database Configure

database:

  name: wordpress

  user: wordpress

  pwd: p@ssw0rd

  host: wp-mdb-server-d.mariadb.database.azure.com

  utf: utf8_general_ci

```

  

1) Wordpress 패키지 설치

| Variable      | Description |
| ------------- |--------------|
| wp_version | 설치할 Wordpress 버전 |
| wp_filename | Wordpress 설치를 위한 압축 파일명 |
| wp_url | Wordpress 파일 다운 url |

  

2) Wordpress 실행 계정

| Variable      | Description |
| ------------- |--------------|
| remote_dest | 원격서버에서 Wordpress를 실행할 Host 경로 |

  

3) Database 설정 파일 변경

| Variable      |   Sub   | Description |
| ------------- |--------------|---------|
| database | name| Wordpress 데이터베이스 이름 |
|  | user | 데이터베이스 `wordpress`를 사용할 사용자명 |
|  | pwd | 데이터베이스 `wordpress` 접속을 위한 패스워드 |
|  | host| 데이터베이스 `wordpress`에 접속하는 Endpoint |
|  | utf| 데이터베이스 `wordpress` 데이터 정렬 방법 |

<br><br>

# 2. Playbook :  Wordpress.yaml

---

생성된 VM에 Apache Web Server 설정을 적용할 플레이북이다.
AWS에서 AMI 생성 시 `remote-exec`를 통해  `ansible-playbook` 명령어를 통해 실행되고
Azure에서는 Packer 이미지를 생성 시 `ansible` 프로비저너에 의해 실행된다.

플레이북을 실행하는데 `roles` argument를 통해 role을 참조하고  `vars_files`  argument를 사용해 플레이에 사용할 변수 파일을 참조한다.

  

```yaml

---

- hosts : all

  become : true

  vars_files:

    - /home/vagrant/[실행 디렉토리명]/wp/roles/apache/vars/main.yaml

    - /home/vagrant/[실행 디렉토리명]/wp/roles/wordpress/vars/main.yaml

  

  roles :

    - apache

    - wordpress

```