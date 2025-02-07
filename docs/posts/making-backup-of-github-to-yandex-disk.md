---
created: 1970-01-01T06:00
updated: 2023-12-30T22:36
edited_seconds: 550
template: post.html
title: Бэкапим GitHub аккаунт в Yandex Disk
date: 2024-03-22
authors:
  - Zhymabek Roman
hide:
  - toc
tags:
  - linux
  - github
  - yandex
  - disk
  - cloud
  - backup
---
К сожалению даже Open-Source подвергнут столь сильному влиянию геополитическому положению мира (привет покупка GitHub Microsoft'ом - https://yandex.kz/turbo/overclockers.ru/s/blog/TEXHAPb/show/28902/github-uzhe-ne-tot-posle-pokupki-servisa-kompaniej-microsoft-github-razocharovyvaet-vse-bolshe-polzovatelej). Цитирую: "24 июля GitHub изменил правила размещения проектов, заблокировав тысячи аккаунтов жителей Крыма, Ирана и ряда других государств, против которых правительство США ввело какие-либо ограничительные санкции.". И эта вся волокита началась после покупки Гитхаба Мелкософтом. Ну да ладно, выбор у нас к счастью есть, тот же self-hosted Gitea, Gitlab.

Написал небольшой Python скрипт, с адаптацией на CI окружение. Все данные сжимаются с помощью ZSTD. Использует Yandex API для выгрузки бэкапов в Yandex Disk. Из ограничении - Яндекс позаботились чтобы их халявное детище не использовали для бэкапирование, там есть какие-то лимиты по загрузкам, то есть выше этого лимита нельзя загружать больше в диск. Этот лимит сбрасывается каждый месяц и действует только на WebDav и API, в официальном клиенте и веб морде все отлично работает без всего этого.

- [GitHub - ZhymabekRoman/github-backup-to-yandex: Cкрипт на языке Python 3, который позволяет загрузить бэкап всего аккаунта GitHub в Яндекс.Диск. Скрипт может быть использован в GitHub Action.](https://github.com/ZhymabekRoman/github-backup-to-yandex)

Весь процесс как его юзать есть в репозитории, чекаем, создаем issue если возникнут вопросы

Пища на размышление:
- [GitHub начал блокировать российских разработчиков. Пора искать альтернативы? | ISPsystem | Дзен](https://dzen.ru/a/YnKz1vP3uxXEe0x3)
- [С 13 апреля GitHub начал блокировать аккаунты российских компаний и разработчиков / Хабр](https://habr.com/ru/news/661113/)