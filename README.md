# SAM-Registry-edit
Documentation and instructions for manual editing of SAM in Windows 

======================================
                       EN
======================================

This is a manual for editing the SAM registry hive using standard Windows utilities.

SAM is a database of users and groups. It stores password hashes, flags, names, SIDs, RIDs, and descriptions of users and groups.

This guide explains how to edit SAM by creating a Windows user as an example.
IMPORTANT! Note that for full user creation, other registry areas like ProfileList must also be edited, but this is NOT covered in this description.

This guide was created through personal research and experimentation.

You can use it for troubleshooting or experimenting with custom SIDs.

Always backup the SAM hive before making changes, as errors may render the user account unusable.


------------------- 1. Preparing SAM: -------------------
There are two convenient methods:
1. 
- Boot into another OS (LiveCD) or use a WinPE bootable USB
- Mount the system disk
- Open regedit, select HKEY_LOCAL_MACHINE
- File -> Load Hive -> SYSTEM_ROOT\Windows\System32\config\SAM
- When prompted for a name, enter "SAM_TEMP" (without quotes)

2.
- Open Registry Editor in the booted system and navigate to:
HKEY_LOCAL_MACHINE/SYSTEM/Setup
- Find and edit the "SetupType" value, set to "1", and "CmdLine" value, set to "cmd.exe", then reboot. After reboot, open regedit via cmd.

We do NOT consider the sethc.exe replacement method due to Windows behavior that terminates processes started via sethc replacement after a few minutes, including all child processes.

-------------------2. Structure Overview-------------------

In SAM\SAM\Domains\Account\Users you will see a structure similar to:

Users
|--- 000001f4
|--- 000001f5
|--- Names
    |--- Administrator
    |--- Guest

Explanation:
Folders named like 000001f4 are user RIDs in HEX.
RID is essentially the last digits of the SID, e.g., -1000 or -500.
The Names folder contains subfolders with usernames, each containing a default parameter (@) of type RID. More details in the following steps.


-------------------Finding an Available RID-------------------
- Go to SAM_TEMP\SAM\Domains\Account\Users
- Check existing folders (000001F4 - Admin, 000001F5 - Guest, etc.)
- Choose an available RID, e.g., 3F0 (1008)
- Remember it in HEX: 000003F0 (always 8 characters with leading zeros!)

-------------------Creating the Username-------------------

a) Names Branch:
- Navigate to SAM_TEMP\SAM\Domains\Account\Users\Names
- Create a new key with the username (e.g., "TESTUSER")
- Inside the key, locate the parameter named "(Default)"

Explanation: Here, the parameter TYPE (not its value) of the default parameter will indicate the user's RID.

- Now the critical part - changing the parameter type:
  - Right-click the parameter -> Export -> save as names.reg
  - Open the file in Notepad
  - Replace the line "@=hex(0)" with "@=hex(3f0)"
This is the RID in HEX without leading zeros!
  # The "@" symbol means "(Default)"
  - Save and close
  - Delete the parameter in regedit
  - Import the modified names.reg
  - Verify the parameter type changed to 0x3f0 (via Right-click -> Modify Binary Data)

-------------------Creating the RID Key-------------------
- Return to SAM_TEMP\SAM\Domains\Account\Users
- Create a new key named "000003F0" (your RID)
- Inside, create two binary parameters: "F" and "V"

IMPORTANT! I do NOT recommend creating these manually. I STRONGLY RECOMMEND copying them from another user. Preferably from Administrator (000001f4):
by exporting the RID key to a .reg file. Then edit the .reg file in Notepad, changing the key path from e.g., 000001f4 to 000003f0.
The following instructions assume using a COPIED TEMPLATE


-------------------Populating the F Parameter-------------------
- Find an existing user (e.g., Guest at 000001F5)
- Export its F parameter -> f_template.reg (or another name)
- Open in Notepad

IMPORTANT: Pay attention to offsets. If the final edited value has fewer bytes, pad with 00. This is MANDATORY! Otherwise, it will corrupt the parameter. The same applies if the final byte count exceeds the original.


------------------- Explanation -------------------
This explanation may be INACCURATE and may VARY.
- Basic settings:
 
  00000000: 03 02 14 00 00 00 00 00 00 00 00 00 00 00 00 00
  00000010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  
- Modify only flags (offset 0x08):
  - 0x15 - account enabled
  - 0x11 - password not required
  - 0x10 - standard account
- Save as new F, import into registry

------------------- Changing RID in F -------------------
Using any hex converter, convert the original user's RID and find this value in the string. For example, for Administrator (template we used) it would be:
F4 01 00 00
Then replace this string with your RID in the same format. In our case:
F0 03 00 00 (0x3F0)



-------------------Populating the V Parameter-------------------
a) Create the V parameter using the same method as F

