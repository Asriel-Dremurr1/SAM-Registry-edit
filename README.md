markdown
# Manual for Editing the SAM Registry Hive

> **⚠️ WARNING:** This is an advanced guide. Incorrect modifications can corrupt the SAM database and make user accounts unusable. Always back up the SAM hive before making changes.

## Table of Contents
- [English](#english)
- [Русский](#русский)

---

## English

### Introduction

This is a manual for editing the SAM registry hive using standard Windows utilities.

SAM is a database of users and groups. It stores password hashes, flags, names, SIDs, RIDs, and descriptions of users and groups.

This guide explains how to edit SAM by creating a Windows user as an example.

**IMPORTANT!** Note that for full user creation, other registry areas like `ProfileList` must also be edited, but this is **NOT** covered in this description.

This guide was created through personal research and experimentation. You can use it for troubleshooting or experimenting with custom SIDs.

### 1. Preparing SAM

There are two convenient methods:

#### Method 1: Using Another OS/WinPE
1. Boot into another OS (LiveCD) or use a WinPE bootable USB
2. Mount the system disk
3. Open `regedit`, select `HKEY_LOCAL_MACHINE`
4. **File** → **Load Hive** → `SYSTEM_ROOT\Windows\System32\config\SAM`
5. When prompted for a name, enter `SAM_TEMP` (without quotes)

#### Method 2: Using Setup Registry Keys
1. Open Registry Editor in the booted system and navigate to:
```

HKEY_LOCAL_MACHINE\SYSTEM\Setup

```
2. Find and edit:
   - `SetupType` value, set to `1`
   - `CmdLine` value, set to `cmd.exe`
3. Reboot
4. After reboot, open regedit via cmd

> **Note:** We do NOT consider the sethc.exe replacement method due to Windows behavior that terminates processes started via sethc replacement after a few minutes, including all child processes.

### 2. Structure Overview

In `SAM\SAM\Domains\Account\Users` you will see a structure similar to:

```

Users
|---000001f4
|---000001f5
|---Names
     |--- Administrator
     |--- Guest

```

**Explanation:**
- Folders named like `000001f4` are user RIDs in HEX
- RID is essentially the last digits of the SID (e.g., -1000 or -500)
- The `Names` folder contains subfolders with usernames, each containing a default parameter (`@`) of type RID

### Finding an Available RID

1. Go to `SAM_TEMP\SAM\Domains\Account\Users`
2. Check existing folders (`000001F4` - Admin, `000001F5` - Guest, etc.)
3. Choose an available RID, e.g., `3F0` (1008)
4. Remember it in HEX: `000003F0` (always 8 characters with leading zeros!)

### Creating the Username

#### a) Names Branch:

1. Navigate to `SAM_TEMP\SAM\Domains\Account\Users\Names`
2. Create a new key with the username (e.g., `TESTUSER`)
3. Inside the key, locate the parameter named `(Default)`

**Explanation:** The parameter **TYPE** (not its value) of the default parameter will indicate the user's RID.

4. Change the parameter type:
   - Right-click the parameter → **Export** → save as `names.reg`
   - Open the file in Notepad
   - Replace the line `@=hex(0)` with `@=hex(3f0)` (This is the RID in HEX without leading zeros!)
   - Save and close
   - Delete the parameter in regedit
   - Import the modified `names.reg`
   - Verify the parameter type changed to `0x3f0` (via Right-click → **Modify Binary Data**)

> **Note:** The `@` symbol means `(Default)`

### Creating the RID Key

1. Return to `SAM_TEMP\SAM\Domains\Account\Users`
2. Create a new key named `000003F0` (your RID)
3. Inside, create two binary parameters: `F` and `V`

> **⚠️ IMPORTANT!** I do **NOT** recommend creating these manually. I **STRONGLY RECOMMEND** copying them from another user (preferably from Administrator `000001f4`) by exporting the RID key to a `.reg` file. Then edit the `.reg` file in Notepad, changing the key path from (e.g., `000001f4`) to `000003f0`.

The following instructions assume using a **COPIED TEMPLATE**.

### Populating the F Parameter

1. Find an existing user (e.g., Guest at `000001F5`)
2. Export its `F` parameter → `f_template.reg` (or another name)
3. Open in Notepad

> **⚠️ IMPORTANT:** Pay attention to offsets. If the final edited value has fewer bytes, pad with `00`. This is **MANDATORY**! Otherwise, it will corrupt the parameter. The same applies if the final byte count exceeds the original.

#### Explanation

This explanation may be **INACCURATE** and may **VARY**.

- Basic settings:
```

00000000: 03 02 14 00 00 00 00 00 00 00 00 00 00 00 00 00
00000010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

```

- Modify only flags (offset `0x08`):
  - `0x15` - account enabled
  - `0x11` - password not required
  - `0x10` - standard account

- Save as new F, import into registry

#### Changing RID in F

Using any hex converter, convert the original user's RID and find this value in the string.

For example, for Administrator (template we used) it would be:
```

F4 01 00 00

```

Then replace this string with your RID in the same format. In our case:
```

F0 03 00 00  (0x3F0)

```

### Populating the V Parameter

a) Create the `V` parameter using the same method as `F`

b) Repeat the same RID replacement step here

