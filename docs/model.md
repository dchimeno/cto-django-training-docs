
Django has a model centric approach and the underlying ORM is a heavy coupled part of the system.

In this part you'll learn:

* How to define relational models.
* How to create and perform migrations
* How to add them to the admin.
* Shell to the rescue (ipython + django_extensions)
* How to auto document relational model.
* How to auto generate development data for them.
* How to avoid common N+1 problem.
* Some aggregated queries.


It all start with a definition of a problem that we map to entities called models.

A simple model definition could be:


`````python
from django.db import models

class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()

    def __str__(self):
        return f"{self.headline} - {self.content}"
`````

Field reference: https://docs.djangoproject.com/en/2.1/ref/models/fields/#field-types


## Relations


There are three types of relations between entities:


1:1 --> OneToOneField

n:1 --> ManyToOne

n:m --> ManyToMany


Exercise:

Given the references for fields and relations, create three models for ``Node``,  ``Stream`` and ``Data`` with the description:

Node is like a IoT device that could has a unique ``name``, has two ``node_type``: white and black, could be ``enabled`` or  not (default is enabled) and has 0 or more ``Stream``

Stream is the channel for we can retrieve different data in nodes, has ``name``, ``enabled`` and a relation with Node.

Data is where we we store data for each Stream and has ``timestamp`` with a default value to the current timestamp, a ``data``
field where we actually stores floats, and a relation with ``Stream``


After making the models, we should create and execute the migrations:


```bash
docker-compose exec web python manage.py makemigrations
```

```bash
docker-compose exec web python manage.py migrate
```


## Admin

Now we want to display data in a way we can list, modify it and probably add some actions.

The admin is intented for staff, superusers, managers, etc. but never end clients (customers)

The admin is very extensible and customizable, it has a lot of magic, but it does the job 90% of the times
saving hours of creating a custom backoffice.


Exercise: 
    Now that we have the models (tables) created in the database, we will add them to the admin.
    Following  the below example, create the classes for 
    
    
`````python
from django.contrib import admin
from nodes.models import Node

class NodeAdmin(admin.ModelAdmin):
    pass
    
admin.site.register(NodeAdmin, NodeAdmin)

`````     

## Shell to the rescue

We just notice we can create, modify, delete... entries from the admin panel, but we are developers, right?

Thanks to ``ipython`` and ``django_extensions`` we'll have a productive shell to interact without code and data.

````bash
docker-compose exec web python manage.py shell_plus --ipython --print-sql
````

Now we have a shell history (Cmd+R), auto loaded models and most used functions, and we can see sql directly in our shell.


Let's try with some code:

````python

# Create a node
alcala_node = Node.objects.create(name='Alcalá', enabled=False, node_type='basic')

# See all nodes
Node.objects.all()

# Create a Stream attached  to the previous node.

riego_stream = Stream.objects.create(name='Riego', enabled=False, node=alcala_node)

# See all streams attached to node
Stream.objects.filter(node=alcala_node)

riego_data = Data.objects.create(data=2.3, stream=riego_stream)

````

## Auto generating ER model

This tool is provided by ``django_extensions`` and needs some system dependencies.

Its great when you are developing a project with a lot of relations to have a better understating of them.

Reference: https://django-extensions.readthedocs.io/en/latest/graph_models.html

In this example we use ``pydot`` because graphviz is more complex to install.

````bash
docker-compose exec web python manage.py graph_models -a -g -o docs/graph_models.png
````

### Django project with existing database

Until now we  have created the models given a problem/specification, but there are cases when the database is already
created (and probably with data). 

In this case, Django cover us with ``inspectdb``

````bash
docker-compose exec web python manage.py inspectdb
````



## Generate development data

Given we are working on a database driven project, we need data in order to develop and test our code.

A common and simple approach is copy production data and dump it in the DB, but that obviously comes with drawbacks:
 
 * Could contain sensitive data
 * Production systems move slowly than development, so there could be missing columns/tables.
 
Another approach is the use of fixtures, Django support them but:

 * They are static.
 * In case of large tables, the size is too much to put inside the repository.
 * If you use a small subset of data, you are probably missing some challenges (real use of indexes, N+1)..
 

There is another option to create dynamically data based on our previously created models and some annotated data.

We'll use the [Factory Boy](https://factoryboy.readthedocs.io/en/latest/) library


Main benefit:

````python
class FooTests(unittest.TestCase):

    def test_with_factory_boy(self):
        # We need a 200€, paid order, shipping to australia, for a VIP customer
        order = OrderFactory(
            amount=200,
            status='PAID',
            customer__is_vip=True,
            address__country='AU',
        )
        # Run the tests here

    def test_without_factory_boy(self):
        address = Address(
            street="42 fubar street",
            zipcode="42Z42",
            city="Sydney",
            country="AU",
        )
        customer = Customer(
            first_name="John",
            last_name="Doe",
            phone="+1234",
            email="john.doe@example.org",
            active=True,
            is_vip=True,
            address=address,
        )
        # etc.
````

Main drawbacks:

 * In some case we could end with duplicated code.
  
  
Real example:

````python
import datetime

import factory  # noqa
import factory.fuzzy
import pytz
from django.contrib.auth import get_user_model
from nodes.models import Data, Node, Stream

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    is_superuser = False
    is_staff = False

    username = factory.Faker('user_name')
    first_name = factory.Faker('name')

    @factory.lazy_attribute
    def email(self):
        return '%s@example.com' % self.username

````  
  
**Exercise**:

With the given example, create a factory for our models: Node, Stream and Data. 


More Reference: https://factoryboy.readthedocs.io/en/latest/examples.html 


## How to avoid common problems with relations

Django querysets are lazy, that means that query against the database is not executed until the last possible moment.

That means that we could easily trap into the N+1 problem.


`````python
# This block is 1 query for data
# and 20 queries for each stream that is lazy-loaded
data = Data.objects.all()[0:19]
for d in data:
    print(d.data, d.stream.name)

`````

Fix:

````python

# only 1 query
data = Data.objects.select_related('stream').all()
for d in data:
    print(d.data, d.stream.name)
````

Would result in
````sql
SELECT `nodes_data`.`id`,
       `nodes_data`.`timestamp`,
       `nodes_data`.`data`,
       `nodes_data`.`stream_id`,
       `nodes_stream`.`id`,
       `nodes_stream`.`name`,
       `nodes_stream`.`enabled`,
       `nodes_stream`.`node_id`
  FROM `nodes_data`
 INNER JOIN `nodes_stream`
    ON (`nodes_data`.`stream_id` = `nodes_stream`.`id`)
````

**select_related** is used for ``ForeignKey`` relations, for ``ManyToMany`` relations we should use **prefetch_related**


## Aggregated queries.


**Aggregate** calculates values for the entire queryset.

````python
Book.objects.aggregate(average_price=Avg('price'))
# {'average_price': 34.35} 
````

**Annotate** calculates summary values for each item in the queryset.

````python
q = Book.objects.annotate(num_authors=Count('authors'))
q[0].num_authors
# 2
````


Exercise.

How many Streams are with the name ``Riego``?

>>

How many water has been registered for those streams?

>> 

Which is the node who has registered more water?

>>








 
