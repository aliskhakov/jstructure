---
title: "Re Execution of Tasks in Rabbitmq"
date: 2017-11-12T18:17:54+03:00
draft: true
---

Во время выполнения задач, которые приходят подписчику из очереди в виде сообщения, возможны случаи сбоев и ошибок. При этом бывает, что задачу нужно попробовать выполнить некоторое количество раз, прежде чем удалить ее из очереди.

## Решение 1. Возврат сообщения в очередь.
При возникновении ошибки на стороне подписчика можно отклонить сообщение (*reject* или *nack* с параметром `requeue=true`), оставив сообщение в очереди. Если очередь объявлена с параметром `no_ack=false` и подписчик не выполняет *ack*, сообщение также возвращается в очередь. При возврате сообщения в очередь ему автоматически добавляется заголовок `redelivered=true`. Количество возвратов при этом нельзя зафиксировать. По наличию этого заголовка можно понять только то, что сообщение уже обрабатывалось и по каким-то причинам возвратилось в очередь. Таким образом, вариантов лимитирования количества попыток обработки сообщений немного &mdash; либо 1 раз, либо 2 раза, либо возвращать сообщение в очередь до тех пор, пока оно не обработается без ошибки. Возможность появления дублей и потери сообщений в данном решении отсутсвуют.

## Решение 2. Публикация сообщения с подтверждением обрабатываемого сообщения.
Как вариант, можно заново опубликовать сообщение в очередь и добавить при этом в сообщение специальный заголовок для хранения количества "возвратов". После публикации сообщения нужно выполнить *ack* по исходному сообщению, в результате обработки которого произошла ошибка. Сохранность данных в этой реализации гарантируется, однако, возможны случаи **дублирования сообщения**. Это возможно в случае завершения работы подписчика между этапами публикации сообщения и выполнением действия *ack* по обрабатываемому сообщению.

## Решение 3. Подтверждение обрабатываемого сообщения с последующей публикацией.
Можно изменить порядок выполнения в предыдущем решении. Вначале выполнять *ack* по исходному сообщению и только потом опубликовать сообщение с необходимым заголовком. В этом случае возможна **потеря сообщения**, но дублирование сообщения полностью исключается.

## Решение 4. Использование Dead Letter Exchanges.
**RabbitMQ**, начиная с версии [2.8.0](https://www.rabbitmq.com/release-notes/README-2.8.0.txt), предоставляет функционал перенаправления сообщения из очереди в другую точку доступа (*exchange*), называемую *[Dead letter exchange](https://www.rabbitmq.com/dlx.html) (DLX)*. Этот функционал выходит за рамки протокола [AMQP](https://www.amqp.org/). Ниже перечисленны случаи, при которых это возможно.

 - Отклонение сообщения (*reject* или *nack*) с параметром `requeue=false`.
 - Истечение *TTL* сообщения.
 - Достижение лимита очереди.

*DLX* можно задать на стороне клиента для любой очереди, задав соответствующий аргумент `x-dead-letter-exchange` при объявлении очереди. Также это можно сделать и на стороне сервера.

Стоит отметить, что при публикации сообщения в другую очередь, сообщение не удаляется до тех пор, пока *dead-letter* очереди не подтвердят получение сообщений. Таким образом, непредвиденное выключение брокера во время сбоя может привести к **дублированию сообщения**. То есть *dead-letter* очередь получит сообщение, а подтверждение получения отправить не успеет, поэтому в первичной очереди сообщение останется.

Используя функционал *DLX*, можно реализовать повторную обработку сообщений. Пример показан в статье &laquo;[RabbitMQ delay retry/schedule with Dead Letter Exchange](https://medium.com/@kiennguyen88/rabbitmq-delay-retry-schedule-with-dead-letter-exchange-31fb25a440fc)&raquo;. Ниже приводится немного измененная реализация.

Необходимо:
 - создать две точки доступа: *WorkExchange* и *RetryExchange*;
 - cоздать очередь *WorkQueue* с параметрами `x-dead-letter-exchange=RetryExchange` и связать ее c *WorkExchange*;
 - cоздать очередь *RetryQueue* с параметрами `x-dead-letter-exchange=WorkExchange` и `x-message-ttl=300000`, и связать ее c *RetryExchange*.

Исходное сообщение попадает в очередь *WorkQueue*. Подписчик забирает сообщение и в случае сбоя выполняет *reject* (или *nack*) с параметром `requeue=false`. После этого сообщение попадает в очередь *RetryQueue*, откуда, спустя 300000 мс, отправляется, минуя точку доступа *WorkExchange*, обратно в очередь *WorkQueue*.

   [Dead Letter Exchanges]: <https://www.rabbitmq.com/dlx.html>
   [AMQP]: <https://www.amqp.org/>
   [RabbitMQ delay retry/schedule with Dead Letter Exchange]: <https://medium.com/@kiennguyen88/rabbitmq-delay-retry-schedule-with-dead-letter-exchange-31fb25a440fc>
