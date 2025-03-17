---
template: post.html
title: "UI design - the bad and the good"
date: 2025-03-17
authors:
  - Zhymabek Roman
tags: ui
hide: [toc]
---

# Good
## Form action button still active when user doesn't select all fields
Very cool design logic in Github.com repository creating page. I didn't select owner, and after clicking "Create repository" it's shows me warning under owner input "Please select an owner". Most services probably would just make button disabled, until inner form validation logic wouldn't pass requirements.

As alternative I would suggest to hide all other junk stuff below description, and after selection owner and repository name it would just pop up under

![[Pasted image 20250104180222.png]]

# Bad
## Burger on right side, but sidebar itself on the left side
![[Pasted image 20250314012936.png]]

## Не разделять сущности по цвету/дизайну
### Example 1 - Mobile application Active
![[Pasted image 20250104180947.png]]

Загаловок написан - Выберите номер. Так а какого фига задается PAPA, то есть считай мой добавленый номер по дизайну в одном ряду с действиями? Ну вот я назвал номер PAPA, а что если назову "- DELETE A NUMBER" или же добавлю десять "CANCEL"?. Круто? Круто!

Второй прикол здесь - когда нажимаешь на Log out он тебя выбрасывает сразу же без дополнительных предупреждении. Тип прикинь ты хотел нажать на Cancel но "привет тачпад" экран у тебя залип и ты случайно тыкнул Log out.

### Example 2 - Bitrix
Тоже самое, сами смотрите:
![[Pasted image 20250104181742.png]]
Самое последнее это совсем другое окно, притом самого Битрикса, а верхние это уже пользовательские воронки. Я эту кнопку после гуглинга только смог найти, хотя вроде его местоположение правильное, но вот дизайн сам увы.