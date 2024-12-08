---
layout: post
title: "AltBase 1: Как hasher собирает rpm пакеты"
date: 2024-12-07 19:55:44 -0300
categories: AltBase
---

Это первая статья в серии про сборку rpm-пакетов в ALTLinux, в ней я расскажу,
как hasher собирает rpm-пакеты.

Для сборки rpm-пакетов hasher создает изолированное chroot окружение,
устанавливает туда все необходимые зависимости и вызывает в нем rpmbuild. Сам
hasher состоит из нескольких вспомогательных утилит и команды hsh, которая их
объединяет. Hasher умеет производить сборку из двух типов пакетов: pkg.tar и
src.rpm. Первых тип pkg.tar - это архив с исходниками, который получается в
результате работы gear. Второй тип - это сборка бинарного rpm-пакета из src.rpm
пакета.

В качестве примера рассмотрим сборку пакета audit.
Для сборки пакета из gear-репозитория с verbose выводом достаточно следующих
команд:
```shell
git clone git://git.altlinux.org/gears/a/audit.git
cd audit
gear-hsh -v -- -v
```
Или можно скачать src.rpm и собрать его. Возьмем актуальный на данный момент src.rpm:
```shell
wget https://git.altlinux.org/tasks/354743/build/100/x86_64/srpm/audit-4.0.2-alt1.src.rpm
hsh -v ./audit-4.0.2-alt1.src.rpm
```
В результате мы получаем готовые бинарные rpm-пакеты в
~/hasher/repo/x86_64/RPMS.hasher/

Но чтобы понять, как это все работает на самом деле заменим gear-hsh на все
вспомогательные утилиты, которые запускаются в процессе сборки:
```shell
git clone git://git.altlinux.org/gears/a/audit.git
cd audit

hsh-mkchroot -v
mkaptbox -v
hsh-initroot -v
gear -v ./pkg.tar
# Альтернативно можно запустить hsh-rebuild с *.src.rpm файлом вместо pkg.tar
hsh-rebuild -v ./pkg.tar
hsh-rmchroot -v
```

Остановимся на каждой команде подробнее:

```shell
hsh-mkchroot -v
```
Эта команда инициализирует директорию как рабочую директорию hasher. По
умолчанию рабочей директорией считается ~/hasher, но ее можно заменить, указав
нужную директорию последним аргументом.

```shell
mkaptbox -v
```
Эта команда создает aptbox в рабочей директории hasher. Ей так же можно указать
рабочую директорию hasher последним аргументом. По сути, aptbox - это инстанс apt
настроенный на работу с будущим chroot'ом. Именно этой команде можно передать
опцию --apt-config, чтобы указать кастомный файл конфигурации apt, например для
сборки в окружении, отличающимся от хостового.

```shell
hsh-initroot -v
```
Эта команда создает базовое chroot окружение, наполняя этот chroot минимальным
набором пакетов для работы системы. Базовое окружение состоит из двух наборов
пакетов: pkg_init_list и pkg_build_list. Их содержимое можно посмотреть в файле
[hasher/hsh-sh-functions.in]():
```shell
# build stage package file list
pkg_build_list='basesystem rpm-build>=0:4.0.4-alt21 kernel-headers-common>=0:1.1.4-alt1 sisyphus_check>=0:0.7.3 time'
build_list=
Helpify pkg_build_list

# initial stage package file list
pkg_init_list='setup filesystem rpm fakeroot>=0:0.7.3'
init_list=
Helpify pkg_init_list
```
Их можно переопределить опциями `--pkg-init-list=LIST` и
`--pkg-build-list=LIST`, или дополнить, указав '+' перед списком пакетов `LIST`,
например `--pkg-init-list=+less` добавит less в базовое окружение.


```shell
gear -v ./pkg.tar
```
Этой командой мы создаем pkg.tar архив из текущего gear репозитория. При
использовании команды `gear-hsh` или `gear --hasher -- ...` этот архив создается
в TMPDIR и передается в hasher автоматически. Это можно проверить таким образом:
```
$gear --hasher -- echo
/tmp/.private/user/gear.OuNNNWJV/work/pkg.tar
```
Мы видим, что полный путь к pkg.tar передался команде echo как аргумент.

```shell
hsh-rebuild -v ./pkg.tar
```
Далее запустим hsh-rebuild, эта команда как раз запускает rpmbuild для сборки
пакета внутри созданного ранее chroot окружения. Ей можно передать опцию
--rpmbuild-args чтобы указать дополнительные аргументы для rpmbuild.

```shell
hsh-rmchroot -v
```
В конце, для удаления chroot файлов, нужно использовать hsh-rmchroot, так как у
нашего пользователя нет прав на удаление файлов, которые принадлежат
пользователям builder и rooter, созданных для работы hasher при настройке
командой hasher-useradd.
