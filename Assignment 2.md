# task 4

Бинарник берет строку "Hello" и 3 раза применяет алгоритм шифрования RC5 со следующими тремя строками:
![[assets/rc5_main.png]]

Соответственно адреса строк:
![[assets/rc5_3strs.png]]

Алгоритм удалось легко [нагуглить](https://en.wikipedia.org/wiki/RC5) по его двум константам `P` и `Q` соотвественно:
![[assets/rc5_p_q.png]]

Эталонную имплементацию RC5 на C я взял [отсюда](https://github.com/stamparm/cryptospecs/blob/master/symmetrical/sources/rc5.c)

Константы просто идеально совпадают с нашим случаем:
![[assets/rc5_const.png]]

Соответственно, `rc5_setup()` выглядит примерно так:
![[assets/rc5_setup.png]]

`rc5_encrypt()` выглядит так:
![[assets/rc5_encrypt.png]]

`rc5_decrypt()`, соответственно, так:
![[assets/rc5_decrypt.png]]

Очень даже похоже на эталон вышло, только `ROTR / ROTL` функции развернуты:
![[assets/rc5_github.png]]
