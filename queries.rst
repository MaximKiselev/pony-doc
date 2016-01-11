﻿Queries
===============

Pony provides a very convenient way to query the database using the generator expression syntax. Pony allows programmers to work with objects which are stored in a database as if they were stored in memory, using native Python syntax. It makes development easier.

Pony ORM functions used to query the database
--------------------------------------------------------------------

.. py:function:: select(gen[, globals[, locals])

   This function translates the generator expression into SQL query and returns an instance of the :py:class:`Query` class. If necessary, you can apply any :py:class:`Query` method to the result, e.g. :py:meth:`Query.order_by` or :py:meth:`Query.count`. If you just need to get a list of objects you can either iterate over the result or get a full slice::

       for p in select(p for p in Product):
           print p.name, p.price

       prod_list = select(p for p in Product)[:]

   The ``select()`` function can also return a list of single attributes or a list of tuples::

       select(p.name for p in Product)

       select((p1, p2) for p1 in Product
                       for p2 in Product if p1.name == p2.name and p1 != p2)

       select((p.name, count(p.orders)) for p in Product)

   You can pass the ``globals`` and ``locals`` dictionaries for using them as global and local namespace.

   Also, you can apply the ``select`` method to relationship attributes. See :ref:`the examples <col_queries_ref>`

.. py:function:: get(gen[, globals[, locals])

   Extracts one entity instance from the database. The function returns the object if an object with the specified parameters exists, or ``None`` if there is no such object. If there are more than one objects with the specified parameters, raises the ``MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`` exception. Example::

       get(o for o in Order if o.id == 123)

   The equivalent query can be generated using the :py:meth:`Query.get` method::

       select(o for o in Order if o.id == 123).get()

.. py:function:: left_join(gen[, globals[, locals])

      The results of a left join always contain the result from the 'left' table, even if the join condition doesn't find any matching record in the 'right' table.

      Let's say we need to calculate the amount of orders for each customer. Let's use the example which comes with Pony distribution and write the following query:

      .. code-block:: python

          from pony.orm.examples.estore import *
          populate_database()

          select((c, count(o)) for c in Customer for o in c.orders)[:]

      It will be translated to the following SQL:

      .. code-block:: sql

          SELECT "c"."id", COUNT(DISTINCT "o"."id")
          FROM "Customer" "c", "Order" "o"
          WHERE "c"."id" = "o"."customer"
          GROUP BY "c"."id"

      And return the following result::

          [(Customer[1], 2), (Customer[2], 1), (Customer[3], 1), (Customer[4], 1)]

      But if there are customers that have no orders, they will not be selected by this query, because the condition ``WHERE "c"."id" = "o"."customer"`` doesn't find any matching record in the Order table. In order to get the list of all customers, we should use the ``left_join`` function:

      .. code-block:: python

          left_join((c, count(o)) for c in Customer for o in c.orders)[:]

      .. code-block:: sql

          SELECT "c"."id", COUNT(DISTINCT "o"."id")
          FROM "Customer" "c"
            LEFT JOIN "Order" "o"
              ON "c"."id" = "o"."customer"
          GROUP BY "c"."id"

      Now we will get the list of all customers with the number of order equal to zero for customers which have no orders::

          [(Customer[1], 2), (Customer[2], 1), (Customer[3], 1), (Customer[4], 1), (Customer[5], 0)]

      We should mention that in most cases Pony can understand where LEFT JOIN is needed. For example, the same query can be written this way:

      .. code-block:: python

          select((c, count(c.orders)) for c in Customer)[:]

      .. code-block:: sql

          SELECT "c"."id", COUNT(DISTINCT "order-1"."id")
          FROM "Customer" "c"
            LEFT JOIN "Order" "order-1"
              ON "c"."id" = "order-1"."customer"
          GROUP BY "c"."id"


.. py:function:: count(gen)

      Returns the number of objects that match the query condition. Example:

      .. code-block:: python

          count(c for c in Customer if len(c.orders) > 2)

      This query will be translated to the following SQL:

      .. code-block:: sql

          SELECT COUNT(*)
          FROM "Customer" "c"
            LEFT JOIN "Order" "order-1"
              ON "c"."id" = "order-1"."customer"
          GROUP BY "c"."id"
          HAVING COUNT(DISTINCT "order-1"."id") > 2

      The equivalent query can be generated using the :py:meth:`Query.count` method::

          select(c for c in Customer if len(c.orders) > 2).count()


.. py:function:: min(gen)

   Returns the minimum value from the database. The query should return a single attribute::

       min(p.price for p in Product)

   The equivalent query can be generated using the :py:meth:`Query.min` method::

       select(p.price for p in Product).min()


.. py:function:: max(gen)

   Returns the maximum value from the database. The query should return a single attribute::

       max(o.date_shipped for o in Order)

   The equivalent query can be generated using the :py:meth:`Query.max` method::

       select(o.date_shipped for o in Order).max()


.. py:function:: sum(gen)

   Returns the sum of all values selected from the database::

       sum(o.total_price for o in Order)

   The equivalent query can be generated using the :py:meth:`Query.sum` method::

       select(o.total_price for o in Order).sum()

   If the query returns no items, the ``sum()`` method returns 0.


.. py:function:: avg(gen)

   Returns the average value for all selected attributes::

       avg(o.total_price for o in Order)

   The equivalent query can be generated using the :py:meth:`Query.avg` method::

       select(o.total_price for o in Order).avg()


.. py:function:: exists(gen[, globals[, locals])

   Returns ``True`` if at least one instance with the specified condition exists and ``False`` otherwise::

       exists(o for o in Order if o.date_delivered is None)


.. py:function:: distinct(gen)

   When you need to force DISTINCT in a query, it can be done using the distinct() function::

       distinct(o.date_shipped for o in Order)

   But usually this is not necessary, because Pony adds DISTINCT keyword automatically in an intelligent way. See more information about it in the :ref:`Automatic DISTINCT <automatic_distinct>` section later in this chapter.

   Another usage of distinct() function is with the sum() aggregate function - you can write::

       select(sum(distinct(x.val)) for x in X)

   to generate the following SQL:

   .. code-block:: sql

       SELECT SUM(DISTINCT x.val)
       FROM X x

   but it is rarely used in practice.


.. py:function:: desc()
                 desc(attribute)

   This function is used inside :py:meth:`Query.order_by` for ordering in descending order.


.. py:function:: delete(gen)

   Deletes objects from the database.


.. py:function:: raw_sql(sql, result_type=None)

   A function that encapsulates a part of a query expressed in a raw SQL format. When the ``result_type`` is specified, Pony will convert the result of raw SQL fragment to the specified format.

   See examples :ref:`here <using_raw_sql_ref>`.



.. _query_object_ref:

Query object methods
--------------------------

.. class:: Query

   .. py:method:: [start:end]
                  limit(limit, offset=None)

      This method is used for limiting the number of instances selected from the database. In the example below we select the first ten instances::

          select(c for c in Customer).order_by(Customer.name).limit(10)

      Generates the following SQL:

      .. code-block:: sql

          SELECT "c"."id", "c"."email", "c"."password", "c"."name", "c"."country", "c"."address"
          FROM "Customer" "c"
          ORDER BY "c"."name"
          LIMIT 10

      The same effect can be reached using the Python slice operator::

          select(c for c in Customer).order_by(Customer.name)[:10]

      If we need select instances with offset, we should use the second parameter::

          select(c for c in Customer).order_by(Customer.name).limit(10, 20)

      Or using the slice operator::

          select(c for c in Customer).order_by(Customer.name)[20:30]

      It generates the following SQL:

      .. code-block:: sql

          SELECT "c"."id", "c"."email", "c"."password", "c"."name", "c"."country", "c"."address"
          FROM "Customer" "c"
          ORDER BY "c"."name"
          LIMIT 10 OFFSET 20

      Also you can use the :py:meth:`page()<Query.page>` method for the same purpose.

   .. py:method:: avg()

      Returns the average value for all selected attributes::

          select(o.total_price for o in Order).avg()

      The function :py:func:`avg` does a similar thing.


   .. py:method:: count()

      Returns the number of objects that match the query condition::

          select(c for c in Customer if len(c.orders) > 2).count()

      The function :py:func:`count` does a similar thing.


   .. py:method:: distinct()

      Forces DISTINCT in a query::

          select(c.name for c in Customer).distinct()

      But usually this is not necessary, because Pony adds DISTINCT keyword automatically in an intelligent way. See more information about it in the :ref:`Automatic DISTINCT <automatic_distinct>` section later in this chapter.
      The function :py:func:`distinct` does a similar thing.


   .. py:method:: delete(bulk=False)

      Deletes instances selected by a query. When ``bulk=False`` Pony will load each instance into memory and call the ``delete()`` method on each instance (calling ``before_delete`` and ``after_delete`` hooks if they were defined). If ``bulk=True`` Pony doesn't load instances, it just generates the SQL ``DELETE`` statement which deletes objects in the database.

   .. note:: Be careful with the bulk delete:

      - ``before_delete`` and ``after_delete`` hooks will not be called on deleted objects.
      - If an object was loaded into memory, it will not be removed from the db_session cache on bulk delete.


   .. py:method:: exists()

      Returns ``True`` if at least one instance with the specified condition exists and ``False`` otherwise::

          select(c for c in Customer if len(c.cart_items) > 10).exists()

      This query generates the following SQL:

      .. code-block:: sql

          SELECT "c"."id"
          FROM "Customer" "c"
            LEFT JOIN "CartItem" "cartitem-1"
              ON "c"."id" = "cartitem-1"."customer"
          GROUP BY "c"."id"
          HAVING COUNT(DISTINCT "cartitem-1"."id") > 20
          LIMIT 1


   .. py:method:: filter(lambda[, globals[, locals])
                  filter(str)

      The ``filter()`` method of the ``Query`` object is used for filtering the result of a query. The conditions which are passed as parameters to the ``filter()`` method will be translated into the WHERE section of the resulting SQL query.

      Before Pony ORM release 0.5 the ``filter()`` method affected the underlying query updating the query in-place, but since the release 0.5 it creates and returns a new Query object with the applied conditions.

      The number of ``filter()`` arguments should correspond to the query result. The ``filter()`` method can receive a lambda expression with a condition::

          q = select(p for p in Product)
          q2 = q.filter(lambda x: x.price > 100)

          q = select((p.name, p.price) for p in Product)
          q2 = q.filter(lambda n, p: n.name.startswith("A") and p > 100)

      Also the ``filter()`` method can receive a text string where you can specify just the expression::

          q = select(p for p in Product)
          x = 100
          q2 = q.filter("p.price > x")

      Another way to filter the query result is to pass parameters in the form of named arguments::

          q = select(p for p in Product)
          q2 = q.filter(price=100, name="iPod")


   .. py:method:: first()

      Returns the first element from the selected results or ``None`` if no objects were found::

          select(p for p in Product if p.price > 100).first()


   .. py:method:: for_update(nowait=False)

      Sometimes there is a need to lock objects in the database in order to prevent other transactions from modifying the same instances simultaneously. Within the database such lock should be done using the SELECT FOR UPDATE query. In order to generate such a lock using Pony you can call the ``for_update`` method::

          select(p for p in Product if p.picture is None).for_update()[:]

      This query selects all instances of Product without a picture and locks the corresponding rows in the database. The lock will be released upon commit or rollback of current transaction.


   .. py:method:: get()

      Extracts one entity instance from the database. The function returns the object if an object with the specified parameters exists, or ``None`` if there is no such object. If there are more than one objects with the specified parameters, raises the ``MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`` exception. Example::

          select(o for o in Order if o.id == 123).get()

      The function :py:func:`get` does a similar thing.


   .. py::method:: get_sql()

      Returns SQL statement as a string::

          sql = select(c for c in Category if c.name.startswith('a')).get_sql()
          print sql

          SELECT "c"."id", "c"."name"
          FROM "category" "c"
          WHERE "c"."name" LIKE 'a%%'


   .. py:method:: max()

      Returns the maximum value from the database. The query should return a single attribute::

          select(o.date_shipped for o in Order).max()

      The function :py:func:`max` does a similar thing.


   .. py:method:: min()

      Returns the minimum value from the database. The query should return a single attribute::

          select(p.price for p in Product).min()

      The function :py:func:`min` does a similar thing.


   .. py:method:: order_by(attr1 [, attr2, ...])
                  order_by(pos1 [, pos2, ...])
                  order_by(lambda[, globals[, locals])
                  order_by(str)

      This method is used for ordering the results of a query. There are several options options available:

      * Using entity attributes

      .. code-block:: python

          select(o for o in Order).order_by(Order.customer, Order.date_created)

      For ordering in descending order, use the function ``desc()`` or the method ``desc()`` of any attribute::

          select(o for o in Order).order_by(Order.date_created.desc())

      Which is similar to::

          select(o for o in Order).order_by(desc(Order.date_created))

      * Using position of query result variables

      .. code-block:: python

          select((o.customer.name, o.total_price) for o in Order).order_by(-2, 1)

      Minus means sorting in descending order. In this example we sort the result by the total price in descending order and by the customer name in ascending order.

      * Using lambda

      .. code-block:: python

          select(o for o in Order).order_by(lambda o: (o.customer.name, o.date_shipped))

      If the lambda has a parameter (``o`` in our example) then ``o`` represents the result of the ``select`` and will be applied to it. If you specify the lambda without a parameter, then inside lambda you have access to all names defined inside the query::

          select(o.total_price for o in Order).order_by(lambda: o.customer.id)

      * Using a string

      This approach is similar to the previous one, but you specify the body of a lambda as a string:

      .. code-block:: python

          select(o for o in Order).order_by("o.customer.name, o.date_shipped")

   .. py:method:: page(pagenum, pagesize=10)

      Pagination is used when you need to display results of a query divided into multiple pages. The page numbering starts with page 1. This method returns a slice [start:stop] where ``start = (pagenum - 1) * pagesize``, ``stop = pagenum * pagesize``.

   .. py:method:: prefetch

      Allows you to specify which related objects or attributes should be loaded from the database along with the query result.

      Usually there is no need to prefetch related objects. When you work with the query result within the ``@db_session``, Pony gets all related objects once you need them. Pony uses the most effective way for loading related objects from the database, avoiding the N+1 Query problem.

      So, if you use Flask, the recommended approach is to use the ``@db_session`` decorator at the top level, at the same place where you put the Flask's ``app.route`` decorator:

      .. code-block:: python

          @app.route('/index')
          @db_session
          def index():
              ...
              objects = select(...)
              ...
              return render_template('template.html', objects=objects)

      Or, even better, wrapping the wsgi application with the ``db_session`` decorator:

      .. code-block:: python

          app.wsgi_app = db_session(app.wsgi_app)

      If for some reason you need to pass the selected instances along with related objects outside of the ``db_session``, then you can use this method. Otherwise, if you'll try to access the related objects outside of the ``db_session``, you might get the ``DatabaseSessionIsOver`` exception, e.g.::

          DatabaseSessionIsOver: Cannot load attribute Customer[3].name: the database session is over

      More information regarding working with the ``db_session`` can be found :ref:`here <db_session_ref>`.

      You can specify entities and/or attributes as parameters. When you specify an entity, then all "to-one" and non-lazy attributes of corresponding related objects will be prefetched. The "to-many" attributes of an entity are prefetched only when specified explicitly.

      If you specify an attribute, then only this specific attribute will be prefetched. You can specify attribute chains, e.g. `order.customer.address`. The prefetching works recursively - it applies the specified parameters to each selected object.

      Examples::

          from pony.orm.examples.presentation import *

      Loading Student objects only, without prefetching::

          students = select(s for s in Student)[:]

      Loading students along with groups and departments::

          students = select(s for s in Student).prefetch(Group, Department)[:]

          for s in students: # no additional query to the DB will be sent
              print s.name, s.group.major, s.group.dept.name

      The same as above, but specifying attributes instead of entities::

          students = select(s for s in Student).prefetch(Student.group, Group.dept)[:]

          for s in students: # no additional query to the DB will be sent
              print s.name, s.group.major, s.group.dept.name

      Loading students and related courses ("many-to-many" relationship)::

          students = select(s for s in Student).prefetch(Student.courses)

          for s in students:
              print s.name
              for c in s.courses: # no additional query to the DB will be sent
                  print c.name


   .. py:method:: random(limit)

      Selects ``limit`` random objects from the database. This method will be translated using the ``ORDER BY RANDOM()`` SQL expression. The entity class method ``select_random()`` provides better performance, although doesn't allow to specify query conditions.


   .. py:method:: show()

      Prints the results of a query to the console. The result is formatted in the form of a table. This method doesn't display "to-many" attributes because it would require additional query to the database and could be bulky.


   .. py:method:: sum()

      Return the sum of all selected items. Can be applied to the queries which return a single numeric expression only. Example::

          select(o.total_price for o in Order).sum()

      If the query returns no items, the query result will be 0.

   .. py:method:: without_distinct()

      By default Pony tries to avoid duplicates in the query result and intellectually adds the ``DISTINCT`` SQL keyword to a query where it thinks it necessary. If you don't want Pony to add ``DISTINCT`` and get possible duplicates, you can use this method. This method returns a new instance of the Query object, so you can chain it with other query methods::

          select(p.name for p in Person).without_distinct().order_by(Person.name)

      Before Pony Release 0.6 the method ``without_distinct()`` returned query result and not a new query instance.


.. _automatic_distinct:

Automatic DISTINCT
----------------------------------

Pony tries to avoid duplicates in a query result by automatically adding the ``DISTINCT`` SQL keyword where it is necessary, because useful queries with duplicates are very rare. When someone wants to retrieve objects with a specific criteria, they typically don't expect that the same object will be returned more than once. Also, avoiding duplicates makes the query result more predictable: you don't need to filter duplicates out of a query result.

Pony adds the ``DISCTINCT`` keyword only when there could be potential duplicates. Let's consider a couple of examples.

1) Retrieving objects with a criteria:

.. code-block:: python

    select(p for p in Person if p.age > 20 and p.name == "John")

In this example, the query doesn't return duplicates, because the result contains the primary key column of a Person. Since duplicates are not possible here, there is no need in the ``DISTINCT`` keyword, and Pony doesn't add it:

.. code-block:: sql

    SELECT "p"."id", "p"."name", "p"."age"
    FROM "Person" "p"
    WHERE "p"."age" > 20
      AND "p"."name" = 'John'


2) Retrieving object attributes:

.. code-block:: python

    select(p.name for p in Person)

The result of this query returns not objects, but its attribute. This query result can contain duplicates, so Pony will add DISTINCT to this query:

.. code-block:: sql

    SELECT DISTINCT "p"."name"
    FROM "Person" "p"

The result of a such query typically used for a dropdown list, where duplicates are not expected. It is not easy to come up with a real use-case when you want to have duplicates here.

If you need to count persons with the same name, you'd better use an aggregate query:

.. code-block:: python

    select((p.name, count(p)) for p in Person)

But if it is absolutely necessary to get all person's names, including duplicates, you can do so by using the :py:meth:`Query.without_distinct()` method:

.. code-block:: python

    select(p.name for p in Person).without_distinct()

3) Retrieving objects using joins:

.. code-block:: python

    select(p for p in Person for c in p.cars if c.make in ("Toyota", "Honda"))

This query can contain duplicates, so Pony eliminates them using ``DISTINCT``:

.. code-block:: sql

    SELECT DISTINCT "p"."id", "p"."name", "p"."age"
    FROM "Person" "p", "Car" "c"
    WHERE "c"."make" IN ('Toyota', 'Honda')
      AND "p"."id" = "c"."owner"

Without using DISTINCT the duplicates are possible, because the query uses two tables (Person and Car), but only one table is used in the SELECT section. The query above returns only persons (and not their cars), and therefore it is typically not desirable to get the same person in the result more than once. We believe that without duplicates the result looks more intuitive.

But if for some reason you don't need to exclude duplicates, you always can add ``.without_disctinct()`` to the query:

.. code-block:: python

    select(p for p in Person for c in p.cars
             if c.make in ("Toyota", "Honda")).without_distinct()

The user probably would like to see the Person objects duplicates if the query result contains cars owned by each person. In this case the Pony query would be different:

.. code-block:: python

    select((p, c) for p in Person for c in p.cars if c.make in ("Toyota", "Honda"))

And in this case Pony will not add the ``DISTINCT`` keyword to SQL query.


To summarize:

1) The principle "all queries do not return duplicates by default" is easy to understand and doesn't lead to surprises.
2) Such behavior is what most users want in most cases.
3) Pony doesn't add DISTINCT when a query is not supposed to have duplicates.
4) The query method ``without_distinct()`` can be used for forcing Pony do not eliminate duplicates.



