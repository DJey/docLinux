# Dockerfile и коммуникация между контейнерами

В прошлой статье мы рассказали, [что такое Docker](http://habrahabr.ru/company/infobox/blog/237405/) и как с его помощью можно обойти Vendor–lock. В этой статье мы поговорим о Dockerfile как о правильном способе подготовки образов для Docker. Также мы рассмотрим ситуацию, когда контейнерам нужно взаимодействовать друг с другом.  
  
![](/images/ed05fa732898684249e799668b50ab64.jpg)  
В [InfoboxCloud](http://infoboxcloud.ru/) мы сделали готовый образ Ubuntu 14.04 с Docker. Не забудьте поставить галочку «Разрешить управление ядром ОС» при создании сервера, это требуется для работы Docker.  
  

#### **Dockerfile**

  
Подход **docker commit**, описанный в предыдущей статье, не является рекомендованным для Docker. Его плюс состоит в том, что мы настраиваем контейнер практически так, как привыкли настраивать стандартный сервер.   
  
Вместо этого подхода мы рекомендуем использовать подход **Dockerfile** и команду **docker build**. Dockerfile использует обычный DSL с инструкциями для построения образов Docker. После этого выполняется команда docker build для построения нового образа с инструкциями в Dockerfile.  
  

##### **Написание Dockerfile**

  
Давайте создадим простой образ с веб-сервером с помощью Dockerfile. Для начала создадим директорию и сам Dockerfile.  

```
mkdir static_web
cd static_web
touch Dockerfile

```

  
Созданная директория — билд-окружение, в которой Docker вызывает контекст или строит контекст. Docker загрузит контекст в папке в процессе работы Docker–демона, когда будет запущена сборка образа. Таким образом будет возможно для Docker–демона получить доступ к любому коду, файлам или другим данным, которые вы захотите включить в образ.  
  
Добавим в Dockerfile информацию по построению образа:  

```
# Version: 0.0.1
FROM ubuntu:14.04
MAINTAINER Yuri Trukhin <trukhinyuri@infoboxcloud.com>
RUN apt-get update
RUN apt-get install -y nginx
RUN echo 'Hi, I am in your container' \
        >/usr/share/nginx/html/index.html
EXPOSE 80

```

  
Dockerfile содержит набор инструкций с аргументами. Каждая инструкция пишется заглавными буквами (например FROM). Инструкции обрабатываются сверху вниз. Каждая инструкция добавляет новый слой в образ и коммитит изменения. Docker исполняет инструкции, следуя процессу:  

*   Запуск контейнера из образа
*   Исполнение инструкции и внесение изменений в контейнер
*   Запуск эквивалента docker commit для записи изменений в новый слой образа
*   Запуск нового контейнера из нового образа
*   Исполнение следующей инструкции в файле и повторение шагов процесса.

  
Это означает, что если исполнение Dockerfile остановится по какой-то причине (например инструкция не сможет завершиться), вы сможете использовать образ до этой стадии. Это очень полезно при отладке: вы можете запустить контейнер из образа интерактивно и узнать, почему инструкция не выполнилась, используя последний созданный образ.   
  
Также Dockerfile поддерживает комментарии. Любая строчка, начинающаяся с # означает комментарий.  
  
Первая инструкция в Dockerfile всегда должна быть **FROM**, указывающая, из какого образа нужно построить образ. В нашем примере мы строим образ из базового образа ubuntu версии 14:04.   
  
Далее мы указываем инструкцию **MAINTAINER**, сообщающую Docker автора образа и его email. Это полезно, чтобы пользователи образа могли связаться с автором при необходимости.   
  
Инструкция RUN исполняет команду в конкретном образе. В нашем примере с помощью ее мы обновляем APT репозитории и устанавливаем пакет с NGINX, затем создаем файл /usr/share/nginx/html/index.html.  
  
По-умолчанию инструкция RUN исполняется внутри оболочки с использованием обертки команд **/bin/sh -c**. Если вы запускаете инструкцию на платформе без оболочки или просто хотите выполнить инструкцию без оболочки, вы можете указать формат исполнения:  

```
RUN ["apt-get", "install", "-y", "nginx"]

```

  
Мы используем этот формат для указания массива, содержащего команду для исполнения и параметры команды.  
  
Далее мы указываем инструкцию EXPOSE, которая говорит Docker, что приложение в контейнере должно использовать определенный порт в контейнере. Это не означает, что вы можете автоматически получать доступ к сервису, запущенному на порту контейнера (в нашем примере порт 80). По соображениям безопасности Docker не открывает порт автоматически, но ожидает, когда это сделает пользователь в команде docker run. Вы можете указать множество инструкций EXPOSE для указания, какие порты должны быть открыты. Также инструкция EXPOSE полезна для проброса портов между контейнерами.  
  

##### **Строим образ из нашего файла**

  

```
docker build -t trukhinyuri/nginx ~/static_web

```

  
, где trukhinyuri – название репозитория, где будет храниться образ, nginx – имя образа. Последний параметр — путь к папке с Dockerfile. Если вы не укажете название образа, он автоматически получит название latest. Также вы можете указать git репозиторий, где находится Dockerfile.  

```
docker build -t trukhinyuri/nginx \ git@github.com:trukhinyuri/docker-static_web

```

  
В данном примере мы строим образ из Dockerfile, расположенном в корневой директории Docker.   
  
Если в корне билд контекста есть файл .dockerignore – он интерпретируется как список паттернов исключений.  
  

##### **Что произойдет, если инструкция не исполнится?**

  
Давайте переименуем в Dockerfile nginx в ngin и посмотрим.  
  
![](/images/4e29294aae9729e09d5aaf231b34ec6c.png)  
  
Мы можем создать контейнер из предпоследнего шага с ID образа 066b799ea548   
**docker run -i -t 066b799ea548 /bin/bash**  
и отладить исполнение.  
  
По-умолчанию Docker кеширует каждый шаг и формируя кеш сборок. Чтобы отключить кеш, например для использования последнего apt-get update, используйте флаг --no-cache.  

```
docker build --no-cache -t trukhinyuri/nginx

```

  
  

##### **Использования кеша сборок для шаблонизации**

  
Используя кеш сборок можно строить образы из Dockerfile в форме простых шаблонов. Например шаблон для обновления APT-кеша в Ubuntu:  

```
FROM ubuntu:14.04
MAINTAINER Yuri Trukhin <trukhinyuri@infoboxcloud.com>
ENV REFRESHED_AT 2014–10–16
RUN apt-get -qq update

```

  
Инструкция ENV устанавливает переменные окружения в образе. В данном случае мы указываем, когда шаблон был обновлен. Когда необходимо обновить построенный образ, просто нужно изменить дату в ENV. Docker сбросит кеш и версии пакетов в образе будут последними.  
  

##### **Инструкции Dockerfile**

  
Давайте рассмотрим и другие инструкции Dockerfile. Полный список можно [посмотреть тут](http://docs.docker.com/reference/builder/).  
  

###### **CMD**

  
Инструкция CMD указывает, какую команду необходимо запустить, когда контейнер запущен. В отличие от команды RUN указанная команда исполняется не во время построения образа, а во время запуска контейнера.   

```
CMD ["/bin/bash", "-l"]

```

  
В данном случае мы запускаем bash и передаем ему параметр в виде массива. Если мы задаем команду не в виде массива — она будет исполняться в /bin/sh -c. **Важно помнить, что вы можете перегрузить команду CMD, используя docker run.**  
  

###### **ENTRYPOINT**

  
Часто команду CMD путают с ENTRYPOINT. Разница в том, что вы не можете перегружать ENTRYPOINT при запуске контейнера.   

```
ENTRYPOINT ["/usr/sbin/nginx"]

```

  
При запуске контейнера параметры передаются команде, указанной в ENTRYPOINT.  

```
docker run -d trukhinyuri/static_web -g "daemon off"

```

  
Можно комбинировать ENTRYPOINT и CMD.  

```
ENTRYPOINT ["/usr/sbin/nginx"]
CMD ["-h"]

```

  
В этом случае команда в ENTRYPOINT выполнится в любом случае, а команда в CMD выполнится, если не передано другой команды при запуске контейнера. Если требуется, вы все-таки можете перегрузить команду ENTRYPOINT с помощью флага --entrypoint.  
  

###### **WORKDIR**

  
С помощью WORKDIR можно установить рабочую директорию, откуда будут запускаться команды ENTRYPOINT и CMD.  

```
WORKDIR /opt/webapp/db
RUN bundle install
WORKDIR /opt/webapp
ENTRYPOINT ["rackup"]

```

  
Вы можете перегрузить рабочую директорию контейнера в рантайме с помощью флага -w.  
  

###### **USER**

  
Специфицирует пользователя, под которым должен быть запущен образ. Мы можем указать имя пользователя или UID и группу или GID.  

```
USER user
USER user:group
USER uid
USER uid:gid
USER user:gid
USER uid:group

```

  
Вы можете перегрузить эту команду, используя глаг -u при запуске контейнера. Если пользователь не указан, используется root по-умолчанию.  
  

###### **VOLUME**

  
Инструкция VOLUME добавляет тома в образ. Том — папка в одном или более контейнерах или папка хоста, проброшенная через Union File System (UFS).  
Тома могут быть расшарены или повторно использованы между контейнерами. Это позволяет добавлять и изменять данные без коммита в образ.  

```
VOLUME ["/opt/project"]

```

  
В примере выше создается точка монтирования /opt/project для любого контейнера, созданного из образа. Таким образом вы можете указывать и несколько томов в массиве.  
  

###### **ADD**

  
Инструкция ADD добавляет файлы или папки из нашего билд-окружения в образ, что полезно например при установке приложения.  

```
ADD software.lic /opt/application/software.lic

```

  
Источником может быть URL, имя файла или директория.  

```
ADD http://wordpress.org/latest.zip /root/wordpress.zip

```

  

```
ADD latest.tar.gz /var/www/wordpress/

```

  
В последнем примере архив tar.gz будет распакован в /var/www/wordpress. Если путь назначения не указан — будет использован полный путь включая директории.  
  

###### **COPY**

  
Инструкция COPY отличается от ADD тем, что предназначена для копирования локальных файлов из билд-контекста и не поддерживает распаковки файлов:  

```
COPY conf.d/ /etc/apache2/

```

  
  

###### **ONBUILD**

  
Инструкция ONBUILD добавляет триггеры в образы. Триггер исполняется, когда образ используется как базовый для другого образа, например, когда исходный код, нужный для образа еще не доступен, но требует для работы конкретного окружения.  

```
ONBUILD ADD . /app/src
ONBUILD RUN cd /app/src && make

```

  
  

#### **Коммуникация между контейнерами**

  
[В предыдущей статье](https://infoboxcloud.ru/community/blog/iaas/149.html) было показано, как запускать изолированные контейнеры Docker и как пробрасывать файловую систему в них. Но что, если приложениям нужно связываться друг с другом. Есть 2 способа: связь через проброс портов и линковку контейнеров.  
  

##### **Проброс портов**

  
Такой способ связи уже был показан ранее. Посмотрим на варианты проброса портов чуть шире.   
Когда мы используем **EXPOSE** в Dockerfile или параметр **\-p номер\_порта** – порт контейнера привязывается к произвольному порту хоста. Посмотреть этот порт можно командой **docker ps** или **docker port имя\_контейнера номер\_порта\_в\_контейнере**. В момент создания образа мы можем не знать, какой порт будет свободен на машине в момент запуска контейнера.   
  
Указать, на какой конкретный порт хоста мы привяжем порт контейнера можно параметром docker run -p порт_хоста: порт_контейнера  
По-умолчанию порт используется на всех интерфейсах машины. Можно, например, привязать к localhost явно:  

```
docker run -p 127.0.0.1:80:80

```

  
Можно привязать UDP порты, указав /udp:  

```
docker run -p 80:80/udp

```

  
  

##### **Линковка контейнеров**

  
Связь через сетевые порты — лишь один способ коммуникации. Docker предоставляет систему линковки, позволяющую связать множество контейнеров вместе и отправлять информацию о соединении от одного контейнера другому.   
  
Для установки связи нужно использовать имена контейнеров. Как было показано ранее, вы можете дать имя контейнеру при создании с помощью флага --name.  
  
Допустим у вас есть 2 контейнера: web и db. Чтобы создать связь, удалите контейнер web и пересоздайте с использованием команды --link name:alias.   

```
docker run -d -P --name web --link db:db trukhinyuri/webapp python app.py

```

  
Используя docker -ps можно увидеть связанные контейнеры.  
  
Что на самом деле происходит при линковке? Создается контейнер, который предоставляет информацию о себе контейнеру-получателю. Это происходит двумя способами:  

*   Через переменные окружения
*   Через /etc/hosts

  
Переменные окружения можно посмотреть, выполнив команду env:  

```
$ sudo docker run --rm --name web2 --link db:db training/webapp env
    . . .
    DB_NAME=/web2/db
    DB_PORT=tcp://172.17.0.5:5432
    DB_PORT_5432_TCP=tcp://172.17.0.5:5432
    DB_PORT_5432_TCP_PROTO=tcp
    DB_PORT_5432_TCP_PORT=5432
    DB_PORT_5432_TCP_ADDR=172.17.0.5

```

  
Префикс DB_ был взят из alias контейнера.   
  
Можно просто использовать информацию из hosts, например команда ping db (где db – alias) будет работать.

**********
[docker](/tags/docker.md)
[Dockerfile](/tags/Dockerfile.md)
