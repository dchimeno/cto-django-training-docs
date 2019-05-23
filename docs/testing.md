> Automated testing is an extremely useful bug-killing tool for the modern Web developer.

## For models and business logic

Unit Testing for models and business logic.

Example test case:

````python

from django.test import TestCase
from nodes.factories import NodeFactory
from nodes import models


class NodeTestCase(TestCase):
    def setUp(self):
        NodeFactory.create_batch(10)

    def test_node_has_a_name(self):

        for node in models.Node.objects.all():
            self.assertIsNotNone(node.name)

````

*Exercise*

Write some tests for Data and Stream models.


### Useful tests

When making a test for a specific view or function, it's important to query how many queries have been done
in order to avoid possible performance regressions in the future.

````python

class NodeTestCase(TestCase):
    def setUp(self):
        NodeFactory.create_batch(10)

    def test_node_only_one_query(self):
        with self.assertNumQueries(1):
            for node in models.Node.objects.all():
                self.assertIsNotNone(node.name)



````

**Exercise**:

Create a test for the queries made in section [project license](model.md#aggregated-queries)



# Other tests

It's also possible to integrate libraries to create selenium tests from Django but it's not covered here.



Main reference:

https://www.obeythetestinggoat.com/