b) Repeat the same RID replacement step here
c) Using the same principle as with RID, you can edit the SID.
In our example, look for the string S-1-5-21 followed by the identifier and RID.
Below is how S-1-5-21 would appear:
53 00 2D 00 31 00 2D 00 35 00 2D 00 32 00 31 00
Or
01 05 00 00 00 00 00 05 15 00 00 00
d) Repeat similarly for Name, description, etc. if needed.
For the name in our example, look for something like the following for Administrator:
10 04 3C 04 34 04 38 04 3D 04 38 04 41 04 42 04 40 04 30 04 42 04 3E 04 40 04
Or
D0 90 D0 BC D0 B4 D0 B8 D0 BD D0 B8 D1 81 D1 82 D1 80 D0 B0 D1 82 D0 BE D1 80
Or if you have English localization:
41 00 64 00 6D 00 69 00 6E 00 69 00 73 00 74 00 72 00 61 00 74 00 6F 00 72 00
Or
41 64 6D 69 6E 69 73 74 72 61 74 6F 72



e) Password:
- Keep hashes zeroed (all FF) - user without password
- If password is needed - use only syskey-compatible utilities

6. Finalization:
- Save the modified V and F parameters
- Import into registry
- In regedit: Select SAM_TEMP root -> File -> Unload Hive
- Reboot into the system

Critical nuances:
1. Field lengths: Always check data size. If the username is shorter than the template - pad remaining space with zeros (00 00). If longer - increase the total size of the V-parameter.

2. RID in Names: This is not a regular value! The parameter TYPE MUST match the RID in hexadecimal. Regedit doesn't allow this via GUI - only through .reg file editing.

Fact:
SAM does NOT contain default system user data, but using this documentation you can create an entry for e.g., NT AUTHORITY\SYSTEM.


======================================
                       RU
======================================

Это инструкция или документация по ручном редактированию куста реестра SAM с помощью стандартных программ windows.

SAM - Это база данных пользователей и групп. В нем хранятся хэши паролей, флаги, имя, SID, RID, описание пользователей и групп. 

В данной инструкции я объясню как редактировать SAM на примере создания пользователя Windows.
ВАЖНО! Стоит учесть что для полноценного создания пользователя придется редактировать и другие места реестра вроде ProfileList, но в данном описании это НЕ будет описано.

Данная инструкция была создана мной путем личного изучения и эксперементов. 

Вы можете использовать ее для решения проблем или эксперментов с кастомными  SID-ами

Важно делать резервные копии раздела  SAM иначе ошибки приведут к неработоспособности пользователя.


------------------- 1. Подготовка SAM: -------------------
Существует 2 удобных вариант развития:
1. 
- Загрузись в другой ОС (LiveCD) или используй загрузочную флешку с WinPE
- Монтируй системный диск
- Открой regedit, выдели HKEY_LOCAL_MACHINE
- Файл -> Загрузить куст -> SYSTEM_ROOT\Windows\System32\config\SAM
- При запросе имени введи "SAM_TEMP" (без кавычек)

2.
- Откройте редактор реестра в загруженой системе и перейдите по пути:
HKEY_LOCAL_MACHINE/SYSTEM/Setup
- Найдите и отредактируйте параметры "SetupType" установите "1" и "CmdLine" установите на "cmd.exe" и перезагрузитесь. После перезагрузки откройте regedit через cmd.
 
Мы НЕ рассматриваем способ с заменой sethc.exe в связи поведения Windows, которое заключается в закрытии программы открытой заменой sethc после нескольких минут, включая все дочерние процессы.

-------------------2. Описание строения-------------------

В разделе SAM\SAM\Domains\Account\Users Вы можете увидеть структуру похожую на:

Users
|--- 000001f4
|--- 000001f5
|--- Names
    |--- Администратор
    |--- Гость

Пояснение:
Разделы имени рода 000001f4 это RID пользователей в HEX.
RID проще говоря это последние цифры в SID например -1000 или -500. 
В разделе Names хранятся разделы с именем пользователя а в них параметр по умолчанию (@) с типом  RID. об этом будет подробнее указано на следующих этапах.


-------------------Поиск свободного RID-------------------
- Перейди в SAM_TEMP\SAM\Domains\Account\Users
- Посмотри существующие папки (000001F4 - Админ, 000001F5 - Гость и т.д.)
- Выбери свободный RID, например 3F0 (1008)
- Запомни его в HEX: 000003F0 (всегда 8 символов с ведущими нулями!)

-------------------Создание имени пользователя-------------------

а) Ветка Names:
- Перейди в SAM_TEMP\SAM\Domains\Account\Users\Names
- Создай новый раздел с именем пользователя (например "TESTUSER")
- Внутри раздела найди параметр с именем "(По умолчанию)"

Пояснение: Здесь у параметра по умолчанию  ТИП параметра (а не его значение) будет указывать на RID пользователя.

