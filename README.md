# Расчёт высоконагруженного сервиса  


### 1 Тема  
VPN provider.  

### 2 Определение возможного диапазона нагрузок подобного проекта.
### 3 Выбор планируемой нагрузки как 30% рынка в России.
<a href="https://www.gfk.com/ru/insaity/press-release/issledovanie-gfk-proniknovenie-interneta-v-rossii-1/">количество пользователей интернета в России</a> = 90000000  
<a href="https://www.vpnmentor.com/blog/vpn-use-data-privacy-stats/">количество пользователей VPN в России</a> = 24%  
<a href="https://vc.ru/flood/44138-pochemu-rossiyskie-operatory-vozvrashchayut-bezlimitnyy-internet">средний трафик на мобильное устройство в месяц</a> = 8 Gb  
<a href="https://park.mail.ru/blog/topic/view/13345/">моделируемая для рынка в России</a> = 30%  
Получаем нагрузку в 20 Gb/s  

### 4 Логическая схема базы данных (без выбора СУБД)  
Особенности хорошего VPN провайдера:  
- Не хранит данные пользователей, не логгирует конечные адреса.  
- Предоставляет безлимитный доступ, только ограничивает максимальную скорость в зависимости от тарифа. По мнению МТС: "<a href="https://vc.ru/flood/44138-pochemu-rossiyskie-operatory-vozvrashchayut-bezlimitnyy-internet">Наши клиенты живут в цифровой среде и не хотят думать о лимитах на интернет, они воспринимают их как искусственные ограничения.</a>"  
  
Поэтому в базе хранится только чувствительная и важная информация: Логин, пароль пользователя, его текущий тариф в Kb/s.  

### 5 Физическая системы хранения (конкретные СУБД, шардинг, расчет нагрузки, обоснование реализуемости на основе результатов нагрузочного тестирования)

Расчёт нагрузки на базу данных:
VPN сервера делают 1 select в момент подключения клиента, Billing cистема делает update раз в месяц.  
<a href="https://pikabu.ru/story/kak_izmenilsya_pikabu_za_2018_god_6393562">среднее время посещения pikabu.ru</a> = 9 мин.  
Рассматрим этот развлекательный сайт как типичный случай использования. Пользователь будет переподключаться к VPN, генерируя 1 запрос в базу, раз в 9 минут.  
Получается, что наши потенциальные 6480000 пользователей, используя сервис единовременно, будут генерировать нагрузку 12000 rps select по индексу для проверки авторизации.

### 6 Выбор прочих технологий: языки программирования, фреймфорки, протоколы взаимодействия, веб-сервера и т.д. (с обоcнованием выбора)

Части, из которых состоит сервис:

- Сервис обеспечивает VPN. В качестве сервера выберем <a href="https://openvpn.net/">openvpn</a>.
- Сервис проверяет, заплатил ли пользователь за услугу. Нужна база данных с логинами / паролями / тарифами. Выберем Postgres как надёжное хранилище данных.
- OpenVPN должен ходить в сторонний сервис для авторизации. Для этого нужен плагин на C. См обоснование необходимости в 6.1.
- Так как писать логику шардирования и походов в базу данных на C сложно, нужен отдельный сервис. Он должен выдерживать 12000 rps. Golang способен справиться с такой нагрузкой и изменяется вслед за бизнес требованиями гораздо легче С. Для устойчивости используем 2 таких сервера на соседних серверах.
- мониторинг (TODO)

### 6.1 Выбор системы авторизации для OpenVPN
OpenVPN поддерживает авторизацию, для этого используются плагины. На данный момент <a href="https://community.openvpn.net/openvpn/wiki/RelatedProjects#Authentication">не существует</a> готового решения для подключения к PostgreSQL. Интерфейс плагинов авторизации - набор функций на С. Он <a href="https://github.com/OpenVPN/openvpn/blob/master/include/openvpn-plugin.h.in">хорошо задокументирован</a>. Есть <a href="https://github.com/fac/auth-script-openvpn">официальный легковесный плагин</a>, позволяющий запускать внешний исполняемый файл или скрипт. Он делает exec при каджой попытке авторизации, передавая описание пользователя через переменные окружения. Мы считаем, что пользователи равномерно переподключаются к 30 серверам, создавая на одном сервере 400 событий авторизации в секунду.  

Примитивный замер производительности exec:  

<pre><code>time bash -c ' \
    for ((i = 0; i < 400; i++)); do \
        python3 -c \
            "import os; print(os.environ);" \
        > "/dev/null"; \
    done \
';</code></pre>  

<pre><samp>real  8.429s  
user  7.095s  
sys   1.346s  
</samp></pre>  

показывает, что каждый раз инициализировать интерпретатор не получится. В то же время замеры для простого бинарника

<pre><code>time bash -c ' \
    for ((i = 0; i < 400; i++)); do \
        "/usr/bin/env" > "/dev/null"; \
    done \
';
</code></pre> 

<pre><samp>real  0.361s  
user  0.270s  
sys   0.109s  
</samp></pre>  

доказыает, что возможно запускать простой бинарник, хотя это и занимает от 20% до 50% 1 ядра. Так как всё равно придётся писать на компилируемом языке, то лучше всего написать свой плагин на C, используя <a href="https://github.com/fac/auth-script-openvpn">плагин для запуска скриптов</a> как  пример. Так как писать бизнес логику на С довольно долго и дорого, плагин должен:
- устанавливать постоянное соединение с backend сервером  
- упаковывать передаваемые данные о новом подключении в json  
- отправлять в сервер с логикой  
- асинхронно ожидать ответа  
- закрывать все соединения при выключении  

### 7 Расчет нагрузки и потребного оборудования  

Расчёт пропускной способности VPN.
Возьмём <a href="https://community.openvpn.net/openvpn/wiki/Gigabit_Networks_Linux">официальные замеры производительности</a>. Требуются сервера с аппаратной реализацией шифрования: с <a href="https://ru.wikipedia.org/wiki/Расширение_системы_команд_AES">набором инструкций AES-NI</a>. Это увеличивает производительность с 156 Mb/s до 750 Mb/s.
Учитывая моделируемую нагрузку 20Gb/s при одновременном подключении всех пользователей, необходимо 30 серверов Intel Xeon X5660 CPU @ 2.80GHz с сетевой картой gigabit ethernet (как в тесте).  

### 8 Выбор хостинга / облачного провайдера и расположения серверов  

По <a href="https://habr.com/ru/company/yandex/blog/258823/">опыту Yandex</a>, стоит выбрать Финляндию для расположения серверов. Преимущества: датацентры стоят дешевле, за счёт охлаждения уличным воздухом напрямую. Терпимый ping для европейской части России.  
<table><tbody>
<tr><td><a href="https://wondernetwork.com/pings">ping</a></td><td>Helsinki</td></tr>  
<tr><td>Moscow</td>       <td>28.15ms</td></tr>
<tr><td>Novosibirsk</td>  <td>94.05ms</td></tr>
<tr><td>St Petersburg</td><td>7.00ms</td></tr>  
<tr><td>Vladivostok</td>  <td>120.13ms</td></tr>
</tbody></table>  

Выбираем самый дешёвый способ: свои сервера в арендованных стойках.  

### 9 Схема балансировки нагрузки  

Так как наш сервис ориентируется на множество мобильных пользователей, на медленные длинноживущие соединения, то можно использовать DNS балансировку. Вся нагрузка приходится на VPN сервера, доступные по белому ip адресу непосредственно из сети.

### 10 Обеспечение отказоустойчивости  
(TODO)
