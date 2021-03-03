# Как использовать локальные node-пакеты в качестве зависимостей проекта. #

перевод [статьи Генри Блей-Романа](https://www.viget.com/articles/how-to-use-local-unpublished-node-packages-as-project-dependencies/)

Тестируйте ещё не опубликованные пакеты, обходя подводные камни, на которые можно наступить при работе с npm и yarn.[ссылка](## мт,нч;)

Случается, что Вам нужно установить локальный пакет в качестве зависимости на своём проекте. Возможно Вы создаете новый node-модуль для публикации, трудитесь над новым пакетом для его использования только на своих личных проектах или внутри команды, либо участвуете в улучшении уже существующего пакета с открытым кодом. В любом случае, Вам нужно проверить выполненные изменения в коде, прежде чем делать pull-запрос в удаленный репозиторий или публиковать модуль. Как же лучше добавить локальную версию пакета в свой проект для такого тестирования?

Вы можете пушнуть изменения в удаленную ветку, и именно её использовать для установки зависимости. Например при работе с гитхабом Вы можете использовать
команду  `yarn add (npm install) <git remote url>#<branch/commit/tag>` . Но помимо интернет-соединения, такой подход требует постоянно переустанавливать пакет при каждом внесенном в него изменении.

Можно использовать команды `npm add relative/path или yarn add relative/path`, которые скопируют папку с пакетом в каталог с node-модулями проекта. Но этом изменения самого пакета никак не будут отслеживаться, да и относительный путь до папки с ним может быть очень громоздким.
(прим. пер. - в оригинале статьи указана команда `yarn add file:relative/path`, однако корректно работает именно приведенная в переводе, также в оригинале говорится о том, что npm не добавляет в список зависимостей в package.json название пакета при копировании, однако сейчас (01.03.21) -  это не так)

Ещё есть команды `npm link` и `yarn link`. Обе они добавляют в зависимости локальную символическую ссылку ([документация npm link](https://docs.npmjs.com/cli/v7/commands/npm-link), [документация yarn link](https://classic.yarnpkg.com/en/docs/cli/link)). Но у этого решения есть свои [сложности](https://github.com/yarnpkg/yarn/issues/1761) -  имплементации и yarn, и npm
могут доставить кучу хлопот (на момент написания статьи открыто [около 40 вопросов по npm link](https://github.com/npm/cli/search?q=npm-link&type=issues) (прим. пер. - на момент перевода их 227) и [более 150](https://github.com/yarnpkg/yarn/issues?utf8=%E2%9C%93&q=is:issue+is:open+%22yarn+link%22) для yarn link). Если вы уже использовали символические ссылки в качестве зависимостей при разработке своего пакета, наверняка Ваша работа стопорилась из-за какого-нибудь препятствия, будь то неожиданное `unlink` поведение,
проблемы с “peer”-зависимостями и что-то посерьезнее.

Что же тогда делать? Я для себя нашёл ответ - использовать [yalk](https://github.com/wclr/yalc), созданный [@whitecolor](https://medium.com/@_whitecolor).

yalc поддерживает свое хранилище на локальной машине (`~/.yalc`). Публикуете пакет с yalc, и полная копия пакета копируется в хранилище. Устанавливаете пакет из yalc-хранилища - и эта копия добавится в проект практически точно так как при установке пакета из удаленного каталога. Обновили добавленный в хранилище пакет - и изменения доступны для проекта, Вы даже можете опубликовать изменения в хранилище и обновить пакет в проекте одной командой. Во избежание конфликтов yalc хеширует каждую версию пакета, публикуемую в хранилище. И yalc может хранить столько версий пакета, сколько Вам захочется.

yalc позволяет просто и в интуитивно-понятной манере разрабатывать и тестировать пакеты локально. Он отвечает тем ожиданиям, которые Вы возможно возлагали на `(npm|yarn) link`. У yalc есть и другие полезные фичи - можете прочитать в [README](https://github.com/wclr/yalc) об использовании команды add, о продвинутой работой с git и многом другом.

Вот как использовать yalc для работы с локальными пакетами:

## Установка yalc ##

    $  npm install -g yalc   // или `yarn global add yalc`

## Размещение пакета в вашем локальном yalc-хранилище ##

    // в папке разрабатываемого пакета
    $  yalc publish

## Добавление в проект пакета из yalc-хранилища ##

    // в папке зависимого от пакета проекта
    $  yalc add <dependency name>

Если Вы посмотрите в package.json, то увидите, что туда была добавлена зависимость, с префиксом `file:.yalc/` в пути, например:

    “dependencies”: {
      “my-package”: “file:.yalc/my-package”
    }

yalc также добавляет файл `yalc.lock`, который хранит зависимости проекта, установленные с помощью yalc.  После того, как Вы с помощью команды `yalc add` добавите пакет my-package в проект, `yalc.lock` в его корневой папке будет
выглядеть следующим образом:

    {
      “version”: “v1”,
      “packages”: {
        “my-package”: {
	      “signature”:  “...”,     //  хэш-идентификатор версии зависимости
          “file”:  true    //  тип связи с yalc
	      }
      }
    }

yalc сам не устанавливает зависимости, указанные в package.json разрабатываемого Вами пакета. Поэтому, если таковые имеются, их нужно проинсталлировать в тестируемом проекте после добавления туда пакета:

    // в папке зависимого от пакета проекта
    $ npm install  //  или yarn

На момент написания статьи существует баг, при котором у зависимостей, установленный с помощью yalc, возникают проблемы с доступом:

    // в папке зависимого от пакета проекта
    $ <some command that relies on my-project>

    path/to/test-project/node_modules/.bin/my-package
    /bin/sh: path/to/test-project/node_modules/.bin/my-package: Permission denied
    error Command failed with exit code 126.
    info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.

Это можно исправить, обновив разрешение доступа у файла из сообщения об ошибке:

    $ chmod +x  <path from error>   
    // например, chmod +x path/to/test_project/node_modules/.bin/my-package

## Пуш изменений пакета в зависимый проект ##

После сохранения изменений в разрабатываемом модуле просто используйте команду `yalc push` из его корневой папки для пуша изменений в хранилище, и yalc автоматически обновит все зависимые от этого пакета проекты на Вашем компьютере.

    //  в папке разрабатываемого пакета
    $  yalc push  //  краткая запись команды   yalc  publish  --push

    Pushing my-package@<version> in path/to/my-package
    Pushing my-package@<version>-<hash8> added ==> path/to/my-project/node_modules/package.
    Pushing my-package@<version> in path/to/my-project2
    Pushing my-package@<version>-<hash8> added ==> path/to/my-project2/node_modules/package.
    my-package@<version>-<hash8> published in store.

Если Вам не нужно автоматическое обновление зависимостей во всех проектах, в которых установлен пакет (например, если Вам нужно, чтобы разные проекты работали с разными его версиями), используйте команды `yalc publish` и `yalc update`:

    //  в папке разрабатываемого пакета
    $  yalc publish
---
    //  в папке зависимого от пакета проекта
    $  yalc update 

Если будет необходимо, разрешите доступ для зависимости в тестируемом объекте после отработки команд `yalc push`, `yalc publish --push` и `yalc update` (см. выше по тексту).

При изменении зависимостей разрабатываемого пакета, переустановите модули для тестируемого проекта с помощью `npm i` или `yarn`.

На момент написания статьи команда `yalc push` иногда (очень редко) не обновляла файл package.json в папке установленного в тестируемый проект пакета. Если это произойдет, то запуск команды `yalc update` из директории проекта все исправит:

    //  в папке разрабатываемого пакета
    $  yalc push  //  краткая запись команды   yalc publish --push

    $ cd path/to/test-project
    $ <test command>
    Error: …
    $ cat node_modules/my-package/package.json
    #  эй, это не соответствует последним изменениям в path/to/my-package/package.json!
    $ yalc update
    $ cat node_modules/my-package/package.json
    # исправлено!
    $ <test command>
    # ошибка исчезла

Если это вдруг не помогло, переустановите зависимости в проекте, установленные с помощью yalc:

    //  в папке зависимого проекта
    $ yalc remove my-package
    $ yalc add my-package
    //  затем установите зависимости

## мт,нч;
(много текста, не читал)

<a name="nutchell"></a>[“Телеграфным слогом”](https://www.youtube.com/watch?v=A8M8qTclbho&ab_channel=BBC)("in a nutchell"), процесс работы с yalc таков:

    // установите yalc единожды
    // --------------

    $ npm install -g yalc # или yarn global add yalc

    // начните использовать yalc (или переключитесь на работу с ним)
    // -----------------------

    $ cd path/to/package
    my-package $ yalc publish

    // если проект уже имеет пакет в качестве зависимости
    project $ npm uninstall -S my-package # (или `yarn remove my-package`)

    project $ yalc add my-package

    // если my-package имеет зависимости
    project $ npm install # (или `yarn`)

    // разработка
    // -------

    project $
    // тестируйте проект, обновив сначала доступы, если ишью 21 не был пофиксен
    // перейдите в разрабатываемый пакет и внесите нужные изменения

    my-package $ yalc push

    // если зависимости my-package изменились,переустановите их в проекте
    project $ npm install # (или `yarn`)

    // тестируйте проект, изменяйте пакет, пуште с yalc, теституийте проект и т.д.


