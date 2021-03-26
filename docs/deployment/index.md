# Infrastructure

![](images/Infrastructure.png?raw=true)

<!-- Tu Graz Repository has **3** instances running on Debian (VMs).

## Development Instance
[invenio-dev01.tugraz.at](https://invenio-dev01.tugraz.at/)

Development instance has **1** VM:

1. Web Server/Services VM (invenio-dev01)


## Test Instance
[invenio-test.tugraz.at](https://invenio-test.tugraz.at/)

Test instance has **3** VMs:

1. Web Server VM (invenio01-test)
2. Web Server VM (invenio02-test)
3. Services VM (invenio03-test) -->

## Production Instance
[repository.tugraz.at](https://repository.tugraz.at/)

Production instance has **3** VMs:

1. Web Server VM (invenio01-prod)
2. Web Server VM (invenio02-prod)
3. Services VM (invenio03-prod)


These virtual machines are split into two categories, **Web Servers** and **Services**. In this guideline we will take a look on both:

### Web Server VM

Base image of the [repository](https://gitlab.tugraz.at/invenio/repository) instance.

* **[UWSGI](https://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html)**

* **[Celery](https://docs.celeryproject.org/en/stable/userguide/application.html)**

The web server VMs are configured to the F5 load balancer that is provided by [Tu Graz ZID](https://www.tugraz.at/tu-graz/organisationsstruktur/serviceeinrichtungen-und-stabsstellen/zentraler-informatikdienst/).

F5 load balancer is forwarding the requests to one of the (Web Server VM), and then the requests are proxied by [NGINX](https://gitlab.tugraz.at/invenio/nginx) to [UWSGI web applications](https://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html). As shown below:

![](images/invenio-prod.png?raw=true)

### Services VM
Services VM - consist of some services for the Repository such as:

* **[Elasticsearch](https://gitlab.tugraz.at/invenio/elasticsearch)** is a search engine based on the Lucene library.
It provides a distributed, multitenant-capable full-text search engine with an HTTP web interface and schema-free JSON documents. Elasticsearch is developed in Java.

* **[PostgreSQL](https://www.postgresql.org/)** is a free and open-source relational database management system (RDBMS) emphasizing extensibility and SQL compliance.

* **[Redis](https://gitlab.tugraz.at/invenio/cache)** Remote Dictionary Server is an in-memory data structure project implementing a distributed, in-memory key–value database with optional durability. Redis supports different kinds of abstract data structures, such as strings, lists, maps, sets, sorted sets, HyperLogLogs, bitmaps, streams, and spatial indexes.

* **[EXIM4](https://gitlab.tugraz.at/invenio/exim4)** Exim4 is a Message Transfer Agent (MTA) developed at the University of Cambridge for use on Unix systems connected to the Internet. Exim can be installed in place of sendmail, although its configuration is quite different.

* **[RabbitMQ](https://gitlab.tugraz.at/invenio/rabbitmq)** RabbitMQ is an open-source message-broker software that originally implemented the Advanced Message Queuing Protocol and has since been extended with a plug-in architecture to support Streaming Text Oriented Messaging Protocol, MQ Telemetry Transport.
