Сборка FLibrary под macOS
==========================

Скрипт build-mac.sh собирает FLibrary.app, добавляет HD-иконку, раскладывает
Qt/framework зависимости внутрь bundle, подписывает приложение ad-hoc подписью
и собирает DMG с FLibrary.app и ссылкой Applications для установки drag-and-drop.

Зависимости
-----------

Нужны Xcode Command Line Tools или Xcode, Homebrew, CMake, Ninja, Conan 2, Qt 6,
p7zip и librsvg:

    brew install cmake ninja conan qt qtwebengine qtvirtualkeyboard p7zip librsvg

qtwebengine и qtvirtualkeyboard нужны потому, что часть Qt image/input plugins
может ссылаться на QtPdf и QtVirtualKeyboard frameworks. Скрипт подтягивает эти
frameworks в .app, если соответствующие плагины попали в bundle.

Сборка
------

Обычная сборка:

    ./build-mac.sh

По умолчанию скрипт пробует собрать два DMG:

    arm64
    x86_64

Если зависимостей для одной архитектуры нет, она будет пропущена. Это удобно на
Apple Silicon: arm64 собирается из /opt/homebrew, а x86_64 будет пропущен, если
Intel Homebrew в /usr/local не установлен.

Собрать только Apple Silicon:

    ARCHS=arm64 ./build-mac.sh

Собрать только Intel:

    ARCHS=x86_64 ./build-mac.sh

Требовать, чтобы все запрошенные архитектуры обязательно собрались:

    ALLOW_MISSING_ARCHS=0 ./build-mac.sh

Результат
---------

Готовые образы появляются в корне репозитория:

    FLibrary-<version>-macOS-arm64.dmg
    FLibrary-<version>-macOS-x86_64.dmg

Внутри DMG лежит FLibrary.app и ссылка Applications. Пользователь может открыть
DMG и перетащить приложение в Applications.

Сборка x86_64 на Apple Silicon
------------------------------

Для x86_64 нужны именно x86_64-срезы Qt и p7zip. Обычно это Intel Homebrew,
установленный под Rosetta в /usr/local:

    arch -x86_64 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    arch -x86_64 /usr/local/bin/brew install cmake ninja conan qt qtwebengine qtvirtualkeyboard p7zip librsvg

После этого можно запускать:

    ./build-mac.sh

Если Qt или p7zip лежат нестандартно, пути можно указать явно:

    QT_PREFIX_X86_64=/usr/local/opt/qt \
    P7ZIP_DIR_X86_64=/usr/local/opt/p7zip/lib/p7zip \
    ./build-mac.sh

Для arm64 аналогичные переменные:

    QT_PREFIX_ARM64=/opt/homebrew/opt/qt
    P7ZIP_DIR_ARM64=/opt/homebrew/opt/p7zip/lib/p7zip

Проверки, которые делает скрипт
------------------------------

Скрипт сам выполняет:

    codesign --verify --deep --strict
    FLibrary.app/Contents/MacOS/FLibrary --version
    hdiutil verify FLibrary-<version>-macOS-<arch>.dmg

Также bundle проверяется на локальные absolute paths вроде /opt/homebrew,
/usr/local и /Users. Такие зависимости не должны оставаться внутри готового DMG.

Подпись и notarization
----------------------

Сейчас используется ad-hoc подпись:

    codesign --sign -

Этого достаточно для локальной проверки bundle, но для публичного релиза через
Gatekeeper нужен Developer ID certificate и notarization. Это отдельный release
шаг, не выполняемый build-mac.sh.

Частые проблемы
---------------

missing command: rsvg-convert

    brew install librsvg

skipping x86_64: Qt with x86_64 slice was not found

    Установите Intel Homebrew в /usr/local и поставьте x86_64 Qt/p7zip, либо
    укажите QT_PREFIX_X86_64 и P7ZIP_DIR_X86_64.

Finder layout was skipped

    На headless/CI окружениях Finder может не сохранить фон и позиции иконок.
    Это warning: DMG все равно содержит FLibrary.app и Applications symlink.