Functions which can be used inside Query
----------------------------------------------------------------

Here is the list of functions that can be used inside a generator query:

min, max, avg, sum, count, len, concat, abs, random, select, exists



Examples:

.. code-block:: python

    select(avg(c.orders.total_price) for c in Customer)[:]

.. code-block:: sql

    SELECT AVG("order-1"."total_price")
    FROM "Customer" "c"
      LEFT JOIN "Order" "order-1"
        ON "c"."id" = "order-1"."customer"

.. code-block:: python

    select(o for o in Order if o.customer in
           select(c for c in Customer if c.name.startswith('A')))[:]

.. code-block:: sql

    SELECT "o"."id", "o"."state", "o"."date_created", "o"."date_shipped",
           "o"."date_delivered", "o"."total_price", "o"."customer"
    FROM "Order" "o"
    WHERE "o"."customer" IN (
        SELECT "c"."id"
        FROM "Customer" "c"
        WHERE "c"."name" LIKE 'A%'
        )


.. _using_raw_sql_ref:

Using raw SQL
-------------

Pony allows using raw SQL in your queries. There are two options on how you can use raw SQL:

1. Use the :py:func:`raw_sql` function in order to write only a part of a query using raw SQL.
2. Write a complete SQL query using the :py:meth:`Entity.select_by_sql` or :py:meth:`Entity.get_by_sql` methods.