- Теперь самое важное - меняем тип параметра:
  - ПКМ на параметре -> Экспортировать -> сохрани как names.reg
  - Открой файл в Блокноте
  - Замени строку @=hex(0) на "@=hex(3f0)
Это вой RID в HEX без нулей!
  # символ "@" означает "(По умолчанию)"
  - Сохрани, закрой
  - Удали параметр в regedit
  - Импортируй исправленный names.reg
  - Проверь что тип параметра стал 0x3f0 (через ПКМ -> Изменить двоичные данные)

-------------------Создание раздела  RID-------------------
- Вернись в SAM_TEMP\SAM\Domains\Account\Users
- Создай новый раздел с именем "000003F0" (твой RID)
- Внутри создай два двоичных параметра: "F" и "V"

ВАЖНО! Я НЕ рекомендую вам создавать их вручную. Я НАСТОЯТЕЛЬНО РЕКОМЕНДУЮ скопировать их у другого пользователя. Лучше всего у Администратора (000001f4):
путем экспорта в .reg раздела с RID. Далее занимаемся редактированием раздела в . reg открыв его через блокнот изменив путь раздела с например 000001f4 на в нашем случае 000003f0.
Дальнейшее описание инструкции будет происходить НА ОСНОВЕ СКОПИРОВАННОГО ШАБЛОНА


-------------------Заполнение параметра F-------------------
- Найди существующего пользователя (например Гостя в 000001F5)
- Экспортируй его параметр F -> f_template.reg  (или другое имя)
- Открой в блокноте

ВАЖНО: Важно смещение. Если по каким-то причинам итоговое количсетво байтов отредактированого значения заполните их 00. Это ОБЯЗАТЕЛЬНО! Иначе это повредит параметр. То же самое будет и если итоговое количество байтов окажется больше изначального.


------------------- Пояснение -------------------
Данное пояснение может быть НЕ ТОЧНЫМ и может ОТЛИЧАТСЯ.
- Базовые настройки:
 
  00000000: 03 02 14 00 00 00 00 00 00 00 00 00 00 00 00 00
  00000010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  
- Измени только флаги (offset 0x08):
  - 0x15 - аккаунт включен
  - 0x11 - пароль не требуется
  - 0x10 - стандартный аккаунт
- Сохрани как новый F, импортируй в реестр

------------------- Изменение RID в  F-------------------
С помощью различных программ или сервисов конвертируйте исходный RID  пользователя через hex преобразователь любой и найдите это значение в строке. Например для Администратора шаблон которого мы взяли это будет:
F4 01 00 00
Далее замените эту строку на свой RID в том же формате. В нашем случае это 
F0 03 00 00 (0x3F0)



-------------------Заполнение параметра V-------------------
а) Создайте параметр V по тому же способу что и F

б) По аналогичному шагу из F параметра повторите то же самое и тут
в) по такому же прицнепу как и с RID вы можете отредактировать SID. 
Для этого в нашем примере ищите строку S-1-5-21 потом индифактор и RID.
Ниже как будет выглядеть S-1-5-21
53 00 2D 00 31 00 2D 00 35 00 2D 00 32 00 31 00
Или
01 05 00 00 00 00 00 05 15 00 00 00
г) Повторите аналогично для Имени, описания и прочего если нужно. 
Для имени в нашем примере стоит искать что то вроде следщуюгео для Администратор.
10 04 3C 04 34 04 38 04 3D 04 38 04 41 04 42 04 40 04 30 04 42 04 3E 04 40 04
Или
D0 90 D0 BC D0 B4 D0 B8 D0 BD D0 B8 D1 81 D1 82 D1 80 D0 B0 D1 82 D0 BE D1 80
или если у вас английская локализация то
41 00 64 00 6D 00 69 00 6E 00 69 00 73 00 74 00 72 00 61 00 74 00 6F 00 72 00
Или
41 64 6D 69 6E 69 73 74 72 61 74 6F 72



д) Пароль:
- Оставь хэши нулевыми (все FF) - пользователь без пароля
- Если нужен пароль - используй только syskey-совместимые утилиты

6. Финализация:
- Сохрани измененный V параметр и F параметр
- Импортируй в реестр
- В regedit: Выдели корень SAM_TEMP -> Файл -> Выгрузить куст
- Перезагрузись в систему

Критические нюансы:
1. Длины полей: Всегда проверяй размер данных. Если имя пользователя короче шаблонного - заполняй оставшееся пространство нулями (00 00). Если длиннее - увеличь общий размер V-параметра.

2. RID в Names: Это не обычное значение! Тип параметра ДОЛЖЕН совпадать с RID в шестнадцатеричном виде. Regedit не позволяет это сделать через GUI - только через правку .reg-файла.

Факт:
SAM НЕ содержит данные системных пользователей по умолчанию, но путем данной документации вы сможете создать запись например  NT AUTHORITY\SYSTEM.
