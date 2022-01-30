# Natch Quickstart

*(актуально на момент 31.01.2022)*

## Введение

Данное руководство является кратким курсом по освоению основных принципов использования системы определения ширины и глубины поверхности атаки Natch (разработчик - ИСП РАН). В руководстве рассмотрены вопросы создания виртуализованных сред выполнения объекта оценки (далее - ОО) в формате [QEMU](https://wiki.qemu.org/Main_Page), запуска данных сред под контролем Natch, анализа информации о движении помеченных данных, получаемой Natch от контролируемой виртуализованной среды.

## 1. Общие вопросы

### 1.1. Общие принципы работы Natch

Natch (Network Application Tainting Can Help) - это инструмент для определения поверхности атаки, основанный на полносистемном эмуляторе Qemu. Основная функция Natch - получение списка модулей (исполняемых файлов и динамических библиотек) и функций, используемных системой во время выполнения задачи. Natch представляет собой набор плагинов для эмулятора Qemu.

Общие принципы работы Natch, доступные плагины, команды управления Natch и их параметры представлены в веб-странице руководства `Natch v.1.1 — QEMU documentation.html`, доступном в комплекте поставки, а также в [Wiki](https://fanta.intra.ispras.ru/mediawiki/natch).

### 1.2. Общие принципы подготовки виртуализированной среды в формате QEMU

Процесс подготовки виртуализированной среды выполнения ОО в общем случае состоит из следующих последовательных шагов:
 - создание образа эмулируемой операционной системы в формате диска [qcow2](https://en.wikipedia.org/wiki/Qcow) на основе базового дистрибутива ОС. Формат qcow2 позволяет эффективно формировать снэпшоты состояния файловой системы в произвольный момент выполнения виртулизованной среды функционирования;
 - сборка дистрибутива ОО с требуемыми параметрами, в частности с генерацией и сохранением отладочных символов;
 - помещение собранного дистрибутива ОО в виртуализированную среду выполнения;
 - подготовка команд запуска QEMU, обеспечивающих эмуляцию аппаратной составляющей среды функционирования, загрузку и выполнение компонент Natch. 

Процесс подготовки виртуализированной среды выполнения ОО в значительной степени сходен с процессом подготовки виртуализированной среды для анализа с помощью инструмента динамического анализа помеченных данных [Блесна](https://www.ispras.ru/technologies/blesna/) (разработчик - ИСП РАН), с точностью до подготовки команд запуск QEMU.

### 1.3. Пример подготовки виртуализированной среды в формате QEMU

#### Подготовка хостовой системы

Подготовим Linux-based рабочую станцию (далее - хост), поддерживающую графический режим выполнения (QEMU демонстрирует вывод эмулируемой среды выполнения в отдельном графическом окне, следовательно нам необходим графический режим). Рабочая станция может быть реализована в формате виртуальной машины. Данное руководство описывает действия пользователя, работающего в виртуальной машине VirtualBox (4 ядра, 8 ГБ ОЗУ) с установленной ОС Ubuntu20:04 (desktop-конфигурация). 

Установим требуемое системное ПО, в т.ч. qemu: 
```bash
sudo apt install  -y curl qemu-system
```

Скачаем на хост выбранный базовый дистрибутив ОС. Поскольку виртуальные машины QEMU в режиме полносистемной эмуляции (анализ выполняется именно в таком режиме (параметр `--enable-kvm` в строке запуска qemu должен отсутствовать отключен), поскольку нам требуется полная эмуляция процессора и, частично, периферии для сбора максимально полной отладочной информации) значительно замедляют выполнение анализируемой среды функционирования, рекомендуется использовать минимальный образ - уменьшение числа установленных, стартующих при запуске служб, позволит значительно сократить нагрузку на процессор и сократить время, требуемое на запись и анализ трасс в режиме полносистемной эмуляции. 
```bash
sudo apt install  -y curl qemu-system
```


## 1. Анализ образа системы, содержащего пресобранную с символами версию программы wget

### 1.1 Подготовка стенда 

Подготовить Linux-based рабочую станцию, поддерживающую графический режим выполнения. Рабочая станция может быть реализована в формате виртуальной машины. Проверенной конфигурацией является Ubuntu20:04, 8 ядер, 8 ГБ ОЗУ. 

Установить на хост требуемое системное ПО, в т.ч. qemu: 
```bash
sudo apt install  -y curl qemu-system
```

Получить представление об общих принципах функционирования и командах управления эмулятора qemu - в [первоисточнике](https://qemu-project.gitlab.io/qemu/system/quickstart.html) или в [переводе](http://onreader.mdl.ru/KVMVirtualizationCookbook/content/Ch01.html).

Создать каталог ~/natch_quickstart (название выбрано произвольно), скачать комлпект поставки Natch в данный каталог:
```bas
cd  ~/natch_quickstart && \
curl -o Natch_documentation.pdf 'https://nextcloud.ispras.ru/s/raFWeX6B7XgYWDt/download?path=%2FNatch%20v.1.1&files=Natch_Documentation.pdf&downloadStartSecret=w0qrqfp8d9' && \
curl -o libs.zip 'https://nextcloud.ispras.ru/s/raFWeX6B7XgYWDt/download?path=%2FNatch%20v.1.1&files=libs_1804_natch_release_latest.zip&downloadStartSecret=shqvee1d2de' && \
curl -o plugins.zip 'https://nextcloud.ispras.ru/s/raFWeX6B7XgYWDt/download?path=%2FNatch%20v.1.1&files=qemu_plugins_1804_natch_release_latest.zip&downloadStartSecret=j0vq3mla9o8' && \
unzip libs.zip && \
unzip plugins.zip && \
rm -f libs.zip plugins.zip
```

Ознакомиться с документацией на Natch в файле `Natch_Documentation.pdf`, а также [Natch Wiki](https://fanta.intra.ispras.ru/mediawiki/natch), в частности получить представление:

- об основных принципах работы Natch
- об основных видах анализа и реализующих их плагинах
- о типовых командах Natch

Скачать [обучающий комплект](https://nextcloud.ispras.ru/s/o4w2mSqYAP4xFY3), включающий в себя в том числе qemu-образ, содержащий пресобранную с отладочными символами версию программы wget. Наличие символов позволит Natch формировать более читабельные отчеты, подставляя имена реальных функций анализируемой программы вместо адресов в памяти. Скачать образ можно командой: 

```bash
cd  ~/natch_quickstart && \
curl -o wget_image.zip https://nextcloud.ispras.ru/s/o4w2mSqYAP4xFY3/download && \
unzip wget_image.zip && \
rm -f wget_image.zip && \
mv 'For Dmitriy' wget_image # Временное название каталога на сервере надо будет заменить на что-то общеупотребительное
```

### 1.2 Запись трассы выполнения

1. Выполнить ...

```
Хорошо бы выполнить chmod +x natch_run.sh && ./natch_run.sh
```

### 1.3 Анализ записанной трассы выполнения

1. Выполнить ... 