Using the raw_sql() function
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Let's explore examples of using the ``raw_sql()`` function. Here is the schema and initial data that we'll use for our examples:

.. code-block:: python

    from datetime import date
    from pony.orm import *

    db = Database('sqlite', ':memory:')

    class Person(db.Entity):
        id = PrimaryKey(int)
        name = Required(str)
        age = Required(int)
        dob = Required(date)

    db.generate_mapping(create_tables=True)

    with db_session:
        Person(id=1, name='John', age=30, dob=date(1986, 1, 1))
        Person(id=2, name='Mike', age=32, dob=date(1984, 5, 20))
        Person(id=3, name='Mary', age=20, dob=date(1996, 2, 15))


``raw_sql()`` result can be treated as a logical expression:

.. code-block:: python

    select(p for p in Person if raw_sql('abs("p"."age") > 25'))


``raw_sql()`` result can be used for comparison:

.. code-block:: python

    q = Person.select(lambda x: raw_sql('abs("x"."age")') > 25)
    print(q.get_sql())

    SELECT "x"."id", "x"."name", "x"."age", "x"."dob"
    FROM "Person" "x"
    WHERE abs("x"."age") > 25

Also, in the example above we use ``raw_sql()`` in a lambda query and print out the resulting SQL. As you can see the raw SQL part becomes a part of the whole query.

