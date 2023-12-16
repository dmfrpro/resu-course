# Task 5

Я сначала решил поставить `aflplusplus` с AUR, но быстро поняв, что он там кастрированный, послушал совет Артема и развернул в докере:

```bash
docker pull aflplusplus/aflplusplus
docker run -ti -v ~/Downloads/json_fuzz:/src aflplusplus/aflplusplus
```

Я подготовил следующие инпут файлики:
![[assets/a58_inputs.png]]

Далее компилируемся:

```bash
afl-cc main.c json_fuzz.c -o fuzz -lm
```

![[assets/a58_compile.png]]

И запускаемся:

```bash
afl-fuzz -i inputs/ -o outputs/ -- ./fuzz @@
```

![[a58_stats.png]]

За 15 минут `afl++` нашел 33 краша, го смотреть:
![[assets/a58_outputs.png]]

Из вменяемых - null pointer dereference в 179й строке:
![[assets/a58_nptr_deref.png]]

Еще пример - невалидный поинтер во `free()`:
![[assets/a58_invalid_ptr.png]]

Пробуем второй вариант, пусть лучше с санитайз - `AFL_USE_ASAN=1`:
```bash
AFL_USE_ASAN=1 afl-cc main.c json_fuzz.c -o fuzz_asan -lm
```

И запускаемся:

```bash
afl-fuzz -i inputs/ -o outputs_asan/ -- ./fuzz_asan @@
```

![[assets/a58_stats2.png]]

Статистика:
![[assets/a58_afl_stats.png]]

Пробуем запустить что-нибудь из сгенерированных аутпутов:
![[assets/a58_stripped_run.png]]

Сборка с ASAN также выдает варнинги:
![[assets/a58_compile_warns.png]]

Статистика:
![[assets/a58_afl_asan_stats.png]]

Вместе с инструментацией от ASAN параметры `execs_per_sec`, `execs_ps_last_min` меньше. Но при этом без асана `execs_done` больше.
![[assets/a58_execs.png]]

# Task 6

Копируем собранный с ASAN бинарь себе:
![[assets/a58_docker_cp.png]]

Итак, как AFL++, так и ASAN - оба ставили свои перехватчики:
![[assets/a58_handlers.png]]

AFL++ от себя вставил функции для code coverage - сохраняет статистику, какие блоки кода выполняются чаще, по каким бренчам прыжки чаще и т.д.

У ASAN из интересного помимо собственных реализаций `memcpy` и подобной телеметрии (с включенными листенерами) - memory poisoning:
![[assets/a58_mem_poisoning_docs.png]]

Вот пример:
![[assets/a58_poison_funcs.png]]

`__asan_region_is_poisoned()` вставлен в функции, где потенциально можно словить buffer overflow, остальне же функции/макросы, исходя из XREF'ов, вызываются в entry point - то есть инициализируют ASAN внутри бинаря.

# Task 7

В слайдах упоминались Frida и мой любимый QEMU, на случай если попался хейтер опенсорса и не дает сорцы. 

Для начала пофиксим багу и соберем поддержку QEMU себе:
![[assets/a58_bug.png]]

```bash
cd /AFLplusplus/qemu_mode
ln -s /usr/local/bin/afl-showmap ../afl-showmap
bash build_qemu_support.sh
```

Оно таки собралося:
![[assets/a58_build_finished.png]]

Запускаемся в _куему_, предварительно собрав бинарь самым обычным `gcc`:
```bash
gcc *.c -o program -lm -g
afl-fuzz -Q -i inputs/ -o outputs_qemu/ -- ./program @@
```

22 краша за 10 минут
![[assets/a58_results_qemu.png]]

Почитав про фриду, она делает инъекции (ассемблерные вставки) в динамически линкующиеся библиотеки, таким образом фрида должна быть намного быстрее, чем QEMU с его эмуляцией в юзерспейсе.

Собираем режим фриды себе:
```bash
cd /AFLplusplus/frida_mode
make
```

![[assets/a58_frida_build.png]]

Прога скомпиленная уже есть, просто запускаемся с флажком `-O`:
```bash
afl-fuzz -O -i inputs/ -o outputs_frida/ -- ./program @@
```

Я поспрашивал челов, кто уже сдал 5-8, и у них фрида нашла гораздо больше крашей, но в моем случае это не так, и честно сказать - я сам не знаю почему. При этом метрики `execs` с QEMU одинаковые, но парпаметры, например, `item geometry`, `findings` за исключением крашей - выше:
![[assets/a58_results_frida.png]]
# Task 8

`libfuzzer` - built-in фича шланга, поэтому на этот раз попробуем шлангом пособирать, но для начала поменяем немного `json_fuzz.c`:
![[assets/a58_patch.png]]

`main.c` нам не нужен, так как `main()` будет предоставлен `libfuzzer`'ом, поэтому все ок.
Все команды можно в [доке](https://llvm.org/docs/LibFuzzer.html) взять.

Собираемся и запускаемся без всего:
```bash
clang -O1 json_fuzz.c -o program -lm -g -fsanitize=fuzzer
./program
```

Словили ошибочку:
![[assets/a58_libfuzzer_0.png]]

Собираемся и запускаемся с ASAN:
```bash
clang -O1 json_fuzz.c -o program -lm -g -fsanitize=fuzzer,address
./program
```

>(Я) - О, упало!
>(Сосед, стоящий сзади) - еще встанет!
>(Я) - я импотент

![[assets/a58_impotent_meme.png]]

Обнаружился double free:
![[assets/a58_libfuzzer_1.png]]

Собираемся и запускаемся с UBSAN:
```bash
clang -O1 json_fuzz.c -o program -lm -g -fsanitize=fuzzer,signed-integer-overflow
./program
```

Такое ощущение, что в самом первом случае undefined behavior по этой же причине - оверфлоу инта:
![[assets/a58_libfuzzer_2.png]]

Собираемся и запускаемся с MSAN:
```bash
clang -O1 json_fuzz.c -o program -lm -g -fsanitize=fuzzer,memory
./program
```

Use of uninitialized value ошибочка:
![[assets/a58_libfuzzer_3.png]]

В итоге каждый санитайзер - под свои задачи, соотвественно и находят ошибки разного рода.