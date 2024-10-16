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
- [GitHub - open-wa/wa-automate-nodejs: 💬 🤖 The most reliable tool for chatbots with advanced features. Be sure to 🌟 this repository for updates!](https://github.com/open-wa/wa-automate-nodejs)
- [GitHub - sigalor/whatsapp-web-reveng: Reverse engineering WhatsApp Web.](https://github.com/sigalor/whatsapp-web-reveng) - description itself stands on its own.
- [GitHub - Auties00/Cobalt: Standalone unofficial fully-featured Whatsapp Web and Mobile API for Java and Kotlin](https://github.com/Auties00/Cobalt) - I don't know if it's still maintained, but it's a good start point for Java developers.
- [GitHub - wppconnect-team/wppconnect: WPPConnect is an open source project developed by the JavaScript community with the aim of exporting functions from WhatsApp Web to the node, which can be used...](https://github.com/wppconnect-team/wppconnect)
- [GitHub - tulir/whatsmeow: Go library for the WhatsApp web multidevice API](https://github.com/tulir/whatsmeow)
- [GitHub - Rhymen/go-whatsapp: WhatsApp Web API](https://github.com/Rhymen/go-whatsapp) - solution in Golang

## Rest-API
- [WAHA](https://waha.devlike.pro/) - WAHA - WhatsApp HTTP API (REST API), under hood have two engines: chromium-based wweb-js and pure-websocket Baileys 
- [GitHub - chrishubert/whatsapp-api: This project is a REST API wrapper for the whatsapp-web.js library, providing an easy-to-use interface to interact with the WhatsApp Web platform.](https://github.com/chrishubert/whatsapp-api)
- [GitHub - ookamiiixd/baileys-api: Simple WhatsApp REST API with multiple device support](https://github.com/ookamiiixd/baileys-api)
- [GitHub - danielcardeenas/sulla: 👩🏻🔬 Javascript Whatsapp api library for chatbots](https://github.com/danielcardeenas/sulla)
  Child projects:
     - [GitHub - orkestral/venom: Venom is a high-performance system developed with JavaScript to create a bot for WhatsApp, support for creating any interaction, such as customer service, media sending, sentence recognition...](https://github.com/orkestral/venom)
     - [GitHub - wppconnect-team/wppconnect: WPPConnect is an open source project developed by the JavaScript community with the aim of exporting functions from WhatsApp Web to the node, which can be used...](https://github.com/wppconnect-team/wppconnect/)
     - [GitHub - open-wa/wa-automate-nodejs: 💬 🤖 The most reliable tool for chatbots with advanced features. Be sure to 🌟 this repository for updates!](https://github.com/open-wa/wa-automate-nodejs)

## Outdated
- [GitHub - danielcardeenas/whatsapp-framework: ⚗️Whatsapp python api](https://github.com/danielcardeenas/whatsapp-framework) - Whatsapp blocks numbers now. Framework wont work properly until next version
- [GitHub - Neurotech-HQ/heyoo: Opensource python wrapper to WhatsApp Cloud API](https://github.com/Neurotech-HQ/heyoo) - solution in Python
- [GitHub - Kalebu/alright: Python wrapper for WhatsApp web-based on selenium](https://github.com/Kalebu/alright)

## Third-party API solutions
- [API for messengers gateway for sending messages, marketing campaigns and bots for PHP, JavaScript and Python](https://chat-api.com/en/) - looks kinda sus
- [WhatsMate - Easy API for WhatsApp messaging, Telegram messaging and translating languages](https://www.whatsmate.net/) 
- [WhatsApp API for bulk messages, groups, channels, statuses](https://whapi.cloud/)
- [Unofficial WhatsApp API for Developers | Free Trial WhatsApp API](https://maytapi.com/)
### WABA (WahtsApp Buisness API)
- [WhatsApp Business Management API](https://developers.facebook.com/docs/whatsapp/business-management-api/get-started/)
- [WhatsApp Business API | Twilio](https://www.twilio.com/en-us/messaging/channels/whatsapp)
- [Gupshup - Conversational tools for customer engagement](https://www.gupshup.io/?ysclid=m2bh5xzkxi818861991)
- [WhatsApp Business Platform](https://www.infobip.com/whatsapp-business)
- [360dialog - Performance marketing for WhatsApp](https://www.360dialog.com/#)
- [Center your workflow through Conversational AI](https://sleekflow.io/)
- [Wati | Business Messaging Made Simple on Your Favourite App!](https://www.wati.io/)