``raw_sql()`` can accept $parameters:

.. code-block:: python

    x = 25
    select(p for p in Person if raw_sql('abs("p"."age") > $x'))


You can change the content of the ``raw_sql()`` function dynamically and still use parameters inside:

.. code-block:: python

    x = 1
    s = 'p.id > $x'
    select(p for p in Person if raw_sql(s))


Another way of using dynamic raw SQL content:

.. code-block:: python

    x = 1
    cond = raw_sql('p.id > $x')
    select(p for p in Person if cond)


You can use various types inside the raw SQL query:

.. code-block:: python

    x = date(1990, 1, 1)
    select(p for p in Person if raw_sql('p.dob < $x'))


Parameters inside the raw SQL part can be combined:

.. code-block:: python

    x = 10
    y = 15
    select(p for p in Person if raw_sql('p.age > $(x + y)'))


You can even call Python functions inside:

.. code-block:: python

    select(p for p in Person if raw_sql('p.dob < $date.today()'))


``raw_sql()`` can be used not only in the condition part, but also in the part which returns the result of the query:

.. code-block:: python

    names = select(raw_sql('UPPER(p.name)') for p in Person)[:]
    print(names)

    ['JOHN', 'MIKE', 'MARY']


But when you return data using the ``raw_sql()`` function, you might need to specify the type of the result, because Pony has no idea on what the result type is:

