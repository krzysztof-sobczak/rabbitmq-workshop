# rabbitmq-workshop

## Installation

1. [Install docker](https://www.docker.com/products/overview#/install_the_platform) & [Install docker-compose](https://docs.docker.com/compose/install/)
2. Clone this repository somewhere `git clone git@github.com:olaurendeau/rabbitmq-workshop.git && cd rabbitmq-workshop`
3. Run containers in background `docker-compose up -d`
4. Check if :
  * app is properly working at [http://localhost:4446/](http://localhost:4446/)
  * RabbitMQ management interface is available at [http://guest:guest@localhost:15672/#/queues](http://guest:guest@localhost:15672/#/queues)

## Cheat sheet

### Do not get stucked !

All step are implemented in the following branches :
- master : code base
- step1/asynchronize-request : Step 1 implemented
- step2/fill-the-gap : Step 2 implemented
- step3/handle-failures : Step 3 implemented
- step4/dispatch-events : Step 4 implemented
- step5/push : Step 5 implemented
- step6/configuration : Step 6 implemented

### Docker

* `docker-compose up -d` build and run as daemon all containers define in [`docker-compose.yml`](https://github.com/olaurendeau/rabbitmq-workshop/blob/master/docker-compose.yml)
* `docker-compose restart {container}` restart a daemonized container e.g. `docker-compose restart worker` will restart the container `worker`
* `docker-compose logs -f {container}` display logs of a container e.g. `docker-compose logs -f worker` show logs of all container of type `worker`
* `docker-compose scale {container}={instance number}` e.g. `docker-composer scale worker=5` will daemonize 5 container of type `worker`
* `docker-compose run {container} {command}` e.g. `docker-compose run command php script.php` will run the command `php script.php` on the container `command`

### RabbitMQWrapper

* Publish a message `$rabbitmq->publish('exchangeName', new \Swarrot\Broker\Message(json_encode(['id' => uniqid(), ...])));`
* Create a temporary queue `$rabbitMQ->createTemporaryQueue('queue.log-catcher.'.uniqid(), ['exchangeName' => 'routingKey']);`

### Request / Response format

Request
```json
{
  "id":"894f8ad6-bbae-eb53-6fe3-f35c3e24e537",
  "method":"sendVerySlowEmail",
  "params":{
    "email":"john.doe@foobar.com",
    "channel":"14fa8b0c-945f-d52d-5e18-d91d37479de1"
  }
}
```

Response
```json
{
  "id":"894f8ad6-bbae-eb53-6fe3-f35c3e24e537",
  "result":"success"
}
```

## Workshop

### 1 - Asynchronize request

Create a queue `queue.document` and bind it to `amq.direct` exchange without routing_key

In `backend/api.php` instead of generating the invoice, publish the request as a json string on `amq.direct` exchange. Change result in response to `pending`. Using `backend/src/RabbitMQ/RabbitMQWrapper::publish` method should help.

Move InvoiceGenerator code to `backend/worker.php`.

Change `web/app.js` success message to something more "pending"

Run `docker-compose restart worker` to restart worker

### 2 - Fill the gap

Create a `backend/script.php` file generating 100 messages to `amq.direct` exchange.

Run `docker-compose run command php script.php` to produce 100 messages

Play with the mailer stoping and starting it `docker-compose stop mailer` / `docker-compose up -d mailer`

Scale workers `docker-compose scale worker=5`

### 3 - Handle failures

In `backend/worker.php` randomly (1 over 4) fail the processor by sending an exception

Swarrot library propose a [retry processor](https://github.com/swarrot/swarrot/tree/master/src/Swarrot/Processor/Retry), replace `Swarrot\Processor\ExceptionCatcher\ExceptionCatcherProcessor` with the `Swarrot\Processor\Retry\RetryProcessor`.
RetryProcessor will need a [`MessagePublisherInterface`](https://github.com/olaurendeau/rabbitmq-workshop/blob/master/backend/src/RabbitMQ/RabbitMQWrapper.php#L45-L51) to publish failed messages on `amq.fanout` exchange.

If stucked see [https://github.com/olaurendeau/rabbitmq-workshop/blob/step3/handle-failures/backend/worker.php](https://github.com/olaurendeau/rabbitmq-workshop/blob/step3/handle-failures/backend/worker.php)

In management UI, create a new queue `queue.document.retry` with following arguments :
* x-message-ttl:	5000
* x-dead-letter-exchange:	amq.direct
* x-dead-letter-routing-key:	

And bind it to `amq.fanout` exchange

Run `docker-compose restart worker` to restart worker

### 4 - Dispatch events

In `backend/src/Logger/Logger.php` publish messages like `{"id": "1234", "message": "[18:21:26] afe4da94-55bf-4dd0-1a3c-e38e03dbb8ac [worker] Message processed"}` on exchange `amq.topic` with routing key `log.{application}`

Copy `backend/worker.php` to a `backend/log-catcher.php`, build a LogProcessor that define a temporary queue with a unique name binded to `amq.topic` with default routing key `#` using `backend/src/RabbitMQ/RabbitMQWrapper::createTemporaryQueue`, consume from that queue and write the log message to standard output.

Run `docker-compose restart worker` to restart worker

Run `docker-compose run command bash` to ssh on command container

Play with `php log-catcher.php`

Optionnaly your script can take an application name as argument to filter on routing_key when binding the queue to exchange

### 5 - Push

In `backend/src/Logger/Logger.php` add `request_id` and `channel` in json payload

Have a look at `backend/pusher.php` shouldn't have much to do.

Run `docker-compose restart worker pusher` to restart worker & pusher

### 6 - Configuration

Have a look at https://github.com/olaurendeau/rabbit-mq-admin-toolkit-cli readme and define a `.rabbit-mq-admin-toolkit.yml` file defining our rabbitmq objects configuration

Run `docker-compose run command php rabbit-mq-admin-toolkit.phar define`
