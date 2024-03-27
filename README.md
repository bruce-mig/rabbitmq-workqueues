# rabbitmq-workqueues

Work Queue that will be used to distribute time-consuming tasks among multiple workers.

The main idea behind Work Queues (aka: Task Queues) is to avoid doing a resource-intensive task immediately and having to wait for it to complete. Instead we schedule the task to be done later. We encapsulate a task as a message and send it to a queue. A worker process running in the background will pop the tasks and eventually execute the job. When you run many workers the tasks will be shared between them.

This concept is especially useful in web applications where it's impossible to handle a complex task during a short HTTP request window.

## Run with Docker.  

 `docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.13-management`

You need three consoles open. Two will run the worker.go script. These consoles will be our two consumers- C1 and C2.

```bash
# shell 1
go run worker/worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```

```bash
# shell 2
go run worker/worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```


In the third one we'll publish new tasks. Once you've started the consumers you can publish a few messages:

```bash
# shell 3
go run new_task.go First message.
go run new_task.go Second message..
go run new_task.go Third message...
go run new_task.go Fourth message....
go run new_task.go Fifth message.....
```

Typical output for round-robin dispatch:
```bash
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```

```bash
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

###  Forgotten acknowledgment
It's a common mistake to miss the ack. It's an easy error, but the consequences are serious. Messages will be redelivered when your client quits (which may look like random redelivery), but RabbitMQ will eat more and more memory as it won't be able to release any unacked messages.

In order to debug this kind of mistake you can use rabbitmqctl to print the messages_unacknowledged field:

```bash
docker exec -it rabbitmq /bin/sh
rabbitmqctl list_queues name messages_ready messages_unacknowledged
```