.. code-block:: python

    dates = select(raw_sql('(p.dob)') for p in Person)[:]
    print(dates)

    ['1985-01-01', '1983-05-20', '1995-02-15']


If you want to get the result as a list of dates, you need to specify the ``result_type``:

.. code-block:: python

    dates = select(raw_sql('(p.dob)', result_type=date) for p in Person)[:]
    print(dates)

    [datetime.date(1986, 1, 1), datetime.date(1984, 5, 20), datetime.date(1996, 2, 15)]


``raw_sql()`` can be used in a ``filter`` too:

.. code-block:: python

    x = 25
    select(p for p in Person).filter(lambda p: p.age > raw_sql('$x'))


It can be used inside the ``filter`` without lambda. In this case you have to use the first letter of entity name in lower case as the alias:

.. code-block:: python

    x = 25
    Person.select().filter(raw_sql('p.age > $x'))


You can use several ``raw_sql()`` expressions in a single query:

.. code-block:: python

    x = '123'
    y = 'John'
    Person.select(lambda p: raw_sql("UPPER(p.name) || $x")
                            == raw_sql("UPPER($y || '123')"))


The same parameter names can be used several times with different types and values:

.. code-block:: python

    x = 10
    y = 31
    q = select(p for p in Person if p.age > x and p.age < raw_sql('$y'))
    x = date(1980, 1, 1)
    y = 'j'
    q = q.filter(lambda p: p.dob > x and p.name.startswith(raw_sql('UPPER($y)')))
    persons = q[:]


