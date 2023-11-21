
# Task 1

Код говорит сам за себя:
![[assets/task1_nonstrip.png]]

`getString()` просто аллоцирует динамически память под строчку:
![[assets/task1_dynalloc.png]]

username: john
password: the ripper

![[assets/task1_fr.png]]

Пароль для прохождения теста: 987654321

# Task 2 (Python PYC)

Гуглеж быстро привел к тулзе [pycdc](https://github.com/zrax/pycdc), которая мастерски справилась со своей задачей:
![[assets/pyc_decompiled.png]]

Более того, она еще и [торчала](https://aur.archlinux.org/packages/pycdc-git) в AUR'е :)
```bash
yay -S pycdc-git
```

Применяем стандартный подход к реверсингу алгоритма:
![[assets/pyc_ruby.png]]

Проверяем:
![[assets/pyc_answer.png]]