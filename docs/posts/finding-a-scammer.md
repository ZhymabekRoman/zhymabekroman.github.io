---
template: post.html
title: "Как меня пытались развести, но что-то пошло не так..."
date: 2025-03-29
authors:
  - Zhymabek Roman
tags: scam olx kazakhstan
hide: [toc]
---

Здравствуйте. Хочу поделится с вами информацией о скамере, кторый хотел украсть данные кредитной карты. Я как хобби занимаюсь разработкой ПО и увелкаюсь OSINT, и переодическии я выкладываю на площадке OLX объявления. Спустя пару дней как я выложил и купил VIP пакет для объявления мне написал один человек с предложением перейти в Telegram. Я не стал отвечать, т.к. сразу было ясно что он пытается меня развести, только спустия 3-5 дней я все таки решил ответить ему, и заодно попытатся задеононить его. 

Продавал я аквариум, цена была около 100к тенге. Схема очень простая - тебя просят перейти в Телеграм, дальше тебе отправляют ссылку, где надо ввести данные своей карты, и все деньги уходят скамеру.

И так по порядку. Прикладываю скрины ниже:



Telegram: @whwvnsn - ID: 6308353080, Estimated registration date: October 2023. 

Домен который мне он скинул чтобы я заполнил данные карты своей - KzOperations.online. IP: 194.58.112.174, Created on 2023-10-07, можно сказать создан почти вместе с Telegram аккаунтом. Поиск по WHOIS не дал каких либо зацепок по этом домену, но удалось найти очень схожии домен который был в работе пол года назад, но уже с дригм TLD - post-kz.kzoperations.shop. Система URL scan определила данный домен как фишенговый - https://urlscan.io/result/aac994ab-5183-4a61-9837-0ed6dbe6fbf9#summary. Работал за Cloudflare - 188.114.96.3, created 2023-02-02 - https://2ip.io/a/post-kz.kzoperations.shop/ . Немного поисках в интернете удалось найти WHOIS запись данного домена - https://service.loxotrona.net/check/post-kz.kzoperations.shop . Почти все данные мусорные, мы просто отметаем их, но обращаем внимание на почту - badboy8905@gmail.com . Пытаемся reverse lookup WHOIS по данной почте, и мы полчаем аж 276 результата (https://whoxy.com/email/163061384
и 77 результатов по этой же почте но в другом сервис (https://viewdns.info/reversewhois/?q=badboy8905%40gmail.com). На основе этих данных мы предпологаем что человек начал заниматся этим делом примерно с августа 2022 года. 

Пытаемся пробить инфу через OSINT утилиты:

badboy8905@gmail.com:
1. Google
    Name: —
    GAIA ID: 108070244582627351651
    👤 Plus Profile (https://web.archive.org/web/*/plus.google.com/108070244582627351651)

2. COMB
    Email:Assword:
    badboy8905@email.com:sasha00
    badboy8905@gmail.com:Ghost_99gg
    badboy8905@gmail.com:PaTrOnAs0000
    badboy8905@gmail.com:sasha00
    badboy8905@mail.ru:PaTrOnAs0000
    badboy8905@mail.ru:sasha00
    badboy8905@yandex.ru:sasha00 -> Account not found in Yandex Login

3. OkRu
    Name part: Ghost A***, 23 old - ссылку на профиль не удалось откопать к сожалению
    Location: —
    Phone part: —
    Registered: 4 ноября 2018

📥  Аккаунты «lolz.guru» [2018]:
├ ID: 70489
├ Ник: Ghost_99gg
├ Пол: male
├ Часовой пояс: Europe/Athens
├ Email: badboy8905@gmail.com
├ Дата регистрации: 2017-01-04 11:03:00
└ Последняя активность: 2017-01-04 11:03:14
  RAW: (70489, 'Ghost_99gg', 'badboy8905@gmail.com', 'male', '', 2, 0, 'Europe/Athens', 1, 1, 2, '', 2, 2, 0, 10, 1483527780, 1483527794, 0, 5, 0, 0, 0, '', 'valid', 0, 0, 0, 0, 0, 0, '0.00', '0.00', '0.00', '', 1, '', 0, 0, NULL, 0, 0, '', 1, 1, 0)

🏤  Новая Почта Украина [2016-2022]:
├ Адрес: Ponornytsya , Kovpaka 27
├ Город: Koropskyi
├ Страна: Ukraine
├ Телефон: +380636252658
├ Email: badboy8905@gmail.com
├ ФИО: Alexsandr Chertopolokh
├ Адрес доставки: Depot №1: vul. Gagarina, 2
├ Город доставки: Ponornytsia
└ Страна доставки: Ukraine

├ Адрес: 27 Ковпака 27, Новая Почта 1 Понорница
├ Город: SMTPonornitsa
├ Страна: Ukraine
├ Телефон: +380636252658
├ Email: badboy8905@gmail.com
├ ФИО: Чертополох Александр Юрьевич
├ Адрес доставки: Depot №1: vul. Gagarina, 2
├ Город доставки: Ponornytsia
└ Страна доставки: Ukraine

├ Адрес: 27 Ковпака 27, Новая Почта 1 Понорница
├ Город: SMTPonornitsa
├ Страна: Ukraine
├ Телефон: +380636252658
├ Email: badboy8905@gmail.com
├ ФИО: Чертополох Александр Юрьевич
├ Адрес доставки: Відділення №1: вул. Гагаріна, 2
├ Город доставки: смт. Понорниця, Чернігівська обл., Коропський р-н
└ Страна доставки: Ukraine

Тааакс, теперь мы имеем адресс и номер телефона, уже отлично. Пытаемся найти информацию по телефону +380636252658:

Getcontact - Сашко

1. Details
    Location: 🇺🇦 Ukraine 
    Carrier: Life:)

2. OkRu
    Name: OTPOK Cw, July 13 1997 (26 years old) - https://ok.ru/profile/567937246291 (https://web.archive.org/web/20231015060050/https://ok.ru/profile/567937246291)
    Location: Киев
    Email: badboy8905@gmail.com
    Registered: 4 февраля 2016

3. IDInfo
    Name: Miss Love

Тут ничего интересного. Пытаемся сделать поиск в Yandex по ФИО и мы видим результат обмен криптовалюты - https://clickpay.cx/ru/order/2WY4WU1WZRO . Ковыряемся в исходниках страницы и мы получаем тот же саммую почту и из новых данных номер карты и еще один номер телефона:

5375 4114 1209 6997 - Карта Universalbank (Mastercard, Debit) -> получить хоть какие-то данные из номера карты имея только OSINT под рукой думаю сложно, либо я просто не настолько опытный

+380954543439 - Получаем данные по Getcontakt, результат - Чертополох Саша и на этом же номере Telegram: @alex004k.

---

Поиск по паролю PaTrOnAs0000 удалось выйти на его другие почты на других доменах:
├1.badboy8904@mail.ru
├2.badboy8905@gmail.com
├3.badboy8905@mail.ru
└4.badboy8906@mail.ru

Также попытался сделать поиск по паролю sasha00, но получил уж слишком много номеров и почт (спасибо Глазу Бога), но остановился на парочку инетерсных номеров:
+380935603722 - Саша Мой -> Telegram: @chertopolokh_69 -> тут гадалке не ходи
+380965834883 - Саша -> Telegram: Name: C -> Сложно сказать его ли

badboy8906@mail.ru -> https://my.mail.ru/mail/badboy8906/ (https://web.archive.org/web/20231015055941/https://my.mail.ru/mail/badboy8906/) - Видим что человек (был) любитель игры contact wars, его профиль Odnoklassniki поддверждает это, как и возраст. Phone: +380966******, Name: саша иванов. Еще один номер который нам не известен. Еще на этой почте есть ICQ - https://icq.im/badboy8906@mail.ru , name: саша иванов

badboy8906@rambler.ru:4521688

badboy8905@mail.ru
badboy8904@mail.ru

---

Сложно сказать его ли Twitter, возможно использовал как приманку для наживки adult контентом:
https://twitter.com/badboy8906 (http://web.archive.org/web/20231015060816/https://twitter.com/badboy8906)

---