You can use ``raw_sql()`` in ``order_by`` section:

.. code-block:: python

    x = 9
    Person.select().order_by(lambda p: raw_sql('SUBSTR(p.dob, $x)'))


Or without lambda, if you use the same alias, that you used in previous filters. In this case we use the default alias - the first letter of the entity name:

.. code-block:: python

    x = 9
    Person.select().order_by(raw_sql('SUBSTR(p.dob, $x)'))


.. _entities_raw_sql_ref:

Using the select_by_sql() and get_by_sql() methods
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Although Pony can translate almost any condition written in Python to SQL, sometimes the need arises to use raw SQL, for example - in order to call a stored procedure or to use a dialect feature of a specific database system. In this case, Pony allows the user to write a query in a raw SQL, by placing it inside the function :py:meth:`Entity.select_by_sql` or :py:meth:`Entity.get_by_sql` as a string:

.. code-block:: python

    Product.select_by_sql("SELECT * FROM Products")

Unlike the method :py:meth:`Entity.select`, the method :py:meth:`Entity.select_by_sql` does not return the :py:class:`Query` object, but a list of entity instances.

Parameters are passed using the following syntax: "$name_variable" or "$(expression in Python)". For example:

.. code-block:: python

    x = 1000
    y = 500
    Product.select_by_sql("SELECT * FROM Product WHERE price > $x OR price = $(y * 2)")

When Pony encounters a parameter within a raw SQL query, it gets the variable value from the current frame (from globals and locals) or from the dictionaries which can be passed as parameters:

.. code-block:: python

    Product.select_by_sql("SELECT * FROM Product WHERE price > $x OR price = $(y * 2)",
                           globals={'x': 100}, locals={'y': 200})

Variables and more complex expressions specified after the ``$`` sign, will be automatically calculated and transferred into the query as parameters, which makes SQL-injection impossible. Pony automatically replaces $x in the query string with "?", "%S" or with other paramstyle, used in your database.

If you need to use the ``$`` sign in the query (for example, in the name of a system table), you have to write two ``$`` signs in succession: ``$$``.