c) Using the same principle as with RID, you can edit the SID

In our example, look for the string `S-1-5-21` followed by the identifier and RID.

Below is how `S-1-5-21` would appear:
```

53 00 2D 00 31 00 2D 00 35 00 2D 00 32 00 31 00

```
Or
```

01 05 00 00 00 00 00 05 15 00 00 00

```

d) Repeat similarly for Name, description, etc. if needed

For the name in our example, look for something like the following for Administrator:

**Cyrillic:**
```

10 04 3C 04 34 04 38 04 3D 04 38 04 41 04 42 04 40 04 30 04 42 04 3E 04 40 04

```
Or
```

D0 90 D0 BC D0 B4 D0 B8 D0 BD D0 B8 D1 81 D1 82 D1 80 D0 B0 D1 82 D0 BE D1 80

```

**English:**
```

41 00 64 00 6D 00 69 00 6E 00 69 00 73 00 74 00 72 00 61 00 74 00 6F 00 72 00

```
Or
```

41 64 6D 69 6E 69 73 74 72 61 74 6F 72

```

e) **Password:**
- Keep hashes zeroed (all `FF`) - user without password
- If password is needed - use only syskey-compatible utilities

### 6. Finalization

1. Save the modified `V` and `F` parameters
2. Import into registry
3. In regedit: Select `SAM_TEMP` root → **File** → **Unload Hive**
4. Reboot into the system

### Critical Nuances

1. **Field lengths:** Always check data size. If the username is shorter than the template - pad remaining space with zeros (`00 00`). If longer - increase the total size of the V-parameter.

2. **RID in Names:** This is not a regular value! The parameter **TYPE MUST** match the RID in hexadecimal. Regedit doesn't allow this via GUI - only through `.reg` file editing.

### Fact

SAM does **NOT** contain default system user data, but using this documentation you can create an entry for e.g., `NT AUTHORITY\SYSTEM`.

---

## Русский

### Введение

Это инструкция или документация по ручном редактированию куста реестра SAM с помощью стандартных программ Windows.

SAM - Это база данных пользователей и групп. В нем хранятся хэши паролей, флаги, имя, SID, RID, описание пользователей и групп.

В данной инструкции я объясню как редактировать SAM на примере создания пользователя Windows.

**ВАЖНО!** Стоит учесть что для полноценного создания пользователя придется редактировать и другие места реестра вроде `ProfileList`, но в данном описании это **НЕ** будет описано.

Данная инструкция была создана мной путем личного изучения и экспериментов. Вы можете использовать ее для решения проблем или экспериментов с кастомными SID-ами.

### 1. Подготовка SAM

Существует 2 удобных варианта:

#### Метод 1: Использование другой ОС/WinPE
1. Загрузись в другой ОС (LiveCD) или используй загрузочную флешку с WinPE
2. Монтируй системный диск
3. Открой `regedit`, выдели `HKEY_LOCAL_MACHINE`
4. **Файл** → **Загрузить куст** → `SYSTEM_ROOT\Windows\System32\config\SAM`
5. При запросе имени введи `SAM_TEMP` (без кавычек)

#### Метод 2: Использование ключей реестра Setup
1. Откройте редактор реестра в загруженной системе и перейдите по пути:
```

HKEY_LOCAL_MACHINE\SYSTEM\Setup

```
2. Найдите и отредактируйте:
   - Параметр `SetupType` установите `1`
   - Параметр `CmdLine` установите на `cmd.exe`
3. Перезагрузитесь
4. После перезагрузки откройте regedit через cmd

> **Примечание:** Мы НЕ рассматриваем способ с заменой sethc.exe в связи поведения Windows, которое заключается в закрытии программы открытой заменой sethc после нескольких минут, включая все дочерние процессы.

### 2. Описание строения

В разделе `SAM\SAM\Domains\Account\Users` Вы можете увидеть структуру похожую на:

```

Users
|---000001f4
|---000001f5
|---Names
     |--- Администратор
     |--- Гость

```

**Пояснение:**
- Разделы имени рода `000001f4` это RID пользователей в HEX
- RID проще говоря это последние цифры в SID например -1000 или -500
- В разделе `Names` хранятся разделы с именем пользователя а в них параметр по умолчанию (`@`) с типом RID

### Поиск свободного RID

1. Перейди в `SAM_TEMP\SAM\Domains\Account\Users`
2. Посмотри существующие папки (`000001F4` - Админ, `000001F5` - Гость и т.д.)
3. Выбери свободный RID, например `3F0` (1008)
4. Запомни его в HEX: `000003F0` (всегда 8 символов с ведущими нулями!)

### Создание имени пользователя

#### а) Ветка Names:

1. Перейди в `SAM_TEMP\SAM\Domains\Account\Users\Names`
2. Создай новый раздел с именем пользователя (например `TESTUSER`)
3. Внутри раздела найди параметр с именем `(По умолчанию)`

**Пояснение:** Здесь у параметра по умолчанию **ТИП** параметра (а не его значение) будет указывать на RID пользователя.

