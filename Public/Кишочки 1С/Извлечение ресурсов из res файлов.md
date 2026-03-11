
	Для использования - извлечь res файлы в кат лог tmp (или любой другой на выбор), скоипровать файл в скрипт и запустить
	
```sh
#!/bin/bash

# Скрипт для обработки .res файлов:
# 1. Для каждого .res файла создается подкаталог с его именем (без расширения).
# 2. Выполняется декомпиляция командой derb -i . -d <каталог> <файл.res>.
# 3. В созданном каталоге ищется файл <имя>.txt (предполагаемый результат derb).
# 4. В этом файле анализируются строки вида "имя_файла:binary { base64_data }".
# 5. Извлеченные base64 данные декодируются и сохраняются в файл с указанным именем
#    внутри того же подкаталога.

# Проверка наличия необходимых утилит
for cmd in derb base64; do
    if ! command -v "$cmd" >/dev/null 2>&1; then
        echo "Ошибка: утилита '$cmd' не найдена. Установите её и повторите попытку."
        exit 1
    fi
done

# Обработка всех .res файлов в текущем каталоге
shopt -s nullglob  # чтобы цикл не выполнялся, если нет .res файлов
for resfile in *.res; do
    # Получаем имя без расширения
    CURRENT=$(basename "$resfile" .res)
    echo ">>> Обработка $resfile -> каталог $CURRENT"

    # Создаем каталог (если его нет)
    mkdir -p "$CURRENT"

    # Декомпилируем .res файл в созданный каталог
    echo "    Запуск derb -i . -d \"$CURRENT\" \"$resfile\""
    if ! derb -i . -d "$CURRENT" "$resfile"; then
        echo "    Ошибка при выполнении derb для $resfile, пропускаем."
        continue
    fi

    # Переходим в каталог
    pushd "$CURRENT" >/dev/null || { echo "    Не удалось войти в $CURRENT"; continue; }

    # Предполагаем, что derb создал файл с именем CURRENT.txt
    txtfile="${CURRENT}.txt"
    if [[ ! -f "$txtfile" ]]; then
        echo "    Предупреждение: файл $txtfile не найден в $CURRENT, пропускаем анализ."
        popd >/dev/null
        continue
    fi

    echo "    Анализ $txtfile на наличие записей binary"

    # Используем awk для извлечения и декодирования base64 данных
    awk '
    function process(fname, data) {
        # Удаляем все пробельные символы из данных (base64 не должен их содержать)
        gsub(/[[:space:]]/, "", data);
        if (data == "") {
            print "        Предупреждение: пустые данные для " fname > "/dev/stderr";
            return;
        }
	# Определяем, похоже ли на hex (только 0-9A-F)
	    if (data ~ /^[0-9A-Fa-f]+$/) {
	        cmd = "xxd -r -p > \"" fname "\"";
	    } else {
	        cmd = "base64 -d > \"" fname "\"";
	    }
        print data | cmd;
        close(cmd);
        print "        Создан файл: " fname;
    }

    BEGIN { in_binary = 0; }

    # Поиск строки, содержащей ":binary {"
    /:binary[[:space:]]*\{/ {
        # Извлекаем имя файла (часть до двоеточия)
        colon_pos = index($0, ":");
        if (colon_pos == 0) next;
        filename = substr($0, 1, colon_pos - 1);
        gsub(/^[[:space:]]+|[[:space:]]+$/, "", filename);

        # Находим позицию открывающей фигурной скобки
        brace_pos = index($0, "{");
        if (brace_pos == 0) next;

        # Данные после скобки
        data_part = substr($0, brace_pos + 1);

        # Проверяем, есть ли закрывающая скобка на этой же строке
        close_pos = index(data_part, "}");
        if (close_pos > 0) {
            # Скобка есть – данные на одной строке
            data = substr(data_part, 1, close_pos - 1);
            process(filename, data);
            in_binary = 0;
        } else {
            # Скобки нет – начинаем многострочный сбор данных
            in_binary = 1;
            cur_filename = filename;
            cur_data = data_part;
        }
        next;
    }

    # Если мы внутри многострочного блока binary
    in_binary {
        # Ищем закрывающую скобку
        close_pos = index($0, "}");
        if (close_pos > 0) {
            # Добавляем часть строки до скобки
            before = substr($0, 1, close_pos - 1);
            cur_data = cur_data before;
            process(cur_filename, cur_data);
            in_binary = 0;
        } else {
            # Продолжаем накопление
            cur_data = cur_data $0;
        }
        next;
    }
    ' "$txtfile"

    # Возвращаемся в исходный каталог
    popd >/dev/null
done

echo "Готово."
```