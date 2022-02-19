---
layout: post
title: MariaDB installation and configuration, 2018-05-10
---

## MariaDB 설치 및 셋업

`mysql_secure_installation` 이라는 스크립트(?)가 있어서 테스트 목적으로 허용된 액세스 등을 production 들어가기 전에 제거할 수 있다.

역시 접근 권한 관련 문제. Ubuntu에서 설치 후 root가 아닌 계정에서 `mysql -u root -p` 로 로그인 하려면 ERROR 1698 (28000): Access denied for user 'root'@'localhost' 라는 에러가 발생한다. 이것은 MariaDB가 기본적으로 Unix Socket을 이용해서 호스트의 user credential을 이용하기 때문에 발생하는 에러라고 한다 [[1](https://stackoverflow.com/questions/39281594/error-1698-28000-access-denied-for-user-rootlocalhost)][[2](https://mariadb.com/kb/en/library/authentication-plugin-unix-socket/)].  

[[1](https://stackoverflow.com/questions/39281594/error-1698-28000-access-denied-for-user-rootlocalhost)] 에 따르면 해결방법은:
1. MariaDB의 root user가 `mysql_native_password` authentication plugin을 쓰게 하든가
2. 일단 root로 로그인한 후에 MariaDB에 새로운 계정(시스템에서 쓰는 것과 같은 이름)을 추가하여 사용하든가
3. 새 계정을 만든 다음에 `mysql_native_password` authentication plugin을 쓰게 할 수도 있겠다.

authentication plugin을 바꾸는 것은 다음과 같이 한다 (출처 [[2](https://mariadb.com/kb/en/library/authentication-plugin-unix-socket/)]):

```
mysql --user=root --host=localhost

ALTER USER root@localhost identified with 'mysql_native_password';
SET PASSWORD = password('foo');
quit
```

2. 는 다음과 같다. (출처 [[1](https://stackoverflow.com/questions/39281594/error-1698-28000-access-denied-for-user-rootlocalhost)])

```
$ sudo mysql -u root # I had to use "sudo" since is new installation

mysql> USE mysql;
mysql> CREATE USER 'YOUR_SYSTEM_USER'@'localhost' IDENTIFIED BY '';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'YOUR_SYSTEM_USER'@'localhost';
mysql> UPDATE user SET plugin='auth_socket' WHERE User='YOUR_SYSTEM_USER';
mysql> FLUSH PRIVILEGES;
mysql> exit;

$ service mysql restart
```

`service mysql restart`를 해야 하는지는 확신이 없다. 계정 추가하려고 DB 서비스를 재시작 하는 것은 이상하다. 