4. Меняем тип параметра:
   - ПКМ на параметре → **Экспортировать** → сохрани как `names.reg`
   - Открой файл в Блокноте
   - Замени строку `@=hex(0)` на `@=hex(3f0)` (Это RID в HEX без нулей!)
   - Сохрани, закрой
   - Удали параметр в regedit
   - Импортируй исправленный `names.reg`
   - Проверь что тип параметра стал `0x3f0` (через ПКМ → **Изменить двоичные данные**)

> **Примечание:** Символ `@` означает `(По умолчанию)`

### Создание раздела RID

1. Вернись в `SAM_TEMP\SAM\Domains\Account\Users`
2. Создай новый раздел с именем `000003F0` (твой RID)
3. Внутри создай два двоичных параметра: `F` и `V`

> **⚠️ ВАЖНО!** Я **НЕ** рекомендую вам создавать их вручную. Я **НАСТОЯТЕЛЬНО РЕКОМЕНДУЮ** скопировать их у другого пользователя (лучше всего у Администратора `000001f4`) путем экспорта в `.reg` раздела с RID. Далее занимаемся редактированием раздела в `.reg` открыв его через блокнот изменив путь раздела с например `000001f4` на в нашем случае `000003f0`.

Дальнейшее описание инструкции будет происходить **НА ОСНОВЕ СКОПИРОВАННОГО ШАБЛОНА**.

### Заполнение параметра F

1. Найди существующего пользователя (например Гостя в `000001F5`)
2. Экспортируй его параметр `F` → `f_template.reg` (или другое имя)
3. Открой в блокноте

> **⚠️ ВАЖНО:** Важно смещение. Если по каким-то причинам итоговое количество байтов отредактированного значения заполните их `00`. Это **ОБЯЗАТЕЛЬНО**! Иначе это повредит параметр. То же самое будет и если итоговое количество байтов окажется больше изначального.

#### Пояснение

Данное пояснение может быть **НЕ ТОЧНЫМ** и может **ОТЛИЧАТЬСЯ**.

- Базовые настройки:
```

00000000: 03 02 14 00 00 00 00 00 00 00 00 00 00 00 00 00
00000010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

```

- Измени только флаги (offset `0x08`):
  - `0x15` - аккаунт включен
  - `0x11` - пароль не требуется
  - `0x10` - стандартный аккаунт

- Сохрани как новый F, импортируй в реестр

#### Изменение RID в F

С помощью различных программ или сервисов конвертируйте исходный RID пользователя через hex преобразователь любой и найдите это значение в строке.

Например для Администратора шаблон которого мы взяли это будет:
```

F4 01 00 00

```

Далее замените эту строку на свой RID в том же формате. В нашем случае это:
```

F0 03 00 00  (0x3F0)

```

### Заполнение параметра V

а) Создайте параметр `V` по тому же способу что и `F`

б) По аналогичному шагу из F параметра повторите то же самое и тут

в) По такому же принципу как и с RID вы можете отредактировать SID

Для этого в нашем примере ищите строку `S-1-5-21` потом идентификатор и RID.

Ниже как будет выглядеть `S-1-5-21`:
```

53 00 2D 00 31 00 2D 00 35 00 2D 00 32 00 31 00

```
Или
```

01 05 00 00 00 00 00 05 15 00 00 00

```

г) Повторите аналогично для Имени, описания и прочего если нужно

Для имени в нашем примере стоит искать что-то вроде следующего для Администратор:

**Кириллица:**
```

10 04 3C 04 34 04 38 04 3D 04 38 04 41 04 42 04 40 04 30 04 42 04 3E 04 40 04

```
Или
```

D0 90 D0 BC D0 B4 D0 B8 D0 BD D0 B8 D1 81 D1 82 D1 80 D0 B0 D1 82 D0 BE D1 80

```

**Английская локализация:**
```

41 00 64 00 6D 00 69 00 6E 00 69 00 73 00 74 00 72 00 61 00 74 00 6F 00 72 00

```
Или
```

41 64 6D 69 6E 69 73 74 72 61 74 6F 72

```

д) **Пароль:**
- Оставь хэши нулевыми (все `FF`) - пользователь без пароля
- Если нужен пароль - используй только syskey-совместимые утилиты

### 6. Финализация

1. Сохрани измененный `V` параметр и `F` параметр
2. Импортируй в реестр
3. В regedit: Выдели корень `SAM_TEMP` → **Файл** → **Выгрузить куст**
4. Перезагрузись в систему

### Критические нюансы

1. **Длины полей:** Всегда проверяй размер данных. Если имя пользователя короче шаблонного - заполняй оставшееся пространство нулями (`00 00`). Если длиннее - увеличь общий размер V-параметра.

2. **RID в Names:** Это не обычное значение! Тип параметра **ДОЛЖЕН** совпадать с RID в шестнадцатеричном виде. Regedit не позволяет это сделать через GUI - только через правку `.reg`-файла.

### Факт

SAM **НЕ** содержит данные системных пользователей по умолчанию, но путем данной документации вы сможете создать запись например `NT AUTHORITY\SYSTEM`.
