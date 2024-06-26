Сборка проекта из исходных кодов
================================

Получение исходного кода
------------------------

Извлеките исходный код из репозитория и перейдите в него.

.. code:: shell

    git clone git@github.com:FirebirdSQL/firebird.git
    cd firebird

Последняя команда переходит в корневой каталог исходных кодов. Далее будет рассматривать его как текущий каталог проекта. В примерах будем использовать ветку ``master``.

Кроме этого есть ветки, соответствующие релизам:

    * B1_5_Release
    * B2_0_Release
    * B2_1_Release
    * B2_5_Release
    * B3_0_Release
    * v4.0-release

Остальные ветки создаются для реализации новых функций и вливаются через Pull Request-ы
с целью обеспечения ревью кода.


Сборка под Linux
----------------

Необходимые пакеты для сборки в Ubuntu

.. code:: shell

   sudo apt install git autotools-dev build-essential automake cmake libtool zlib1g-dev libicu-dev libncurses-dev libkrb5-dev

Перед сборкой необходимо собрать скрипт конфигурирования "configure".
Он генерируется с помощью скрипта ``autogen.sh``, расположенного в корневой директории исходных кодов.

``autogen.sh`` зависит от ``GNU autotools``. В современных дистрибутивах эти утилиты
уже установлены, но ряд других платформ могут потребовать их установки.

Когда вы один раз получили скрипт ``configure`` можно уже не вызывать ``autogen.sh``.
Однако для целей разработки удобнее вместо ``configure`` всегда вызывать ``autogen.sh``, но с параметрами конфигурирования, т.к. ``autogen.sh`` все равно вызовет ``configure`` и передаст ему свои параметры. Рассмотрим некоторые из них, важные при сборке на машине разработчика. Полный набор можно увидеть в самом скрипте.

    ``--enable-developer`` - конфигурирует проект для сборки на машине разработчика. По умолчанию будет собираться отладочная версия, отключены оптимизации и предопределен макрос ``DEBUG``.

    ``--enable-binreloc`` - позволяет перемещаемый исполнимый код. Если кратко, то это позволит вам запускать исполняемые файлы прямо из каталога сборки без предварительной установки.

После конфигурирования останется лишь вызвать команду ``make``. Весь набор команд выглядит следующим образом.

.. code:: shell

    ./autogen.sh
    make

``make install`` требуется если вы хотите установить собранный проект в систему.

При вызове ``make`` полезно указать опцию -jN, где N количество ядер процессора, которые вы готовы выделить для многопоточной сборки проекта. Это существенно ускоряет сборку. Разумно использовать N равным количеству ядер вашего компьютера. Например,

.. code:: shell

    make -j8

Другим параметром команды ``make`` может стать конфигурация сборки: ``release`` или ``debug``.
Если проект сконфигурирован с опцией ``--enable-developer``, то по умолчанию ``make`` будет собирать ``debug`` сборку, иначе ``release``. Вы можете без переконфигурирования собрать любую сборку, явно указав. Например,

.. code:: shell

    make -j8 debug
    make -j8 release

Соберет обе сборки и вы сможете отлаживать, запускать и профилировать любую из них.

Результат отладочной сборки будет располагаться в каталоге ``./gen/Debug/firebird``

.. code:: shell

    % ls ./gen/Debug/firebird/
    bin/              fbtrace.conf      intl/             replication.conf
    databases.conf    firebird.conf     lib/              security5.fdb
    examples/         firebird.log      misc/             tests/
    fb_init           firebird.msg      plugins/          tzdata/
    fbsrvreport.txt   include/          plugins.conf

А результат релизной сборки будет располагаться в каталоге ``./gen/Release/firebird``

.. code:: shell

    % ls ./gen/Release/firebird/
    bin/              fbtrace.conf      intl/             replication.conf
    databases.conf    firebird.conf     lib/              security5.fdb
    examples/         firebird.log      misc/             tests/
    fb_init           firebird.msg      plugins/          tzdata/
    fbsrvreport.txt   include/          plugins.conf

В релизной сборке включены все основные оптимизации, однако отладочные символы не выделены из исполняемых файлов.


Сборка под Windows
------------------

Сборка и разработка осуществляется в основном с использованием MS Visual Studio Community Edition.
Поддерживается 2017 и выше. После установки с поддержкой разработки на C++, необходимо прописать переменные окружения.

VS170COMNTOOLS - для сборки с помощью Visual Studio 2022.

VS160COMNTOOLS - для сборки с помощью Visual Studio 2019.

VS150COMNTOOLS - для сборки с помощью Visual Studio 2017.

Успешность можно проверить в командной строке. Для Visual Studio 2022 например должно получиться следующее.

.. code:: shell

    >echo %VS170COMNTOOLS%
    C:\Program Files\Microsoft Visual Studio\2022\Community\Common7\Tools

При сборке используются дополнительные утилиты. Один из способов их установки заключается в использовании пакета msys2. 
Получить его можно здесь: https://www.msys2.org/

После установки по умолчанию, добавить путь к исполняемым файлам ("c:\msys64\usr\bin") в переменную среды PATH.
В результате должно получиться похожее на

.. code:: shell

    >echo %PATH%
    C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\msys64\usr\bin;C:\Users\roman\AppData\Local\Microsoft\WindowsApps;

В извлеченном каталоге исходных кодов, перейдем в builds\win32 и выполним там команды сборки отладочной версии:

.. code:: shell

    cd builds\win32
    run_all DEBUG

В результате в каталоге исходных кодов должен появиться каталог ``output_x64_debug``.

