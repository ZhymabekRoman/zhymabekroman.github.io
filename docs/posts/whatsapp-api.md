---
template: post.html
title: Leverage Whatsapp like your own API!
date: 2024-06-21
authors:
  - Zhymabek Roman
tags:
  - web
  - api
  - whatsapp
hide:
  - toc
---
I have collected different variations of toolkits to make communicating with Whatsapp easier without creating a business account.
Use at your own risk! I'm not responsible for any misuse of the information provided here.

## Libraries
- [GitHub - pedroslopez/whatsapp-web.js: A WhatsApp client library for NodeJS that connects through the WhatsApp Web browser app](https://github.com/pedroslopez/whatsapp-web.js) - I think the most stable implementation of Whatsapp is based on the web version. Minimal reverse engineering goes on here (versus other implementations), so it's more likely to be stable than others.
- [GitHub - WhiskeySockets/Baileys: Lightweight full-featured typescript/javascript WhatsApp Web API](https://github.com/WhiskeySockets/Baileys) - reverse engineered implementation of Whatsapp. Communicates using Websockets. Pros: No browser simulation, Cons: Hight risk of getting banned. Problems - there's some problem with implementation itself, so it will require some time to get it working by writing plenty workaround above the library abstraction.
- [GitHub - sigalor/whatsapp-web-reveng: Reverse engineering WhatsApp Web.](https://github.com/sigalor/whatsapp-web-reveng) - description itself stands on its own.
- [GitHub - Auties00/Cobalt: Standalone unofficial fully-featured Whatsapp Web and Mobile API for Java and Kotlin](https://github.com/Auties00/Cobalt) - I don't know if it's still maintained, but it's a good start point for Java developers.
- [GitHub - wppconnect-team/wppconnect: WPPConnect is an open source project developed by the JavaScript community with the aim of exporting functions from WhatsApp Web to the node, which can be used...](https://github.com/wppconnect-team/wppconnect)
- [GitHub - Rhymen/go-whatsapp: WhatsApp Web API](https://github.com/Rhymen/go-whatsapp) - solution in Golang
- [GitHub - Neurotech-HQ/heyoo: Opensource python wrapper to WhatsApp Cloud API](https://github.com/Neurotech-HQ/heyoo) - solution in Python, I think it's outdated

## Third-party API solutions
- [WhatsApp Business Management API](https://developers.facebook.com/docs/whatsapp/business-management-api/get-started/)
- [WhatsApp Business API | Twilio](https://www.twilio.com/en-us/messaging/channels/whatsapp)
- [API for messengers gateway for sending messages, marketing campaigns and bots for PHP, JavaScript and Python](https://chat-api.com/en/) - looks kinda sus
- [WhatsMate - Easy API for WhatsApp messaging, Telegram messaging and translating languages](https://www.whatsmate.net/) 