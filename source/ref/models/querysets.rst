========================
``QuerySet`` API 参考
========================

.. currentmodule:: django.db.models.query

本篇文档包含 ``QuerySet`` API的详细信息. 它建立在 :doc:`模型 </topics/db/models>` 和
:doc:`数据库查询 </topics/db/queries>` 指南之上, 所以在阅读本文档之前, 你需要先阅读和理解这两部分的文档.

本文档将通篇使用在 :doc:`数据库查询指南
</topics/db/queries>` 中用到的 :ref:`WebBlog模型例子
<queryset-model-example>`.

.. _when-querysets-are-evaluated:

``QuerySet`` 何时求值
======================

实际上, 当一个 ``QuerySet`` 被创建, 过滤, 切片和传递时并不会实际操作数据库.
在对查询集做求值之前, 不会产生任何实际的数据库操作.

``QuerySet`` 求值有以下几种方式:

* **迭代.** ``QuerySet`` 是可迭代的, 当它被首次迭代是会执行数据库查询. 例如,
  下面的语句会将数据库中所有Entry的headline打印出来::

      for e in Entry.objects.all():
          print(e.headline)

  主要: 不要使用上面语句来验证数据库中是否存在某条记录, 使用 :meth:`~QuerySet.exists` 方法更高效.

* **切片.** 正如 :ref:`limiting-querysets` 中描述一样, 可以使用Python的序列切片语法对 ``QuerySet`` 进行切片操作,
  对一个未求值的 ``QuerySet`` 进行切片操作会返回另一个未求值的 ``QuerySet``,
  但是如果使用了 "step" 参数, Django将执行数据库查询, 然后返回查询结果的列表.
  对已经求值过的 ``QuerySet`` 切片也是返回一个列表.

  需要注意的是, 对未求值的 ``QuerySet`` 切片返回的 ``QuerySet`` 不可以再进行修改操作(e.g.,
  新增过滤器, 或者修改排序). 因为这样不难再转化成SQL而且这样的需求没有实际意义.

* **Pickling/缓存.** 有关 `pickling QuerySets`_ 的细节, 请参阅 `pickling QuerySets`_ 部份.
  这里提到它的目的是强调序列化时会读取数据库.

* **repr().** 当对 ``QuerySet`` 调用 ``repr()`` 方法时会对其求值.
  这是为了在Python交互式解释器中使用方便, 这样就可以在交互式解释器中使用这个API立即看到结果.

* **len().** 当对 ``QuerySet`` 调用 ``len()`` 方法时会对其求值.
  正如猜想那样，会返回查询集的长度.

  注意: 如果只是想确认集合中的记录条数(而并不需要实际对象), 使用SQL的 ``SELECT COUNT(*)`` 来处理数据库级别的计数更有效.
  Django为此提供了 :meth:`~QuerySet.count` 方法.

* **list().** 对 ``QuerySet`` 调用 ``list()`` 可以对其进行强制求值, 例如::

      entry_list = list(Entry.objects.all())

* **bool().** 试探 ``QuerySet`` 的布尔值, 例如使用 ``bool()``, ``or``, ``and`` 或者 ``if`` 判断,
  都触发求值操作. 如果结果至少包含一条记录, 则 ``QuerySet`` 为 ``True``, 否则为 ``False``. 例如::

      if Entry.objects.filter(headline="Test"):
         print("Entry 中至少有一条记录的headline为Test")

  注意: 如果只是想确认结果中是否至少存在一条记录(并且不需要实际对象), 使用 :meth:`~QuerySet.exists` 方法更加高效.

.. _pickling QuerySets:

Pickling ``QuerySet``
-----------------------

如果对 ``QuerySet`` 进行 :mod:`pickle` 操作, 它将在Pickle之前强制将所有的结果加载到内存中.
Pickling 通常用缓存之前, 当下次重新加载缓存的查询集时, 其结果已经就是能够直接使用的了(免去了再次从数据库读取的耗时).
也就是说当unpickle ``QuerySet`` 时, 就是从数据库中查询的结果.

如果只是想序列化部分必要的信息, 以便后面可以从数据库中重建 ``Queryset``, 那只序列化 ``QuerySet`` 的 ``query`` 属性即可.
接下来就可以使用下面的代码重建原来的 ``QuerySet`` (这个过程没有数据库读取)::

    >>> import pickle
    >>> query = pickle.loads(s)     # Assuming 's' is the pickled string.
    >>> qs = MyModel.objects.all()
    >>> qs.query = query            # Restore the original 'query'.

``query`` 属性是一个不透明的对象. 它表示查询的内部结构, 不属于公开的API. 即便如此, 对于本节提到的序列化和反序列化来说, 它仍是安全和被完全支持的.

.. admonition:: 不同版本间不能共享Pickle结果

    ``QuerySets`` 的Pickle只能用于生成它们的Django版本中. 如果使用Django的版本N生成一个Pickle,
    不保证这个Pickle在Django 的版本N+1中可以读取. Pickle不可用于归档的长期策略.

    因为Pickle兼容性的错误很难诊断例如产生损坏的对象， 当试图Unpickle的查询集与Pickle时的Django 版本不同时，将引发一个 ``RuntimeWarning``.

.. _queryset-api:

``QuerySet`` API
================

下面是对 ``QuerySet`` 的正式定义:

.. class:: QuerySet(model=None, query=None, using=None)

    通常使用 ``QuerySet`` 时会以 :ref:`链式过滤 <chaining-filters>` 来使用. 因此大部分
    ``QuerySet`` 方法返回的是一个新的查询集. 本节将会详细介绍这些方法.

    ``QuerySet`` 类具有两个可用于自省的公共属性:

    .. attribute:: ordered

        如果 ``QuerySet`` 是有序的则为 ``True``  — 例如
        :meth:`order_by()` 子句或者模型默认的排序. 否则为 ``False`` .

    .. attribute:: db

        如果执行查询, 将使用该数据库.

    .. note::

        :class:`QuerySet` 存在 ``query`` 参数是为了让具有特殊查询用途的子类如
        :class:`~django.contrib.gis.db.models.GeoQuerySet` 可以重新构造内部查询状态.
        这个参数的值是查询状态的不透明表示， 不是一个公开的API.
        简而言之：如果你有疑问，其实你实际上不需要使用它.

.. currentmodule:: django.db.models.query.QuerySet

返回新 ``QuerySet`` 的方法
----------------------------

Django提供了一系列的 ``QuerySet`` 筛选方法，用于修改 ``QuerySet`` 返回的结果类型或者SQL的查询方式.

``filter()``
~~~~~~~~~~~~

.. method:: filter(**kwargs)

返回一个新的包含满足查询参数的 ``QuerySet`` 对象.

查询参数(``**kwargs``) 必须满足下文 `Field lookups`_ 的格式.

如果需要更复杂的查询 (例如 ``OR`` 语句), 可以使用 :class:`Q查询 <django.db.models.Q>`.

``exclude()``
~~~~~~~~~~~~~

.. method:: exclude(**kwargs)

返回一个新的不包含满足查询参数的 ``QuerySet`` 对象.

查询参数(``**kwargs``) 必须满足下文 `Field lookups`_ 的格式. 在底层SQL语句中, 多个参数通过 ``AND`` 连接.
然后所查的内容都会被放入 ``NOT()`` 句子中.

下面的示例排除所有 ``pub_date`` 大于2005-1-3 且 ``headline`` 为“Hello”的记录::

    Entry.objects.exclude(pub_date__gt=datetime.date(2005, 1, 3), headline='Hello')

用SQL语句表示, 它等同于::

    SELECT ...
    WHERE NOT (pub_date > '2005-1-3' AND headline = 'Hello')

下面示例排序所有 whose ``pub_date`` 大于 2005-1-3 或者
headline 为 "Hello"的记录::

    Entry.objects.exclude(pub_date__gt=datetime.date(2005, 1, 3)).exclude(headline='Hello')

用SQL语句表示, 它等同于::

    SELECT ...
    WHERE NOT pub_date > '2005-1-3'
    AND NOT headline = 'Hello'

第二个例子过滤更严格.

如果需要更复杂的查询 (例如 ``OR`` 语句), 可以使用 :class:`Q查询 <django.db.models.Q>`.

``annotate()``
~~~~~~~~~~~~~~

.. method:: annotate(*args, **kwargs)

使用 :doc:`查询表达式 </ref/models/expressions>` 注解 ``QuerySet`` 中的每个对象.
表达式可以是简单的值、模型或关联模型的字段引用,或者是对与 ``QuerySet`` 中对象相关的对象进行计算的聚合表达式(平均值、总和等).

``annotate()`` 的每个参数都是一个注解，将添加到返回的 ``QuerySet`` 中的每个对象中.

Django提供的聚合函数在下文的 `Aggregation Functions`_ 文档中有详细介绍.

关键字参数指定的注解将使用关键字作为注解的别名. 匿名参数的别名将基于聚合函数的名称和模型的字段生成.
只有引用单个字段的聚合表达式才可以使用匿名参数. 其它所有形式都必须用关键字参数.

例如，如果操作一个Blog列表，如果想知道每个Blog有多少Entry::

    >>> from django.db.models import Count
    >>> q = Blog.objects.annotate(Count('entry'))
    # The name of the first blog
    >>> q[0].name
    'Blogasaurus'
    # The number of entries on the first blog
    >>> q[0].entry__count
    42

``Blog`` 模型本身并没有定义 ``entry__count`` 属性, 但是如果使用关键字参数来指定聚合函数. 就生成了相应注解名称的属性::

    >>> q = Blog.objects.annotate(number_of_entries=Count('entry'))
    # The number of entries on the first blog, using the name provided
    >>> q[0].number_of_entries
    42

有关聚合的深入讨论,参考 :doc:`聚合主题指南 </topics/db/aggregation>`.

``order_by()``
~~~~~~~~~~~~~~

.. method:: order_by(*fields)

默认情况下， ``QuerySet`` 返回的结果是根据模型 ``Meta`` 中的 ``ordering`` 选项给出的排序元组排序.
也可以使用 ``order_by`` 方法给每个 ``QuerySet`` 指定特定的排序.

例如::

    Entry.objects.filter(pub_date__year=2005).order_by('-pub_date', 'headline')

上面结果将根据 ``pub_date`` 降序, 按 ``headline`` 升序. ``"-pub_date"`` 前面的负号表示
*降序* 排列. 隐式形式是升序排列, 使用 ``"?"`` 表示随机排序, 例如::

    Entry.objects.order_by('?')

注意: ``order_by('?')`` 查询可能会耗费资源且很慢, 这也取决于使用的数据库.

若要按照另外一个模型中的字段排序, 可以使用查询关联模型时的语法. 即通过字段的名称后面跟上两个下划线(``__``),
再跟上新模型中的字段的名称,像这样::

    Entry.objects.order_by('blog__name', 'headline')

如果根据关联模型字段排序, Django将使用关联的模型的默认排序, 或者如果没有指定 :attr:`Meta.ordering
<django.db.models.Options.ordering>`  将通过关联的模型的主键排序. 例如, 因为 ``Blog`` 模型没有指定默认的排序::

    Entry.objects.order_by('blog')

...其等价于::

    Entry.objects.order_by('blog__id')

如果 ``Blog`` 设置了 ``ordering = ['name']``, 那么第一个查询等价于::

    Entry.objects.order_by('blog__name')

通过引用相关字段的 ``_id`` , 同样可以通过相关字段来排序查询集，而不会导致JOIN开销::

    # No Join
    Entry.objects.order_by('blog_id')

    # Join
    Entry.objects.order_by('blog__id')

你也可以通过 :doc:`查询表达式 </ref/models/expressions>` 调用
 ``asc()`` 或者 ``desc()`` 排序::

    Entry.objects.order_by(Coalesce('summary', 'headline').desc())

当使用关联模型排序还使用到了 :meth:`distinct()` 时需要注意,
:meth:`distinct` 中有说明关联模型的排序如何会对预期结果产生影响.

.. note::
    指定一个多值字段来排序结果(例如, 一个 :class:`~django.db.models.ManyToManyField` 字段,
    或者 :class:`~django.db.models.ForeignKey` 的反向关联字段)

    考虑下面这种情况::

         class Event(Model):
            parent = models.ForeignKey(
                'self',
                on_delete=models.CASCADE,
                related_name='children',
            )
            date = models.DateField()

         Event.objects.order_by('children__date')

    在这里，每个 ``Event`` 可能有多个排序数据；具有多个 ``children`` 的每个 ``Event`` 将被多次返回到 ``order_by()``
    创建的新的 ``QuerySet`` 中. 换句话说, 用 ``order_by()`` 方法对 ``QuerySet`` 对象进行操作会返回一个扩大版的新
    ``QuerySet`` 对象——新增的条目也许并没有什么用，你也用不着它们.

    因此，当使用多值字段对结果进行排序时要格外小心. **如果** 可以确保每个排序项只有一个排序数据,
    这种方法不会出现问题. 如果不确定，请确保结果是你期望的.


是没有方法指定排序是否对大小写敏感. 对于大小写的敏感性, Django将根据数据库中的排序方式给出排序结果.

你可以通过 :class:`~django.db.models.functions.Lower` 将字段转换为小写来排序, 这样就能达到大小写一致的排序::

    Entry.objects.order_by(Lower('headline').desc())

如果你不需要查询做任何排序,默认排序也不需要! 可以调用不带参数的 :meth:`order_by()` .

可以通过检查 :attr:`.QuerySet.ordered` 来判断查询结果是否有序. 不论 ``QuerySet`` 以任何方式排序，它将是 ``True``.

每个 ``order_by()`` 都会清除它之前的所有排序. 例如, 下面查询将会按照
``pub_date`` 排序而不是 ``headline``::

    Entry.objects.order_by('headline').order_by('pub_date')

.. warning::

    排序不是没有开销的操作. 添加到排序中的每个字段都将带来数据库的开销. 添加的每个外键也都将隐式包含进它的默认排序.

    如果查询没有指定顺序，则会以未指定的顺序从数据库返回结果. 仅当通过唯一标识结果中的每个对象的一组字段排序时，
    才能保证特定的排序。 例如，如果 ``name`` 字段不唯一，由其排序则不会保证具有相同名称的对象总是以相同的顺序显示.

``reverse()``
~~~~~~~~~~~~~

.. method:: reverse()

 ``reverse()`` 方法用于反向排序QuerySet中的元素. 再次调用 ``reverse()`` 将恢复原有排序.

比如要获取QuerySet中的最后五个元素,可以这样::

    my_queryset.reverse()[:5]

注意, 这和Python中的在列表末尾切片不一样. 上面例子将先返回最后一个元素,然后是倒数第二个,依次类推.
如果在Python序列中调用 ``seq[-5:]``, 我们将先看到返回的倒数第五个元素. Django 并不支持这种模式访问(从末尾切片),
因此这不好在SQL中高效实现.

同时, ``reverse()`` 也只能在定义了ordering的 ``QuerySet`` 上调用
(e.g., 一个定义了默认排序的模型，或者使用了 :meth:`order_by()` 方法).
如果给定的 ``QuerySet`` 没有定义这样的ordering，那么调用 ``reverse()`` 就没有实际效果
(``reverse()`` 之前没有定义ordering, 之后也将保持未定义).

``distinct()``
~~~~~~~~~~~~~~

.. method:: distinct(*fields)

返回一个在SQL查询中使用 ``SELECT DISTINCT`` 句子的 ``QuerySet``.  它将去除查询结果中重复的行.

默认情况下, ``QuerySet`` 不会进行去重操作. 而在实际情况中, 这一般不会有问题, 因为像 ``Blog.objects.all()`` 这样简答的查询不会引入重复的行.
但是, 在跨多表查询时， ``QuerySet`` 可能就会包含重复的结果. 这时候就应该使用 ``distinct()``.

.. note ::
    :meth:`order_by` 调用中使用的任何字段都包含在SQL的 ``SELECT`` 列当中. 当和 ``distinct()`` 一起使用时，可能会导致意料之外的结果.
    如果根据关联模型的字段排序，那么这个字段将被添加到查询字段中，这样它们可能会使其他本来是重复的行看起来不同了.
    而由于这个额外的字段不会出现在返回的结果中(它们只用于排序)，所以这时看起来返回的结果并不正确.

    类似地，如果使用 :meth:`values()` 查询来限制所选的列，那么 :meth:`order_by()` (或默认的模型排序)中使用的列仍然会涉及，并可能影响结果的唯一性。

    上面的意思是，如果您使用的是 ``distinct()`` ，那么使用相关模型字段排序时一定得小心。同样，
    当将 ``distinct()`` 和 :meth:`values()` 一起使用时，请注意字段在不在 :meth:`values()` 中。

仅在PostgreSQL上, 可以传递位置参数(``*fields``), 用来指定 ``DISTINCT`` 应该应用到的字段的名称.
转换为SQL查询上的 ``SELECT DISTINCT ON``. 区别于其他的普通 ``distinct()`` 调用,
数据库在确定哪些行是不同的时候比较每一行中的每个字段. 对于具有指定字段名的 ``distinct()`` 调用, 数据库将只比较指定字段名.

.. note::

    当指定字段名时, *必须* 在 ``QuerySet`` 中使用 ``order_by()``, ``order_by()`` 中的字段必须和 ``distinct()`` 字段顺序相同.

    例如, ``SELECT DISTINCT ON (a)`` 为每个列 ``a`` 中提供第一行, 如果没有指定顺序就会返回随机的一行.


示例 (除第一个例子,其他仅在PostgreSQL上有效)::

    >>> Author.objects.distinct()
    [...]

    >>> Entry.objects.order_by('pub_date').distinct('pub_date')
    [...]

    >>> Entry.objects.order_by('blog').distinct('blog')
    [...]

    >>> Entry.objects.order_by('author', 'pub_date').distinct('author', 'pub_date')
    [...]

    >>> Entry.objects.order_by('blog__name', 'mod_date').distinct('blog__name', 'mod_date')
    [...]

    >>> Entry.objects.order_by('author', 'pub_date').distinct('author')
    [...]

.. note::
    注意, :meth:`order_by` 使用在定义了默认排序的关联模型中时，可能需要使用关联 ``_id`` 或者关联字段显式排序,
    以 ``DISTINCT ON`` 表达式与 ``ORDER BY`` 子句开头的表达式匹配. 例如，如果 ``Blog`` 模型按定义了一个
    按 ``name`` 的 :attr:`~django.db.models.Options.ordering`::

        Entry.objects.order_by('blog').distinct('blog')

    ...将不会生效, 因为查询时将会使用 ``blog_name`` 排序, 这与 ``DISTINCT ON`` 表达式不匹配. 这种情况下必须使用关联 `_id` 字段
     (该例中为 ``blog_id`` ) 或者引用的字段(``blog__pk``) 显式排序, 保证两个表达式匹配.

``values()``
~~~~~~~~~~~~

.. method:: values(*fields)

返回一个 ``QuerySet`` ，该 ``QuerySet`` 返回字典，而不是可迭代的模型实例.

每个字典都表示一个对象, 其键对应于模型对象的属性名.

这个例子比较了 ``values()`` 字典和普通模型对象::

    # This list contains a Blog object.
    >>> Blog.objects.filter(name__startswith='Beatles')
    <QuerySet [<Blog: Beatles Blog>]>

    # This list contains a dictionary.
    >>> Blog.objects.filter(name__startswith='Beatles').values()
    <QuerySet [{'id': 1, 'name': 'Beatles Blog', 'tagline': 'All the latest Beatles news.'}]>

``values()`` 方法接受可选位置参数 ``*fields``, 其作用是用于指定 ``SELECT`` 中限制的字段名.
如果设置了限制字段, 那个所有字典只会包含指定的 键/值. 如果没有指定字段, 那么所有字段将包含数据库表中所有字段的键值.

例子::

    >>> Blog.objects.values()
    <QuerySet [{'id': 1, 'name': 'Beatles Blog', 'tagline': 'All the latest Beatles news.'}]>
    >>> Blog.objects.values('id', 'name')
    <QuerySet [{'id': 1, 'name': 'Beatles Blog'}]>

值得注意的几点:

* 如果有一个名为 ``foo`` 的 :class:`~django.db.models.ForeignKey` 字段, 那么调用默认的 ``values()`` 返回的字典将
  包含一个 ``foo_id`` 的键, 因为这是存储实际值的隐藏模型属性名称(``foo`` 属性引用相关的模型). 当调用 ``values()`` 并传入字段名时,
  您可以传入 ``foo`` 或 ``foo_id``, 返回相同的内容(字典键会匹配传入的字段名).

  示例::

    >>> Entry.objects.values()
    <QuerySet [{'blog_id': 1, 'headline': 'First Entry', ...}, ...]>

    >>> Entry.objects.values('blog')
    <QuerySet [{'blog': 1}, ...]>

    >>> Entry.objects.values('blog_id')
    <QuerySet [{'blog_id': 1}, ...]>

* 当同时使用 ``values()`` 和 ``distinct()`` 时, 注意排序会影响结果. 详细信息请参阅 :meth:`distinct` 中的注释.

* 如果在 :meth:`extra()` 调用之后使用 ``values()`` 子句, 那么在 :meth:`extra()` 中的 ``select`` 参数定义的任何字段都必须显式包含
  在 ``values()`` 中. 在 ``values()`` 之后进行的任何 :meth:`extra()` 都将忽略其selected的额外字段.

* 在 ``values()`` 之后调用 :meth:`only()` 和 :meth:`defer()` 不太合理, 因此这么做会引发 ``NotImplementedError``.

当只需要少量可用字段的值, 并且不需要模型实例对象的功能时, 只选择需要使用的字段会更有效.

最后, 可以在 ``values()`` 调用之后调用 ``filter()`` 、 ``order_by()`` 等, 这意味着下面这两个调用是相同的::

    Blog.objects.values().order_by('id')
    Blog.objects.order_by('id').values()

Django的开发者喜欢将所有影响sql的方法放在前面(可选), 然后才是影响输出的方法(例如 ``values()`` ),
但是实际上无所谓, 这是卖弄你个性的好机会.

还可以通过 ``OneToOneField``, ``ForeignKey`` 和 ``ManyToManyField`` 属性来引用具有反向关系的相关模型的字段::

    >>> Blog.objects.values('name', 'entry__headline')
    <QuerySet [{'name': 'My blog', 'entry__headline': 'An entry'},
         {'name': 'My blog', 'entry__headline': 'Another entry'}, ...]>

.. warning::

   因为 :class:`~django.db.models.ManyToManyField` 字段和反向关联关系可以有多个关联的行,
   包括这些行可能会使结果集倍数放大.如果在 ``values()`` 查询中包含多个此类字段，这会特别明显，
   在这种情况下，将返回所有可能的组合.

``values_list()``
~~~~~~~~~~~~~~~~~

.. method:: values_list(*fields, flat=False)

它与 ``values()`` 非常类似, 只是在迭代时返回的是元组而不是字典.
每个元组包含传递到 ``values_list()`` 的相应字段的值——因此第一个项是第一个字段, etc. 例如::

    >>> Entry.objects.values_list('id', 'headline')
    [(1, 'First entry'), ...]

如果只传递了一个字段, 可以使用 ``flat`` 参数. 如何设置为 ``True``, 返回的结果将会是单个值而不是元组,
下面的例子更容易理解其作用::

    >>> Entry.objects.values_list('id').order_by('id')
    [(1,), (2,), (3,), ...]

    >>> Entry.objects.values_list('id', flat=True).order_by('id')
    [1, 2, 3, ...]

如果传入多个字段同时设置了 ``flat`` 时将产生错误.

如果没有向``values_list()`` 中传入字段, 那么它将会返回模型中所有字段, 顺序为模型在定义的顺序.

一个常见的需求是获取某个模型实例的特定字段值. 使用 ``values_list()`` 跟上 ``get()`` 调用来实现::

    >>> Entry.objects.values_list('headline', flat=True).get(pk=1)
    'First entry'

这个比喻在处理多对多和其他多值关系（例如反向外键的一对多关系）时分歧，因为“一行一对象”的假设不成立
``values()`` 和 ``values_list()`` 都是用于特定用例的优化: 检索数据子集而不需要创建模型实例.
但是在处理多对多和其他多值关系(比如反向外键的一对多关系)时不适用, 因为“一行，一个对象”的假设都成立.

例子，注意下面通过 :class:`~django.db.models.ManyToManyField` 进行查询时的行为::

    >>> Author.objects.values_list('name', 'entry__headline')
    [('Noam Chomsky', 'Impressions of Gaza'),
     ('George Orwell', 'Why Socialists Do Not Believe in Fun'),
     ('George Orwell', 'In Defence of English Cooking'),
     ('Don Quixote', None)]

具有多个entry的Author会多次出现，而没有任何entry的Author则是 ``None``.

类似地, 当查询反向外键时. 对于没有entry的Author仍然是 ``None`` ::

    >>> Entry.objects.values_list('authors')
    [('Noam Chomsky',), ('George Orwell',), (None,)]

``dates()``
~~~~~~~~~~~

.. method:: dates(field, kind, order='ASC')

返回一个计算结果为 :class:`datetime.date` 列表的 ``QuerySet``. 内容是 ``QuerySet`` 中某一特定类型的所有可用日期.

``field`` 是模型中 ``DateField`` 字段的名称.
``kind`` 接受 ``"year"`` 、 ``"month"`` 或者 ``"day"`` 参数. 结果中每个
``datetime.date`` 对象都会返回按指定的 ``type`` 截断的结果.

* ``"year"`` 返回字段的所有不同年份值的列表.
* ``"month"`` 返回字段的所有不同 year/month 的列表.
* ``"day"`` 返回字段的所有不同 year/month/day 的列表.

``order``, 默认为 ``'ASC'``, 接受 ``'ASC'`` 和 ``'DESC'`` 两种参数. 用于指定排序方式.

Examples::

    >>> Entry.objects.dates('pub_date', 'year')
    [datetime.date(2005, 1, 1)]
    >>> Entry.objects.dates('pub_date', 'month')
    [datetime.date(2005, 2, 1), datetime.date(2005, 3, 1)]
    >>> Entry.objects.dates('pub_date', 'day')
    [datetime.date(2005, 2, 20), datetime.date(2005, 3, 20)]
    >>> Entry.objects.dates('pub_date', 'day', order='DESC')
    [datetime.date(2005, 3, 20), datetime.date(2005, 2, 20)]
    >>> Entry.objects.filter(headline__contains='Lennon').dates('pub_date', 'day')
    [datetime.date(2005, 3, 20)]

``datetimes()``
~~~~~~~~~~~~~~~

.. method:: datetimes(field_name, kind, order='ASC', tzinfo=None)

返回一个计算结果为 :class:`datetime.datetime` 列表的 ``QuerySet``. 内容是 ``QuerySet`` 中某一特定类型的所有可用日期.


``field_name`` 是模型中 ``DateTimeField`` 字段的名称.

``kind`` 接受 ``"year"``, ``"month"``, ``"day"``, ``"hour"``,
``"minute"`` 和 ``"second"`` 参数. 结果中每个
``datetime.datetime`` 对象都会返回按指定的 ``type`` 截断的结果.

``order``, 默认为 ``'ASC'``, 接受 ``'ASC'`` 和 ``'DESC'`` 两种参数. 用于指定排序方式.

``tzinfo`` 定义在截断之前将数据时间转换到的时区. 这取决于使用的时区. 此参数必须是 :class:`datetime.tzinfo` 对象.
如果传入为 ``None``, Django 会使用 :ref:`当前时区 <default-current-time-zone>`. 当 :setting:`USE_TZ` 设置为 ``False`` 时该项无效.

.. _database-time-zone-definitions:

.. note::

    此函数直接在数据库中执行时区转换。因此，您的数据库必须能够解析 ``tzinfo.tzname(None)`` 的值。这意味着以下要求:

    - SQLite: 安装 pytz_ — 转换过程其实是在Python中完成.
    - PostgreSQL: 没有要求 (参考 `Time Zones`_).
    - Oracle: 没有要求 (参考 `Choosing a Time Zone File`_).
    - MySQL: 安装 pytz_ 并且时使用 `mysql_tzinfo_to_sql`_ 加载时区表.

    .. _pytz: http://pytz.sourceforge.net/
    .. _Time Zones: https://www.postgresql.org/docs/current/static/datatype-datetime.html#DATATYPE-TIMEZONES
    .. _Choosing a Time Zone File: https://docs.oracle.com/cd/E11882_01/server.112/e10729/ch4datetime.htm#NLSPG258
    .. _mysql_tzinfo_to_sql: https://dev.mysql.com/doc/refman/en/mysql-tzinfo-to-sql.html

``none()``
~~~~~~~~~~

.. method:: none()

Calling none() will create a queryset that never returns any objects and no
query will be executed when accessing the results. A qs.none() queryset
is an instance of ``EmptyQuerySet``.

Examples::

    >>> Entry.objects.none()
    <QuerySet []>
    >>> from django.db.models.query import EmptyQuerySet
    >>> isinstance(Entry.objects.none(), EmptyQuerySet)
    True

``all()``
~~~~~~~~~

.. method:: all()

Returns a *copy* of the current ``QuerySet`` (or ``QuerySet`` subclass).  This
can be useful in situations where you might want to pass in either a model
manager or a ``QuerySet`` and do further filtering on the result. After calling
``all()`` on either object, you'll definitely have a ``QuerySet`` to work with.

When a ``QuerySet`` is :ref:`evaluated <when-querysets-are-evaluated>`, it
typically caches its results. If the data in the database might have changed
since a ``QuerySet`` was evaluated, you can get updated results for the same
query by calling ``all()`` on a previously evaluated ``QuerySet``.

``select_related()``
~~~~~~~~~~~~~~~~~~~~

.. method:: select_related(*fields)

Returns a ``QuerySet`` that will "follow" foreign-key relationships, selecting
additional related-object data when it executes its query. This is a
performance booster which results in a single more complex query but means
later use of foreign-key relationships won't require database queries.

The following examples illustrate the difference between plain lookups and
``select_related()`` lookups. Here's standard lookup::

    # Hits the database.
    e = Entry.objects.get(id=5)

    # Hits the database again to get the related Blog object.
    b = e.blog

And here's ``select_related`` lookup::

    # Hits the database.
    e = Entry.objects.select_related('blog').get(id=5)

    # Doesn't hit the database, because e.blog has been prepopulated
    # in the previous query.
    b = e.blog

You can use ``select_related()`` with any queryset of objects::

    from django.utils import timezone

    # Find all the blogs with entries scheduled to be published in the future.
    blogs = set()

    for e in Entry.objects.filter(pub_date__gt=timezone.now()).select_related('blog'):
        # Without select_related(), this would make a database query for each
        # loop iteration in order to fetch the related blog for each entry.
        blogs.add(e.blog)

The order of ``filter()`` and ``select_related()`` chaining isn't important.
These querysets are equivalent::

    Entry.objects.filter(pub_date__gt=timezone.now()).select_related('blog')
    Entry.objects.select_related('blog').filter(pub_date__gt=timezone.now())

You can follow foreign keys in a similar way to querying them. If you have the
following models::

    from django.db import models

    class City(models.Model):
        # ...
        pass

    class Person(models.Model):
        # ...
        hometown = models.ForeignKey(
            City,
            on_delete=models.SET_NULL,
            blank=True,
            null=True,
        )

    class Book(models.Model):
        # ...
        author = models.ForeignKey(Person, on_delete=models.CASCADE)

... then a call to ``Book.objects.select_related('author__hometown').get(id=4)``
will cache the related ``Person`` *and* the related ``City``::

    b = Book.objects.select_related('author__hometown').get(id=4)
    p = b.author         # Doesn't hit the database.
    c = p.hometown       # Doesn't hit the database.

    b = Book.objects.get(id=4) # No select_related() in this example.
    p = b.author         # Hits the database.
    c = p.hometown       # Hits the database.

You can refer to any :class:`~django.db.models.ForeignKey` or
:class:`~django.db.models.OneToOneField` relation in the list of fields
passed to ``select_related()``.

You can also refer to the reverse direction of a
:class:`~django.db.models.OneToOneField` in the list of fields passed to
``select_related`` — that is, you can traverse a
:class:`~django.db.models.OneToOneField` back to the object on which the field
is defined. Instead of specifying the field name, use the :attr:`related_name
<django.db.models.ForeignKey.related_name>` for the field on the related object.

There may be some situations where you wish to call ``select_related()`` with a
lot of related objects, or where you don't know all of the relations. In these
cases it is possible to call ``select_related()`` with no arguments. This will
follow all non-null foreign keys it can find - nullable foreign keys must be
specified. This is not recommended in most cases as it is likely to make the
underlying query more complex, and return more data, than is actually needed.

If you need to clear the list of related fields added by past calls of
``select_related`` on a ``QuerySet``, you can pass ``None`` as a parameter::

   >>> without_relations = queryset.select_related(None)

Chaining ``select_related`` calls works in a similar way to other methods -
that is that ``select_related('foo', 'bar')`` is equivalent to
``select_related('foo').select_related('bar')``.

``prefetch_related()``
~~~~~~~~~~~~~~~~~~~~~~

.. method:: prefetch_related(*lookups)

Returns a ``QuerySet`` that will automatically retrieve, in a single batch,
related objects for each of the specified lookups.

This has a similar purpose to ``select_related``, in that both are designed to
stop the deluge of database queries that is caused by accessing related objects,
but the strategy is quite different.

``select_related`` works by creating an SQL join and including the fields of the
related object in the ``SELECT`` statement. For this reason, ``select_related``
gets the related objects in the same database query. However, to avoid the much
larger result set that would result from joining across a 'many' relationship,
``select_related`` is limited to single-valued relationships - foreign key and
one-to-one.

``prefetch_related``, on the other hand, does a separate lookup for each
relationship, and does the 'joining' in Python. This allows it to prefetch
many-to-many and many-to-one objects, which cannot be done using
``select_related``, in addition to the foreign key and one-to-one relationships
that are supported by ``select_related``. It also supports prefetching of
:class:`~django.contrib.contenttypes.fields.GenericRelation` and
:class:`~django.contrib.contenttypes.fields.GenericForeignKey`, however, it
must be restricted to a homogeneous set of results. For example, prefetching
objects referenced by a ``GenericForeignKey`` is only supported if the query
is restricted to one ``ContentType``.

For example, suppose you have these models::

    from django.db import models

    class Topping(models.Model):
        name = models.CharField(max_length=30)

    class Pizza(models.Model):
        name = models.CharField(max_length=50)
        toppings = models.ManyToManyField(Topping)

        def __str__(self):              # __unicode__ on Python 2
            return "%s (%s)" % (
                self.name,
                ", ".join(topping.name for topping in self.toppings.all()),
            )

and run::

    >>> Pizza.objects.all()
    ["Hawaiian (ham, pineapple)", "Seafood (prawns, smoked salmon)"...

The problem with this is that every time ``Pizza.__str__()`` asks for
``self.toppings.all()`` it has to query the database, so
``Pizza.objects.all()`` will run a query on the Toppings table for **every**
item in the Pizza ``QuerySet``.

We can reduce to just two queries using ``prefetch_related``:

    >>> Pizza.objects.all().prefetch_related('toppings')

This implies a ``self.toppings.all()`` for each ``Pizza``; now each time
``self.toppings.all()`` is called, instead of having to go to the database for
the items, it will find them in a prefetched ``QuerySet`` cache that was
populated in a single query.

That is, all the relevant toppings will have been fetched in a single query,
and used to make ``QuerySets`` that have a pre-filled cache of the relevant
results; these ``QuerySets`` are then used in the ``self.toppings.all()`` calls.

The additional queries in ``prefetch_related()`` are executed after the
``QuerySet`` has begun to be evaluated and the primary query has been executed.

If you have an iterable of model instances, you can prefetch related attributes
on those instances using the :func:`~django.db.models.prefetch_related_objects`
function.

Note that the result cache of the primary ``QuerySet`` and all specified related
objects will then be fully loaded into memory. This changes the typical
behavior of ``QuerySets``, which normally try to avoid loading all objects into
memory before they are needed, even after a query has been executed in the
database.

.. note::

    Remember that, as always with ``QuerySets``, any subsequent chained methods
    which imply a different database query will ignore previously cached
    results, and retrieve data using a fresh database query. So, if you write
    the following:

        >>> pizzas = Pizza.objects.prefetch_related('toppings')
        >>> [list(pizza.toppings.filter(spicy=True)) for pizza in pizzas]

    ...then the fact that ``pizza.toppings.all()`` has been prefetched will not
    help you. The ``prefetch_related('toppings')`` implied
    ``pizza.toppings.all()``, but ``pizza.toppings.filter()`` is a new and
    different query. The prefetched cache can't help here; in fact it hurts
    performance, since you have done a database query that you haven't used. So
    use this feature with caution!

You can also use the normal join syntax to do related fields of related
fields. Suppose we have an additional model to the example above::

    class Restaurant(models.Model):
        pizzas = models.ManyToManyField(Pizza, related_name='restaurants')
        best_pizza = models.ForeignKey(Pizza, related_name='championed_by')

The following are all legal:

    >>> Restaurant.objects.prefetch_related('pizzas__toppings')

This will prefetch all pizzas belonging to restaurants, and all toppings
belonging to those pizzas. This will result in a total of 3 database queries -
one for the restaurants, one for the pizzas, and one for the toppings.

    >>> Restaurant.objects.prefetch_related('best_pizza__toppings')

This will fetch the best pizza and all the toppings for the best pizza for each
restaurant. This will be done in 3 database queries - one for the restaurants,
one for the 'best pizzas', and one for one for the toppings.

Of course, the ``best_pizza`` relationship could also be fetched using
``select_related`` to reduce the query count to 2:

    >>> Restaurant.objects.select_related('best_pizza').prefetch_related('best_pizza__toppings')

Since the prefetch is executed after the main query (which includes the joins
needed by ``select_related``), it is able to detect that the ``best_pizza``
objects have already been fetched, and it will skip fetching them again.

Chaining ``prefetch_related`` calls will accumulate the lookups that are
prefetched. To clear any ``prefetch_related`` behavior, pass ``None`` as a
parameter:

   >>> non_prefetched = qs.prefetch_related(None)

One difference to note when using ``prefetch_related`` is that objects created
by a query can be shared between the different objects that they are related to
i.e. a single Python model instance can appear at more than one point in the
tree of objects that are returned. This will normally happen with foreign key
relationships. Typically this behavior will not be a problem, and will in fact
save both memory and CPU time.

While ``prefetch_related`` supports prefetching ``GenericForeignKey``
relationships, the number of queries will depend on the data. Since a
``GenericForeignKey`` can reference data in multiple tables, one query per table
referenced is needed, rather than one query for all the items. There could be
additional queries on the ``ContentType`` table if the relevant rows have not
already been fetched.

``prefetch_related`` in most cases will be implemented using an SQL query that
uses the 'IN' operator. This means that for a large ``QuerySet`` a large 'IN' clause
could be generated, which, depending on the database, might have performance
problems of its own when it comes to parsing or executing the SQL query. Always
profile for your use case!

Note that if you use ``iterator()`` to run the query, ``prefetch_related()``
calls will be ignored since these two optimizations do not make sense together.

You can use the :class:`~django.db.models.Prefetch` object to further control
the prefetch operation.

In its simplest form ``Prefetch`` is equivalent to the traditional string based
lookups:

    >>> from django.db.models import Prefetch
    >>> Restaurant.objects.prefetch_related(Prefetch('pizzas__toppings'))

You can provide a custom queryset with the optional ``queryset`` argument.
This can be used to change the default ordering of the queryset:

    >>> Restaurant.objects.prefetch_related(
    ...     Prefetch('pizzas__toppings', queryset=Toppings.objects.order_by('name')))

Or to call :meth:`~django.db.models.query.QuerySet.select_related()` when
applicable to reduce the number of queries even further:

    >>> Pizza.objects.prefetch_related(
    ...     Prefetch('restaurants', queryset=Restaurant.objects.select_related('best_pizza')))

You can also assign the prefetched result to a custom attribute with the optional
``to_attr`` argument. The result will be stored directly in a list.

This allows prefetching the same relation multiple times with a different
``QuerySet``; for instance:

    >>> vegetarian_pizzas = Pizza.objects.filter(vegetarian=True)
    >>> Restaurant.objects.prefetch_related(
    ...     Prefetch('pizzas', to_attr='menu'),
    ...     Prefetch('pizzas', queryset=vegetarian_pizzas, to_attr='vegetarian_menu'))

Lookups created with custom ``to_attr`` can still be traversed as usual by other
lookups:

    >>> vegetarian_pizzas = Pizza.objects.filter(vegetarian=True)
    >>> Restaurant.objects.prefetch_related(
    ...     Prefetch('pizzas', queryset=vegetarian_pizzas, to_attr='vegetarian_menu'),
    ...     'vegetarian_menu__toppings')

Using ``to_attr`` is recommended when filtering down the prefetch result as it is
less ambiguous than storing a filtered result in the related manager's cache:

    >>> queryset = Pizza.objects.filter(vegetarian=True)
    >>>
    >>> # Recommended:
    >>> restaurants = Restaurant.objects.prefetch_related(
    ...     Prefetch('pizzas', queryset=queryset, to_attr='vegetarian_pizzas'))
    >>> vegetarian_pizzas = restaurants[0].vegetarian_pizzas
    >>>
    >>> # Not recommended:
    >>> restaurants = Restaurant.objects.prefetch_related(
    ...     Prefetch('pizzas', queryset=queryset))
    >>> vegetarian_pizzas = restaurants[0].pizzas.all()

Custom prefetching also works with single related relations like
forward ``ForeignKey`` or ``OneToOneField``. Generally you'll want to use
:meth:`select_related()` for these relations, but there are a number of cases
where prefetching with a custom ``QuerySet`` is useful:

* You want to use a ``QuerySet`` that performs further prefetching
  on related models.

* You want to prefetch only a subset of the related objects.

* You want to use performance optimization techniques like
  :meth:`deferred fields <defer()>`:

    >>> queryset = Pizza.objects.only('name')
    >>>
    >>> restaurants = Restaurant.objects.prefetch_related(
    ...     Prefetch('best_pizza', queryset=queryset))

.. note::

    The ordering of lookups matters.

    Take the following examples:

       >>> prefetch_related('pizzas__toppings', 'pizzas')

    This works even though it's unordered because ``'pizzas__toppings'``
    already contains all the needed information, therefore the second argument
    ``'pizzas'`` is actually redundant.

        >>> prefetch_related('pizzas__toppings', Prefetch('pizzas', queryset=Pizza.objects.all()))

    This will raise a ``ValueError`` because of the attempt to redefine the
    queryset of a previously seen lookup. Note that an implicit queryset was
    created to traverse ``'pizzas'`` as part of the ``'pizzas__toppings'``
    lookup.

        >>> prefetch_related('pizza_list__toppings', Prefetch('pizzas', to_attr='pizza_list'))

    This will trigger an ``AttributeError`` because ``'pizza_list'`` doesn't exist yet
    when ``'pizza_list__toppings'`` is being processed.

    This consideration is not limited to the use of ``Prefetch`` objects. Some
    advanced techniques may require that the lookups be performed in a
    specific order to avoid creating extra queries; therefore it's recommended
    to always carefully order ``prefetch_related`` arguments.

``extra()``
~~~~~~~~~~~

.. method:: extra(select=None, where=None, params=None, tables=None, order_by=None, select_params=None)

Sometimes, the Django query syntax by itself can't easily express a complex
``WHERE`` clause. For these edge cases, Django provides the ``extra()``
``QuerySet`` modifier — a hook for injecting specific clauses into the SQL
generated by a ``QuerySet``.

.. admonition:: Use this method as a last resort

    This is an old API that we aim to deprecate at some point in the future.
    Use it only if you cannot express your query using other queryset methods.
    If you do need to use it, please `file a ticket
    <https://code.djangoproject.com/newticket>`_ using the `QuerySet.extra
    keyword <https://code.djangoproject.com/query?status=assigned&status=new&keywords=~QuerySet.extra>`_
    with your use case (please check the list of existing tickets first) so
    that we can enhance the QuerySet API to allow removing ``extra()``. We are
    no longer improving or fixing bugs for this method.

    For example, this use of ``extra()``::

        >>> qs.extra(
        ...     select={'val': "select col from sometable where othercol = %s"},
        ...     select_params=(someparam,),
        ... )

    is equivalent to::

        >>> qs.annotate(val=RawSQL("select col from sometable where othercol = %s", (someparam,)))

    The main benefit of using :class:`~django.db.models.expressions.RawSQL` is
    that you can set ``output_field`` if needed. The main downside is that if
    you refer to some table alias of the queryset in the raw SQL, then it is
    possible that Django might change that alias (for example, when the
    queryset is used as a subquery in yet another query).

.. warning::

    You should be very careful whenever you use ``extra()``. Every time you use
    it, you should escape any parameters that the user can control by using
    ``params`` in order to protect against SQL injection attacks . Please
    read more about :ref:`SQL injection protection <sql-injection-protection>`.

By definition, these extra lookups may not be portable to different database
engines (because you're explicitly writing SQL code) and violate the DRY
principle, so you should avoid them if possible.

Specify one or more of ``params``, ``select``, ``where`` or ``tables``. None
of the arguments is required, but you should use at least one of them.

* ``select``

  The ``select`` argument lets you put extra fields in the ``SELECT``
  clause.  It should be a dictionary mapping attribute names to SQL
  clauses to use to calculate that attribute.

  Example::

      Entry.objects.extra(select={'is_recent': "pub_date > '2006-01-01'"})

  As a result, each ``Entry`` object will have an extra attribute,
  ``is_recent``, a boolean representing whether the entry's ``pub_date``
  is greater than Jan. 1, 2006.

  Django inserts the given SQL snippet directly into the ``SELECT``
  statement, so the resulting SQL of the above example would be something
  like::

      SELECT blog_entry.*, (pub_date > '2006-01-01') AS is_recent
      FROM blog_entry;


  The next example is more advanced; it does a subquery to give each
  resulting ``Blog`` object an ``entry_count`` attribute, an integer count
  of associated ``Entry`` objects::

      Blog.objects.extra(
          select={
              'entry_count': 'SELECT COUNT(*) FROM blog_entry WHERE blog_entry.blog_id = blog_blog.id'
          },
      )

  In this particular case, we're exploiting the fact that the query will
  already contain the ``blog_blog`` table in its ``FROM`` clause.

  The resulting SQL of the above example would be::

      SELECT blog_blog.*, (SELECT COUNT(*) FROM blog_entry WHERE blog_entry.blog_id = blog_blog.id) AS entry_count
      FROM blog_blog;

  Note that the parentheses required by most database engines around
  subqueries are not required in Django's ``select`` clauses. Also note
  that some database backends, such as some MySQL versions, don't support
  subqueries.

  In some rare cases, you might wish to pass parameters to the SQL
  fragments in ``extra(select=...)``. For this purpose, use the
  ``select_params`` parameter. Since ``select_params`` is a sequence and
  the ``select`` attribute is a dictionary, some care is required so that
  the parameters are matched up correctly with the extra select pieces.
  In this situation, you should use a :class:`collections.OrderedDict` for
  the ``select`` value, not just a normal Python dictionary.

  This will work, for example::

      Blog.objects.extra(
          select=OrderedDict([('a', '%s'), ('b', '%s')]),
          select_params=('one', 'two'))

  If you need to use a literal ``%s`` inside your select string, use
  the sequence ``%%s``.

* ``where`` / ``tables``

  You can define explicit SQL ``WHERE`` clauses — perhaps to perform
  non-explicit joins — by using ``where``. You can manually add tables to
  the SQL ``FROM`` clause by using ``tables``.

  ``where`` and ``tables`` both take a list of strings. All ``where``
  parameters are "AND"ed to any other search criteria.

  Example::

      Entry.objects.extra(where=["foo='a' OR bar = 'a'", "baz = 'a'"])

  ...translates (roughly) into the following SQL::

      SELECT * FROM blog_entry WHERE (foo='a' OR bar='a') AND (baz='a')

  Be careful when using the ``tables`` parameter if you're specifying
  tables that are already used in the query. When you add extra tables
  via the ``tables`` parameter, Django assumes you want that table
  included an extra time, if it is already included. That creates a
  problem, since the table name will then be given an alias. If a table
  appears multiple times in an SQL statement, the second and subsequent
  occurrences must use aliases so the database can tell them apart. If
  you're referring to the extra table you added in the extra ``where``
  parameter this is going to cause errors.

  Normally you'll only be adding extra tables that don't already appear
  in the query. However, if the case outlined above does occur, there are
  a few solutions. First, see if you can get by without including the
  extra table and use the one already in the query. If that isn't
  possible, put your ``extra()`` call at the front of the queryset
  construction so that your table is the first use of that table.
  Finally, if all else fails, look at the query produced and rewrite your
  ``where`` addition to use the alias given to your extra table. The
  alias will be the same each time you construct the queryset in the same
  way, so you can rely upon the alias name to not change.

* ``order_by``

  If you need to order the resulting queryset using some of the new
  fields or tables you have included via ``extra()`` use the ``order_by``
  parameter to ``extra()`` and pass in a sequence of strings. These
  strings should either be model fields (as in the normal
  :meth:`order_by()` method on querysets), of the form
  ``table_name.column_name`` or an alias for a column that you specified
  in the ``select`` parameter to ``extra()``.

  For example::

      q = Entry.objects.extra(select={'is_recent': "pub_date > '2006-01-01'"})
      q = q.extra(order_by = ['-is_recent'])

  This would sort all the items for which ``is_recent`` is true to the
  front of the result set (``True`` sorts before ``False`` in a
  descending ordering).

  This shows, by the way, that you can make multiple calls to ``extra()``
  and it will behave as you expect (adding new constraints each time).

* ``params``

  The ``where`` parameter described above may use standard Python
  database string placeholders — ``'%s'`` to indicate parameters the
  database engine should automatically quote. The ``params`` argument is
  a list of any extra parameters to be substituted.

  Example::

      Entry.objects.extra(where=['headline=%s'], params=['Lennon'])

  Always use ``params`` instead of embedding values directly into
  ``where`` because ``params`` will ensure values are quoted correctly
  according to your particular backend. For example, quotes will be
  escaped correctly.

  Bad::

      Entry.objects.extra(where=["headline='Lennon'"])

  Good::

      Entry.objects.extra(where=['headline=%s'], params=['Lennon'])

.. warning::

    If you are performing queries on MySQL, note that MySQL's silent type coercion
    may cause unexpected results when mixing types. If you query on a string
    type column, but with an integer value, MySQL will coerce the types of all values
    in the table to an integer before performing the comparison. For example, if your
    table contains the values ``'abc'``, ``'def'`` and you query for ``WHERE mycolumn=0``,
    both rows will match. To prevent this, perform the correct typecasting
    before using the value in a query.

``defer()``
~~~~~~~~~~~

.. method:: defer(*fields)

In some complex data-modeling situations, your models might contain a lot of
fields, some of which could contain a lot of data (for example, text fields),
or require expensive processing to convert them to Python objects. If you are
using the results of a queryset in some situation where you don't know
if you need those particular fields when you initially fetch the data, you can
tell Django not to retrieve them from the database.

This is done by passing the names of the fields to not load to ``defer()``::

    Entry.objects.defer("headline", "body")

A queryset that has deferred fields will still return model instances. Each
deferred field will be retrieved from the database if you access that field
(one at a time, not all the deferred fields at once).

You can make multiple calls to ``defer()``. Each call adds new fields to the
deferred set::

    # Defers both the body and headline fields.
    Entry.objects.defer("body").filter(rating=5).defer("headline")

The order in which fields are added to the deferred set does not matter.
Calling ``defer()`` with a field name that has already been deferred is
harmless (the field will still be deferred).

You can defer loading of fields in related models (if the related models are
loading via :meth:`select_related()`) by using the standard double-underscore
notation to separate related fields::

    Blog.objects.select_related().defer("entry__headline", "entry__body")

If you want to clear the set of deferred fields, pass ``None`` as a parameter
to ``defer()``::

    # Load all fields immediately.
    my_queryset.defer(None)

Some fields in a model won't be deferred, even if you ask for them. You can
never defer the loading of the primary key. If you are using
:meth:`select_related()` to retrieve related models, you shouldn't defer the
loading of the field that connects from the primary model to the related
one, doing so will result in an error.

.. note::

    The ``defer()`` method (and its cousin, :meth:`only()`, below) are only for
    advanced use-cases. They provide an optimization for when you have analyzed
    your queries closely and understand *exactly* what information you need and
    have measured that the difference between returning the fields you need and
    the full set of fields for the model will be significant.

    Even if you think you are in the advanced use-case situation, **only use
    defer() when you cannot, at queryset load time, determine if you will need
    the extra fields or not**. If you are frequently loading and using a
    particular subset of your data, the best choice you can make is to
    normalize your models and put the non-loaded data into a separate model
    (and database table). If the columns *must* stay in the one table for some
    reason, create a model with ``Meta.managed = False`` (see the
    :attr:`managed attribute <django.db.models.Options.managed>` documentation)
    containing just the fields you normally need to load and use that where you
    might otherwise call ``defer()``. This makes your code more explicit to the
    reader, is slightly faster and consumes a little less memory in the Python
    process.

    For example, both of these models use the same underlying database table::

        class CommonlyUsedModel(models.Model):
            f1 = models.CharField(max_length=10)

            class Meta:
                managed = False
                db_table = 'app_largetable'

        class ManagedModel(models.Model):
            f1 = models.CharField(max_length=10)
            f2 = models.CharField(max_length=10)

            class Meta:
                db_table = 'app_largetable'

        # Two equivalent QuerySets:
        CommonlyUsedModel.objects.all()
        ManagedModel.objects.all().defer('f2')

    If many fields need to be duplicated in the unmanaged model, it may be best
    to create an abstract model with the shared fields and then have the
    unmanaged and managed models inherit from the abstract model.

.. note::

    When calling :meth:`~django.db.models.Model.save()` for instances with
    deferred fields, only the loaded fields will be saved. See
    :meth:`~django.db.models.Model.save()` for more details.

``only()``
~~~~~~~~~~

.. method:: only(*fields)

The ``only()`` method is more or less the opposite of :meth:`defer()`. You call
it with the fields that should *not* be deferred when retrieving a model.  If
you have a model where almost all the fields need to be deferred, using
``only()`` to specify the complementary set of fields can result in simpler
code.

Suppose you have a model with fields ``name``, ``age`` and ``biography``. The
following two querysets are the same, in terms of deferred fields::

    Person.objects.defer("age", "biography")
    Person.objects.only("name")

Whenever you call ``only()`` it *replaces* the set of fields to load
immediately. The method's name is mnemonic: **only** those fields are loaded
immediately; the remainder are deferred. Thus, successive calls to ``only()``
result in only the final fields being considered::

    # This will defer all fields except the headline.
    Entry.objects.only("body", "rating").only("headline")

Since ``defer()`` acts incrementally (adding fields to the deferred list), you
can combine calls to ``only()`` and ``defer()`` and things will behave
logically::

    # Final result is that everything except "headline" is deferred.
    Entry.objects.only("headline", "body").defer("body")

    # Final result loads headline and body immediately (only() replaces any
    # existing set of fields).
    Entry.objects.defer("body").only("headline", "body")

All of the cautions in the note for the :meth:`defer` documentation apply to
``only()`` as well. Use it cautiously and only after exhausting your other
options.

Using :meth:`only` and omitting a field requested using :meth:`select_related`
is an error as well.

.. note::

    When calling :meth:`~django.db.models.Model.save()` for instances with
    deferred fields, only the loaded fields will be saved. See
    :meth:`~django.db.models.Model.save()` for more details.

``using()``
~~~~~~~~~~~

.. method:: using(alias)

This method is for controlling which database the ``QuerySet`` will be
evaluated against if you are using more than one database.  The only argument
this method takes is the alias of a database, as defined in
:setting:`DATABASES`.

For example::

    # queries the database with the 'default' alias.
    >>> Entry.objects.all()

    # queries the database with the 'backup' alias
    >>> Entry.objects.using('backup')

``select_for_update()``
~~~~~~~~~~~~~~~~~~~~~~~

.. method:: select_for_update(nowait=False)

Returns a queryset that will lock rows until the end of the transaction,
generating a ``SELECT ... FOR UPDATE`` SQL statement on supported databases.

For example::

    entries = Entry.objects.select_for_update().filter(author=request.user)

All matched entries will be locked until the end of the transaction block,
meaning that other transactions will be prevented from changing or acquiring
locks on them.

Usually, if another transaction has already acquired a lock on one of the
selected rows, the query will block until the lock is released. If this is
not the behavior you want, call ``select_for_update(nowait=True)``. This will
make the call non-blocking. If a conflicting lock is already acquired by
another transaction, :exc:`~django.db.DatabaseError` will be raised when the
queryset is evaluated.

Currently, the ``postgresql``, ``oracle``, and ``mysql`` database
backends support ``select_for_update()``. However, MySQL has no support for the
``nowait`` argument. Obviously, users of external third-party backends should
check with their backend's documentation for specifics in those cases.

Passing ``nowait=True`` to ``select_for_update()`` using database backends that
do not support ``nowait``, such as MySQL, will cause a
:exc:`~django.db.DatabaseError` to be raised. This is in order to prevent code
unexpectedly blocking.

Evaluating a queryset with ``select_for_update()`` in autocommit mode on
backends which support ``SELECT ... FOR UPDATE`` is a
:exc:`~django.db.transaction.TransactionManagementError` error because the
rows are not locked in that case. If allowed, this would facilitate data
corruption and could easily be caused by calling code that expects to be run in
a transaction outside of one.

Using ``select_for_update()`` on backends which do not support
``SELECT ... FOR UPDATE`` (such as SQLite) will have no effect.
``SELECT ... FOR UPDATE`` will not be added to the query, and an error isn't
raised if ``select_for_update()`` is used in autocommit mode.

.. warning::

    Although ``select_for_update()`` normally fails in autocommit mode, since
    :class:`~django.test.TestCase` automatically wraps each test in a
    transaction, calling ``select_for_update()`` in a ``TestCase`` even outside
    an :func:`~django.db.transaction.atomic()` block will (perhaps unexpectedly)
    pass without raising a ``TransactionManagementError``. To properly test
    ``select_for_update()`` you should use
    :class:`~django.test.TransactionTestCase`.

``raw()``
~~~~~~~~~

.. method:: raw(raw_query, params=None, translations=None)

Takes a raw SQL query, executes it, and returns a
``django.db.models.query.RawQuerySet`` instance. This ``RawQuerySet`` instance
can be iterated over just like an normal ``QuerySet`` to provide object instances.

See the :doc:`/topics/db/sql` for more information.

.. warning::

  ``raw()`` always triggers a new query and doesn't account for previous
  filtering. As such, it should generally be called from the ``Manager`` or
  from a fresh ``QuerySet`` instance.

Methods that do not return ``QuerySet``\s
-----------------------------------------

The following ``QuerySet`` methods evaluate the ``QuerySet`` and return
something *other than* a ``QuerySet``.

These methods do not use a cache (see :ref:`caching-and-querysets`). Rather,
they query the database each time they're called.

``get()``
~~~~~~~~~

.. method:: get(**kwargs)

Returns the object matching the given lookup parameters, which should be in
the format described in `Field lookups`_.

``get()`` raises :exc:`~django.core.exceptions.MultipleObjectsReturned` if more
than one object was found. The
:exc:`~django.core.exceptions.MultipleObjectsReturned` exception is an
attribute of the model class.

``get()`` raises a :exc:`~django.db.models.Model.DoesNotExist` exception if an
object wasn't found for the given parameters. This exception is an attribute
of the model class. Example::

    Entry.objects.get(id='foo') # raises Entry.DoesNotExist

The :exc:`~django.db.models.Model.DoesNotExist` exception inherits from
:exc:`django.core.exceptions.ObjectDoesNotExist`, so you can target multiple
:exc:`~django.db.models.Model.DoesNotExist` exceptions. Example::

    from django.core.exceptions import ObjectDoesNotExist
    try:
        e = Entry.objects.get(id=3)
        b = Blog.objects.get(id=1)
    except ObjectDoesNotExist:
        print("Either the entry or blog doesn't exist.")

If you expect a queryset to return one row, you can use ``get()`` without any
arguments to return the object for that row::

    entry = Entry.objects.filter(...).exclude(...).get()

``create()``
~~~~~~~~~~~~

.. method:: create(**kwargs)

A convenience method for creating an object and saving it all in one step.  Thus::

    p = Person.objects.create(first_name="Bruce", last_name="Springsteen")

and::

    p = Person(first_name="Bruce", last_name="Springsteen")
    p.save(force_insert=True)

are equivalent.

The :ref:`force_insert <ref-models-force-insert>` parameter is documented
elsewhere, but all it means is that a new object will always be created.
Normally you won't need to worry about this. However, if your model contains a
manual primary key value that you set and if that value already exists in the
database, a call to ``create()`` will fail with an
:exc:`~django.db.IntegrityError` since primary keys must be unique. Be
prepared to handle the exception if you are using manual primary keys.

``get_or_create()``
~~~~~~~~~~~~~~~~~~~

.. method:: get_or_create(defaults=None, **kwargs)

A convenience method for looking up an object with the given ``kwargs`` (may be
empty if your model has defaults for all fields), creating one if necessary.

Returns a tuple of ``(object, created)``, where ``object`` is the retrieved or
created object and ``created`` is a boolean specifying whether a new object was
created.

This is meant as a shortcut to boilerplatish code. For example::

    try:
        obj = Person.objects.get(first_name='John', last_name='Lennon')
    except Person.DoesNotExist:
        obj = Person(first_name='John', last_name='Lennon', birthday=date(1940, 10, 9))
        obj.save()

This pattern gets quite unwieldy as the number of fields in a model goes up.
The above example can be rewritten using ``get_or_create()`` like so::

    obj, created = Person.objects.get_or_create(
        first_name='John',
        last_name='Lennon',
        defaults={'birthday': date(1940, 10, 9)},
    )

Any keyword arguments passed to ``get_or_create()`` — *except* an optional one
called ``defaults`` — will be used in a :meth:`get()` call. If an object is
found, ``get_or_create()`` returns a tuple of that object and ``False``. If
multiple objects are found, ``get_or_create`` raises
:exc:`~django.core.exceptions.MultipleObjectsReturned`. If an object is *not*
found, ``get_or_create()`` will instantiate and save a new object, returning a
tuple of the new object and ``True``. The new object will be created roughly
according to this algorithm::

    params = {k: v for k, v in kwargs.items() if '__' not in k}
    params.update(defaults)
    obj = self.model(**params)
    obj.save()

In English, that means start with any non-``'defaults'`` keyword argument that
doesn't contain a double underscore (which would indicate a non-exact lookup).
Then add the contents of ``defaults``, overriding any keys if necessary, and
use the result as the keyword arguments to the model class. As hinted at
above, this is a simplification of the algorithm that is used, but it contains
all the pertinent details. The internal implementation has some more
error-checking than this and handles some extra edge-conditions; if you're
interested, read the code.

If you have a field named ``defaults`` and want to use it as an exact lookup in
``get_or_create()``, just use ``'defaults__exact'``, like so::

    Foo.objects.get_or_create(defaults__exact='bar', defaults={'defaults': 'baz'})

The ``get_or_create()`` method has similar error behavior to :meth:`create()`
when you're using manually specified primary keys. If an object needs to be
created and the key already exists in the database, an
:exc:`~django.db.IntegrityError` will be raised.

This method is atomic assuming correct usage, correct database configuration,
and correct behavior of the underlying database. However, if uniqueness is not
enforced at the database level for the ``kwargs`` used in a ``get_or_create``
call (see :attr:`~django.db.models.Field.unique` or
:attr:`~django.db.models.Options.unique_together`), this method is prone to a
race-condition which can result in multiple rows with the same parameters being
inserted simultaneously.

If you are using MySQL, be sure to use the ``READ COMMITTED`` isolation level
rather than ``REPEATABLE READ`` (the default), otherwise you may see cases
where ``get_or_create`` will raise an :exc:`~django.db.IntegrityError` but the
object won't appear in a subsequent :meth:`~django.db.models.query.QuerySet.get`
call.

Finally, a word on using ``get_or_create()`` in Django views. Please make sure
to use it only in ``POST`` requests unless you have a good reason not to.
``GET`` requests shouldn't have any effect on data. Instead, use ``POST``
whenever a request to a page has a side effect on your data. For more, see
:rfc:`Safe methods <7231#section-4.2.1>` in the HTTP spec.

.. warning::

  You can use ``get_or_create()`` through :class:`~django.db.models.ManyToManyField`
  attributes and reverse relations. In that case you will restrict the queries
  inside the context of that relation. That could lead you to some integrity
  problems if you don't use it consistently.

  Being the following models::

      class Chapter(models.Model):
          title = models.CharField(max_length=255, unique=True)

      class Book(models.Model):
          title = models.CharField(max_length=256)
          chapters = models.ManyToManyField(Chapter)

  You can use ``get_or_create()`` through Book's chapters field, but it only
  fetches inside the context of that book::

      >>> book = Book.objects.create(title="Ulysses")
      >>> book.chapters.get_or_create(title="Telemachus")
      (<Chapter: Telemachus>, True)
      >>> book.chapters.get_or_create(title="Telemachus")
      (<Chapter: Telemachus>, False)
      >>> Chapter.objects.create(title="Chapter 1")
      <Chapter: Chapter 1>
      >>> book.chapters.get_or_create(title="Chapter 1")
      # Raises IntegrityError

  This is happening because it's trying to get or create "Chapter 1" through the
  book "Ulysses", but it can't do any of them: the relation can't fetch that
  chapter because it isn't related to that book, but it can't create it either
  because ``title`` field should be unique.

``update_or_create()``
~~~~~~~~~~~~~~~~~~~~~~

.. method:: update_or_create(defaults=None, **kwargs)

A convenience method for updating an object with the given ``kwargs``, creating
a new one if necessary. The ``defaults`` is a dictionary of (field, value)
pairs used to update the object.

Returns a tuple of ``(object, created)``, where ``object`` is the created or
updated object and ``created`` is a boolean specifying whether a new object was
created.

The ``update_or_create`` method tries to fetch an object from database based on
the given ``kwargs``. If a match is found, it updates the fields passed in the
``defaults`` dictionary.

This is meant as a shortcut to boilerplatish code. For example::

    defaults = {'first_name': 'Bob'}
    try:
        obj = Person.objects.get(first_name='John', last_name='Lennon')
        for key, value in defaults.items():
            setattr(obj, key, value)
        obj.save()
    except Person.DoesNotExist:
        new_values = {'first_name': 'John', 'last_name': 'Lennon'}
        new_values.update(defaults)
        obj = Person(**new_values)
        obj.save()

This pattern gets quite unwieldy as the number of fields in a model goes up.
The above example can be rewritten using ``update_or_create()`` like so::

    obj, created = Person.objects.update_or_create(
        first_name='John', last_name='Lennon',
        defaults={'first_name': 'Bob'},
    )

For detailed description how names passed in ``kwargs`` are resolved see
:meth:`get_or_create`.

As described above in :meth:`get_or_create`, this method is prone to a
race-condition which can result in multiple rows being inserted simultaneously
if uniqueness is not enforced at the database level.

``bulk_create()``
~~~~~~~~~~~~~~~~~

.. method:: bulk_create(objs, batch_size=None)

This method inserts the provided list of objects into the database in an
efficient manner (generally only 1 query, no matter how many objects there
are)::

    >>> Entry.objects.bulk_create([
    ...     Entry(headline='This is a test'),
    ...     Entry(headline='This is only a test'),
    ... ])

This has a number of caveats though:

* The model's ``save()`` method will not be called, and the ``pre_save`` and
  ``post_save`` signals will not be sent.
* It does not work with child models in a multi-table inheritance scenario.
* If the model's primary key is an :class:`~django.db.models.AutoField` it
  does not retrieve and set the primary key attribute, as ``save()`` does,
  unless the database backend supports it (currently PostgreSQL).
* It does not work with many-to-many relationships.

.. versionchanged:: 1.9

    Support for using ``bulk_create()`` with proxy models was added.

.. versionchanged:: 1.10

    Support for setting primary keys on objects created using ``bulk_create()``
    when using PostgreSQL was added.

The ``batch_size`` parameter controls how many objects are created in single
query. The default is to create all objects in one batch, except for SQLite
where the default is such that at most 999 variables per query are used.

``count()``
~~~~~~~~~~~

.. method:: count()

Returns an integer representing the number of objects in the database matching
the ``QuerySet``. The ``count()`` method never raises exceptions.

Example::

    # Returns the total number of entries in the database.
    Entry.objects.count()

    # Returns the number of entries whose headline contains 'Lennon'
    Entry.objects.filter(headline__contains='Lennon').count()

A ``count()`` call performs a ``SELECT COUNT(*)`` behind the scenes, so you
should always use ``count()`` rather than loading all of the record into Python
objects and calling ``len()`` on the result (unless you need to load the
objects into memory anyway, in which case ``len()`` will be faster).

Depending on which database you're using (e.g. PostgreSQL vs. MySQL),
``count()`` may return a long integer instead of a normal Python integer. This
is an underlying implementation quirk that shouldn't pose any real-world
problems.

Note that if you want the number of items in a ``QuerySet`` and are also
retrieving model instances from it (for example, by iterating over it), it's
probably more efficient to use ``len(queryset)`` which won't cause an extra
database query like ``count()`` would.

``in_bulk()``
~~~~~~~~~~~~~

.. method:: in_bulk(id_list=None)

Takes a list of primary-key values and returns a dictionary mapping each
primary-key value to an instance of the object with the given ID. If a list
isn't provided, all objects in the queryset are returned.

Example::

    >>> Blog.objects.in_bulk([1])
    {1: <Blog: Beatles Blog>}
    >>> Blog.objects.in_bulk([1, 2])
    {1: <Blog: Beatles Blog>, 2: <Blog: Cheddar Talk>}
    >>> Blog.objects.in_bulk([])
    {}
    >>> Blog.objects.in_bulk()
    {1: <Blog: Beatles Blog>, 2: <Blog: Cheddar Talk>, 3: <Blog: Django Weblog>}

If you pass ``in_bulk()`` an empty list, you'll get an empty dictionary.

.. versionchanged:: 1.10

    In older versions, ``id_list`` was a required argument.

``iterator()``
~~~~~~~~~~~~~~

.. method:: iterator()

Evaluates the ``QuerySet`` (by performing the query) and returns an iterator
(see :pep:`234`) over the results. A ``QuerySet`` typically caches its results
internally so that repeated evaluations do not result in additional queries. In
contrast, ``iterator()`` will read results directly, without doing any caching
at the ``QuerySet`` level (internally, the default iterator calls ``iterator()``
and caches the return value). For a ``QuerySet`` which returns a large number of
objects that you only need to access once, this can result in better
performance and a significant reduction in memory.

Note that using ``iterator()`` on a ``QuerySet`` which has already been
evaluated will force it to evaluate again, repeating the query.

Also, use of ``iterator()`` causes previous ``prefetch_related()`` calls to be
ignored since these two optimizations do not make sense together.

.. warning::

    Some Python database drivers like ``psycopg2`` perform caching if using
    client side cursors (instantiated with ``connection.cursor()`` and what
    Django's ORM uses). Using ``iterator()`` does not affect caching at the
    database driver level. To disable this caching, look at `server side
    cursors`_.

.. _server side cursors: http://initd.org/psycopg/docs/usage.html#server-side-cursors

``latest()``
~~~~~~~~~~~~

.. method:: latest(field_name=None)

Returns the latest object in the table, by date, using the ``field_name``
provided as the date field.

This example returns the latest ``Entry`` in the table, according to the
``pub_date`` field::

    Entry.objects.latest('pub_date')

If your model's :ref:`Meta <meta-options>` specifies
:attr:`~django.db.models.Options.get_latest_by`, you can leave off the
``field_name`` argument to ``earliest()`` or ``latest()``. Django will use the
field specified in :attr:`~django.db.models.Options.get_latest_by` by default.

Like :meth:`get()`, ``earliest()`` and ``latest()`` raise
:exc:`~django.db.models.Model.DoesNotExist` if there is no object with the
given parameters.

Note that ``earliest()`` and ``latest()`` exist purely for convenience and
readability.

.. admonition:: ``earliest()`` and ``latest()`` may return instances with null dates.

    Since ordering is delegated to the database, results on fields that allow
    null values may be ordered differently if you use different databases. For
    example, PostgreSQL and MySQL sort null values as if they are higher than
    non-null values, while SQLite does the opposite.

    You may want to filter out null values::

        Entry.objects.filter(pub_date__isnull=False).latest('pub_date')

``earliest()``
~~~~~~~~~~~~~~

.. method:: earliest(field_name=None)

Works otherwise like :meth:`~django.db.models.query.QuerySet.latest` except
the direction is changed.

``first()``
~~~~~~~~~~~

.. method:: first()

Returns the first object matched by the queryset, or ``None`` if there
is no matching object. If the ``QuerySet`` has no ordering defined, then the
queryset is automatically ordered by the primary key.

Example::

    p = Article.objects.order_by('title', 'pub_date').first()

Note that ``first()`` is a convenience method, the following code sample is
equivalent to the above example::

    try:
        p = Article.objects.order_by('title', 'pub_date')[0]
    except IndexError:
        p = None

``last()``
~~~~~~~~~~

.. method:: last()

Works like  :meth:`first()`, but returns the last object in the queryset.

``aggregate()``
~~~~~~~~~~~~~~~

.. method:: aggregate(*args, **kwargs)

Returns a dictionary of aggregate values (averages, sums, etc.) calculated over
the ``QuerySet``. Each argument to ``aggregate()`` specifies a value that will
be included in the dictionary that is returned.

The aggregation functions that are provided by Django are described in
`Aggregation Functions`_ below. Since aggregates are also :doc:`query
expressions </ref/models/expressions>`, you may combine aggregates with other
aggregates or values to create complex aggregates.

Aggregates specified using keyword arguments will use the keyword as the name
for the annotation. Anonymous arguments will have a name generated for them
based upon the name of the aggregate function and the model field that is being
aggregated. Complex aggregates cannot use anonymous arguments and must specify
a keyword argument as an alias.

For example, when you are working with blog entries, you may want to know the
number of authors that have contributed blog entries::

    >>> from django.db.models import Count
    >>> q = Blog.objects.aggregate(Count('entry'))
    {'entry__count': 16}

By using a keyword argument to specify the aggregate function, you can
control the name of the aggregation value that is returned::

    >>> q = Blog.objects.aggregate(number_of_entries=Count('entry'))
    {'number_of_entries': 16}

For an in-depth discussion of aggregation, see :doc:`the topic guide on
Aggregation </topics/db/aggregation>`.

``exists()``
~~~~~~~~~~~~

.. method:: exists()

Returns ``True`` if the :class:`.QuerySet` contains any results, and ``False``
if not. This tries to perform the query in the simplest and fastest way
possible, but it *does* execute nearly the same query as a normal
:class:`.QuerySet` query.

:meth:`~.QuerySet.exists` is useful for searches relating to both
object membership in a :class:`.QuerySet` and to the existence of any objects in
a :class:`.QuerySet`, particularly in the context of a large :class:`.QuerySet`.

The most efficient method of finding whether a model with a unique field
(e.g. ``primary_key``) is a member of a :class:`.QuerySet` is::

    entry = Entry.objects.get(pk=123)
    if some_queryset.filter(pk=entry.pk).exists():
        print("Entry contained in queryset")

Which will be faster than the following which requires evaluating and iterating
through the entire queryset::

    if entry in some_queryset:
       print("Entry contained in QuerySet")

And to find whether a queryset contains any items::

    if some_queryset.exists():
        print("There is at least one object in some_queryset")

Which will be faster than::

    if some_queryset:
        print("There is at least one object in some_queryset")

... but not by a large degree (hence needing a large queryset for efficiency
gains).

Additionally, if a ``some_queryset`` has not yet been evaluated, but you know
that it will be at some point, then using ``some_queryset.exists()`` will do
more overall work (one query for the existence check plus an extra one to later
retrieve the results) than simply using ``bool(some_queryset)``, which
retrieves the results and then checks if any were returned.

``update()``
~~~~~~~~~~~~

.. method:: update(**kwargs)

Performs an SQL update query for the specified fields, and returns
the number of rows matched (which may not be equal to the number of rows
updated if some rows already have the new value).

For example, to turn comments off for all blog entries published in 2010,
you could do this::

    >>> Entry.objects.filter(pub_date__year=2010).update(comments_on=False)

(This assumes your ``Entry`` model has fields ``pub_date`` and ``comments_on``.)

You can update multiple fields — there's no limit on how many. For example,
here we update the ``comments_on`` and ``headline`` fields::

    >>> Entry.objects.filter(pub_date__year=2010).update(comments_on=False, headline='This is old')

The ``update()`` method is applied instantly, and the only restriction on the
:class:`.QuerySet` that is updated is that it can only update columns in the
model's main table, not on related models. You can't do this, for example::

    >>> Entry.objects.update(blog__name='foo') # Won't work!

Filtering based on related fields is still possible, though::

    >>> Entry.objects.filter(blog__id=1).update(comments_on=True)

You cannot call ``update()`` on a :class:`.QuerySet` that has had a slice taken
or can otherwise no longer be filtered.

The ``update()`` method returns the number of affected rows::

    >>> Entry.objects.filter(id=64).update(comments_on=True)
    1

    >>> Entry.objects.filter(slug='nonexistent-slug').update(comments_on=True)
    0

    >>> Entry.objects.filter(pub_date__year=2010).update(comments_on=False)
    132

If you're just updating a record and don't need to do anything with the model
object, the most efficient approach is to call ``update()``, rather than
loading the model object into memory. For example, instead of doing this::

    e = Entry.objects.get(id=10)
    e.comments_on = False
    e.save()

...do this::

    Entry.objects.filter(id=10).update(comments_on=False)

Using ``update()`` also prevents a race condition wherein something might
change in your database in the short period of time between loading the object
and calling ``save()``.

Finally, realize that ``update()`` does an update at the SQL level and, thus,
does not call any ``save()`` methods on your models, nor does it emit the
:attr:`~django.db.models.signals.pre_save` or
:attr:`~django.db.models.signals.post_save` signals (which are a consequence of
calling :meth:`Model.save() <django.db.models.Model.save>`). If you want to
update a bunch of records for a model that has a custom
:meth:`~django.db.models.Model.save()` method, loop over them and call
:meth:`~django.db.models.Model.save()`, like this::

    for e in Entry.objects.filter(pub_date__year=2010):
        e.comments_on = False
        e.save()

``delete()``
~~~~~~~~~~~~

.. method:: delete()

Performs an SQL delete query on all rows in the :class:`.QuerySet` and
returns the number of objects deleted and a dictionary with the number of
deletions per object type.

The ``delete()`` is applied instantly. You cannot call ``delete()`` on a
:class:`.QuerySet` that has had a slice taken or can otherwise no longer be
filtered.

For example, to delete all the entries in a particular blog::

    >>> b = Blog.objects.get(pk=1)

    # Delete all the entries belonging to this Blog.
    >>> Entry.objects.filter(blog=b).delete()
    (4, {'weblog.Entry': 2, 'weblog.Entry_authors': 2})

.. versionchanged:: 1.9

    The return value describing the number of objects deleted was added.

By default, Django's :class:`~django.db.models.ForeignKey` emulates the SQL
constraint ``ON DELETE CASCADE`` — in other words, any objects with foreign
keys pointing at the objects to be deleted will be deleted along with them.
For example::

    >>> blogs = Blog.objects.all()

    # This will delete all Blogs and all of their Entry objects.
    >>> blogs.delete()
    (5, {'weblog.Blog': 1, 'weblog.Entry': 2, 'weblog.Entry_authors': 2})

This cascade behavior is customizable via the
:attr:`~django.db.models.ForeignKey.on_delete` argument to the
:class:`~django.db.models.ForeignKey`.

The ``delete()`` method does a bulk delete and does not call any ``delete()``
methods on your models. It does, however, emit the
:data:`~django.db.models.signals.pre_delete` and
:data:`~django.db.models.signals.post_delete` signals for all deleted objects
(including cascaded deletions).

Django needs to fetch objects into memory to send signals and handle cascades.
However, if there are no cascades and no signals, then Django may take a
fast-path and delete objects without fetching into memory. For large
deletes this can result in significantly reduced memory usage. The amount of
executed queries can be reduced, too.

ForeignKeys which are set to :attr:`~django.db.models.ForeignKey.on_delete`
``DO_NOTHING`` do not prevent taking the fast-path in deletion.

Note that the queries generated in object deletion is an implementation
detail subject to change.

``as_manager()``
~~~~~~~~~~~~~~~~

.. classmethod:: as_manager()

Class method that returns an instance of :class:`~django.db.models.Manager`
with a copy of the ``QuerySet``’s methods. See
:ref:`create-manager-with-queryset-methods` for more details.

.. _field-lookups:

``Field`` lookups
-----------------

Field lookups are how you specify the meat of an SQL ``WHERE`` clause. They're
specified as keyword arguments to the ``QuerySet`` methods :meth:`filter()`,
:meth:`exclude()` and :meth:`get()`.

For an introduction, see :ref:`models and database queries documentation
<field-lookups-intro>`.

Django's built-in lookups are listed below. It is also possible to write
:doc:`custom lookups </howto/custom-lookups>` for model fields.

As a convenience when no lookup type is provided (like in
``Entry.objects.get(id=14)``) the lookup type is assumed to be :lookup:`exact`.

.. fieldlookup:: exact

``exact``
~~~~~~~~~

Exact match. If the value provided for comparison is ``None``, it will be
interpreted as an SQL ``NULL`` (see :lookup:`isnull` for more details).

Examples::

    Entry.objects.get(id__exact=14)
    Entry.objects.get(id__exact=None)

SQL equivalents::

    SELECT ... WHERE id = 14;
    SELECT ... WHERE id IS NULL;

.. admonition:: MySQL comparisons

    In MySQL, a database table's "collation" setting determines whether
    ``exact`` comparisons are case-sensitive. This is a database setting, *not*
    a Django setting. It's possible to configure your MySQL tables to use
    case-sensitive comparisons, but some trade-offs are involved. For more
    information about this, see the :ref:`collation section <mysql-collation>`
    in the :doc:`databases </ref/databases>` documentation.

.. fieldlookup:: iexact

``iexact``
~~~~~~~~~~

Case-insensitive exact match. If the value provided for comparison is ``None``,
it will be interpreted as an SQL ``NULL`` (see :lookup:`isnull` for more
details).

Example::

    Blog.objects.get(name__iexact='beatles blog')
    Blog.objects.get(name__iexact=None)

SQL equivalents::

    SELECT ... WHERE name ILIKE 'beatles blog';
    SELECT ... WHERE name IS NULL;

Note the first query will match ``'Beatles Blog'``, ``'beatles blog'``,
``'BeAtLes BLoG'``, etc.

.. admonition:: SQLite users

    When using the SQLite backend and Unicode (non-ASCII) strings, bear in
    mind the :ref:`database note <sqlite-string-matching>` about string
    comparisons. SQLite does not do case-insensitive matching for Unicode
    strings.

.. fieldlookup:: contains

``contains``
~~~~~~~~~~~~

Case-sensitive containment test.

Example::

    Entry.objects.get(headline__contains='Lennon')

SQL equivalent::

    SELECT ... WHERE headline LIKE '%Lennon%';

Note this will match the headline ``'Lennon honored today'`` but not ``'lennon
honored today'``.

.. admonition:: SQLite users

    SQLite doesn't support case-sensitive ``LIKE`` statements; ``contains``
    acts like ``icontains`` for SQLite. See the :ref:`database note
    <sqlite-string-matching>` for more information.


.. fieldlookup:: icontains

``icontains``
~~~~~~~~~~~~~

Case-insensitive containment test.

Example::

    Entry.objects.get(headline__icontains='Lennon')

SQL equivalent::

    SELECT ... WHERE headline ILIKE '%Lennon%';

.. admonition:: SQLite users

    When using the SQLite backend and Unicode (non-ASCII) strings, bear in
    mind the :ref:`database note <sqlite-string-matching>` about string
    comparisons.

.. fieldlookup:: in

``in``
~~~~~~

In a given list.

Example::

    Entry.objects.filter(id__in=[1, 3, 4])

SQL equivalent::

    SELECT ... WHERE id IN (1, 3, 4);

You can also use a queryset to dynamically evaluate the list of values
instead of providing a list of literal values::

    inner_qs = Blog.objects.filter(name__contains='Cheddar')
    entries = Entry.objects.filter(blog__in=inner_qs)

This queryset will be evaluated as subselect statement::

    SELECT ... WHERE blog.id IN (SELECT id FROM ... WHERE NAME LIKE '%Cheddar%')

If you pass in a ``QuerySet`` resulting from ``values()`` or ``values_list()``
as the value to an ``__in`` lookup, you need to ensure you are only extracting
one field in the result. For example, this will work (filtering on the blog
names)::

    inner_qs = Blog.objects.filter(name__contains='Ch').values('name')
    entries = Entry.objects.filter(blog__name__in=inner_qs)

This example will raise an exception, since the inner query is trying to
extract two field values, where only one is expected::

    # Bad code! Will raise a TypeError.
    inner_qs = Blog.objects.filter(name__contains='Ch').values('name', 'id')
    entries = Entry.objects.filter(blog__name__in=inner_qs)

.. _nested-queries-performance:

.. admonition:: Performance considerations

    Be cautious about using nested queries and understand your database
    server's performance characteristics (if in doubt, benchmark!). Some
    database backends, most notably MySQL, don't optimize nested queries very
    well. It is more efficient, in those cases, to extract a list of values
    and then pass that into the second query. That is, execute two queries
    instead of one::

        values = Blog.objects.filter(
                name__contains='Cheddar').values_list('pk', flat=True)
        entries = Entry.objects.filter(blog__in=list(values))

    Note the ``list()`` call around the Blog ``QuerySet`` to force execution of
    the first query. Without it, a nested query would be executed, because
    :ref:`querysets-are-lazy`.

.. fieldlookup:: gt

``gt``
~~~~~~

Greater than.

Example::

    Entry.objects.filter(id__gt=4)

SQL equivalent::

    SELECT ... WHERE id > 4;

.. fieldlookup:: gte

``gte``
~~~~~~~

Greater than or equal to.

.. fieldlookup:: lt

``lt``
~~~~~~

Less than.

.. fieldlookup:: lte

``lte``
~~~~~~~

Less than or equal to.

.. fieldlookup:: startswith

``startswith``
~~~~~~~~~~~~~~

Case-sensitive starts-with.

Example::

    Entry.objects.filter(headline__startswith='Will')

SQL equivalent::

    SELECT ... WHERE headline LIKE 'Will%';

SQLite doesn't support case-sensitive ``LIKE`` statements; ``startswith`` acts
like ``istartswith`` for SQLite.

.. fieldlookup:: istartswith

``istartswith``
~~~~~~~~~~~~~~~

Case-insensitive starts-with.

Example::

    Entry.objects.filter(headline__istartswith='will')

SQL equivalent::

    SELECT ... WHERE headline ILIKE 'Will%';

.. admonition:: SQLite users

    When using the SQLite backend and Unicode (non-ASCII) strings, bear in
    mind the :ref:`database note <sqlite-string-matching>` about string
    comparisons.

.. fieldlookup:: endswith

``endswith``
~~~~~~~~~~~~

Case-sensitive ends-with.

Example::

    Entry.objects.filter(headline__endswith='cats')

SQL equivalent::

    SELECT ... WHERE headline LIKE '%cats';

.. admonition:: SQLite users

    SQLite doesn't support case-sensitive ``LIKE`` statements; ``endswith``
    acts like ``iendswith`` for SQLite. Refer to the :ref:`database note
    <sqlite-string-matching>` documentation for more.

.. fieldlookup:: iendswith

``iendswith``
~~~~~~~~~~~~~

Case-insensitive ends-with.

Example::

    Entry.objects.filter(headline__iendswith='will')

SQL equivalent::

    SELECT ... WHERE headline ILIKE '%will'

.. admonition:: SQLite users

    When using the SQLite backend and Unicode (non-ASCII) strings, bear in
    mind the :ref:`database note <sqlite-string-matching>` about string
    comparisons.

.. fieldlookup:: range

``range``
~~~~~~~~~

Range test (inclusive).

Example::

    import datetime
    start_date = datetime.date(2005, 1, 1)
    end_date = datetime.date(2005, 3, 31)
    Entry.objects.filter(pub_date__range=(start_date, end_date))

SQL equivalent::

    SELECT ... WHERE pub_date BETWEEN '2005-01-01' and '2005-03-31';

You can use ``range`` anywhere you can use ``BETWEEN`` in SQL — for dates,
numbers and even characters.

.. warning::

    Filtering a ``DateTimeField`` with dates won't include items on the last
    day, because the bounds are interpreted as "0am on the given date". If
    ``pub_date`` was a ``DateTimeField``, the above expression would be turned
    into this SQL::

        SELECT ... WHERE pub_date BETWEEN '2005-01-01 00:00:00' and '2005-03-31 00:00:00';

    Generally speaking, you can't mix dates and datetimes.

.. fieldlookup:: date

``date``
~~~~~~~~

.. versionadded:: 1.9

For datetime fields, casts the value as date. Allows chaining additional field
lookups. Takes a date value.

Example::

    Entry.objects.filter(pub_date__date=datetime.date(2005, 1, 1))
    Entry.objects.filter(pub_date__date__gt=datetime.date(2005, 1, 1))

(No equivalent SQL code fragment is included for this lookup because
implementation of the relevant query varies among different database engines.)

When :setting:`USE_TZ` is ``True``, fields are converted to the current time
zone before filtering.

.. fieldlookup:: year

``year``
~~~~~~~~

For date and datetime fields, an exact year match. Allows chaining additional
field lookups. Takes an integer year.

Example::

    Entry.objects.filter(pub_date__year=2005)
    Entry.objects.filter(pub_date__year__gte=2005)

SQL equivalent::

    SELECT ... WHERE pub_date BETWEEN '2005-01-01' AND '2005-12-31';
    SELECT ... WHERE pub_date >= '2005-01-01';

(The exact SQL syntax varies for each database engine.)

When :setting:`USE_TZ` is ``True``, datetime fields are converted to the
current time zone before filtering.

.. versionchanged:: 1.9

    Allowed chaining additional field lookups.

.. fieldlookup:: month

``month``
~~~~~~~~~

For date and datetime fields, an exact month match. Allows chaining additional
field lookups. Takes an integer 1 (January) through 12 (December).

Example::

    Entry.objects.filter(pub_date__month=12)
    Entry.objects.filter(pub_date__month__gte=6)

SQL equivalent::

    SELECT ... WHERE EXTRACT('month' FROM pub_date) = '12';
    SELECT ... WHERE EXTRACT('month' FROM pub_date) >= '6';

(The exact SQL syntax varies for each database engine.)

When :setting:`USE_TZ` is ``True``, datetime fields are converted to the
current time zone before filtering. This requires :ref:`time zone definitions
in the database <database-time-zone-definitions>`.

.. versionchanged:: 1.9

    Allowed chaining additional field lookups.

.. fieldlookup:: day

``day``
~~~~~~~

For date and datetime fields, an exact day match. Allows chaining additional
field lookups. Takes an integer day.

Example::

    Entry.objects.filter(pub_date__day=3)
    Entry.objects.filter(pub_date__day__gte=3)

SQL equivalent::

    SELECT ... WHERE EXTRACT('day' FROM pub_date) = '3';
    SELECT ... WHERE EXTRACT('day' FROM pub_date) >= '3';

(The exact SQL syntax varies for each database engine.)

Note this will match any record with a pub_date on the third day of the month,
such as January 3, July 3, etc.

When :setting:`USE_TZ` is ``True``, datetime fields are converted to the
current time zone before filtering. This requires :ref:`time zone definitions
in the database <database-time-zone-definitions>`.

.. versionchanged:: 1.9

    Allowed chaining additional field lookups.

.. fieldlookup:: week_day

``week_day``
~~~~~~~~~~~~

For date and datetime fields, a 'day of the week' match. Allows chaining
additional field lookups.

Takes an integer value representing the day of week from 1 (Sunday) to 7
(Saturday).

Example::

    Entry.objects.filter(pub_date__week_day=2)
    Entry.objects.filter(pub_date__week_day__gte=2)

(No equivalent SQL code fragment is included for this lookup because
implementation of the relevant query varies among different database engines.)

Note this will match any record with a ``pub_date`` that falls on a Monday (day
2 of the week), regardless of the month or year in which it occurs. Week days
are indexed with day 1 being Sunday and day 7 being Saturday.

When :setting:`USE_TZ` is ``True``, datetime fields are converted to the
current time zone before filtering. This requires :ref:`time zone definitions
in the database <database-time-zone-definitions>`.

.. versionchanged:: 1.9

    Allowed chaining additional field lookups.

.. fieldlookup:: hour

``hour``
~~~~~~~~

For datetime and time fields, an exact hour match. Allows chaining additional
field lookups. Takes an integer between 0 and 23.

Example::

    Event.objects.filter(timestamp__hour=23)
    Event.objects.filter(time__hour=5)
    Event.objects.filter(timestamp__hour__gte=12)

SQL equivalent::

    SELECT ... WHERE EXTRACT('hour' FROM timestamp) = '23';
    SELECT ... WHERE EXTRACT('hour' FROM time) = '5';
    SELECT ... WHERE EXTRACT('hour' FROM timestamp) >= '12';

(The exact SQL syntax varies for each database engine.)

For datetime fields, when :setting:`USE_TZ` is ``True``, values are converted
to the current time zone before filtering.

.. versionchanged:: 1.9

    Added support for :class:`~django.db.models.TimeField` on SQLite (other
    databases supported it as of 1.7).

.. versionchanged:: 1.9

    Allowed chaining additional field lookups.

.. fieldlookup:: minute

``minute``
~~~~~~~~~~

For datetime and time fields, an exact minute match. Allows chaining additional
field lookups. Takes an integer between 0 and 59.

Example::

    Event.objects.filter(timestamp__minute=29)
    Event.objects.filter(time__minute=46)
    Event.objects.filter(timestamp__minute__gte=29)

SQL equivalent::

    SELECT ... WHERE EXTRACT('minute' FROM timestamp) = '29';
    SELECT ... WHERE EXTRACT('minute' FROM time) = '46';
    SELECT ... WHERE EXTRACT('minute' FROM timestamp) >= '29';

(The exact SQL syntax varies for each database engine.)

For datetime fields, When :setting:`USE_TZ` is ``True``, values are converted
to the current time zone before filtering.

.. versionchanged:: 1.9

    Added support for :class:`~django.db.models.TimeField` on SQLite (other
    databases supported it as of 1.7).

.. versionchanged:: 1.9

    Allowed chaining additional field lookups.

.. fieldlookup:: second

``second``
~~~~~~~~~~

For datetime and time fields, an exact second match. Allows chaining additional
field lookups. Takes an integer between 0 and 59.

Example::

    Event.objects.filter(timestamp__second=31)
    Event.objects.filter(time__second=2)
    Event.objects.filter(timestamp__second__gte=31)

SQL equivalent::

    SELECT ... WHERE EXTRACT('second' FROM timestamp) = '31';
    SELECT ... WHERE EXTRACT('second' FROM time) = '2';
    SELECT ... WHERE EXTRACT('second' FROM timestamp) >= '31';

(The exact SQL syntax varies for each database engine.)

For datetime fields, when :setting:`USE_TZ` is ``True``, values are converted
to the current time zone before filtering.

.. versionchanged:: 1.9

    Added support for :class:`~django.db.models.TimeField` on SQLite (other
    databases supported it as of 1.7).

.. versionchanged:: 1.9

    Allowed chaining additional field lookups.

.. fieldlookup:: isnull

``isnull``
~~~~~~~~~~

Takes either ``True`` or ``False``, which correspond to SQL queries of
``IS NULL`` and ``IS NOT NULL``, respectively.

Example::

    Entry.objects.filter(pub_date__isnull=True)

SQL equivalent::

    SELECT ... WHERE pub_date IS NULL;

.. fieldlookup:: search

``search``
~~~~~~~~~~

.. deprecated:: 1.10

    See :ref:`the 1.10 release notes <search-lookup-replacement>` for how to
    replace it.

A boolean full-text search, taking advantage of full-text indexing. This is
like :lookup:`contains` but is significantly faster due to full-text indexing.

Example::

    Entry.objects.filter(headline__search="+Django -jazz Python")

SQL equivalent::

    SELECT ... WHERE MATCH(tablename, headline) AGAINST (+Django -jazz Python IN BOOLEAN MODE);

Note this is only available in MySQL and requires direct manipulation of the
database to add the full-text index. By default Django uses BOOLEAN MODE for
full text searches. See the `MySQL documentation`_ for additional details.

.. _MySQL documentation: https://dev.mysql.com/doc/refman/en/fulltext-boolean.html

.. fieldlookup:: regex

``regex``
~~~~~~~~~

Case-sensitive regular expression match.

The regular expression syntax is that of the database backend in use.
In the case of SQLite, which has no built in regular expression support,
this feature is provided by a (Python) user-defined REGEXP function, and
the regular expression syntax is therefore that of Python's ``re`` module.

Example::

    Entry.objects.get(title__regex=r'^(An?|The) +')

SQL equivalents::

    SELECT ... WHERE title REGEXP BINARY '^(An?|The) +'; -- MySQL

    SELECT ... WHERE REGEXP_LIKE(title, '^(An?|The) +', 'c'); -- Oracle

    SELECT ... WHERE title ~ '^(An?|The) +'; -- PostgreSQL

    SELECT ... WHERE title REGEXP '^(An?|The) +'; -- SQLite

Using raw strings (e.g., ``r'foo'`` instead of ``'foo'``) for passing in the
regular expression syntax is recommended.

.. fieldlookup:: iregex

``iregex``
~~~~~~~~~~

Case-insensitive regular expression match.

Example::

    Entry.objects.get(title__iregex=r'^(an?|the) +')

SQL equivalents::

    SELECT ... WHERE title REGEXP '^(an?|the) +'; -- MySQL

    SELECT ... WHERE REGEXP_LIKE(title, '^(an?|the) +', 'i'); -- Oracle

    SELECT ... WHERE title ~* '^(an?|the) +'; -- PostgreSQL

    SELECT ... WHERE title REGEXP '(?i)^(an?|the) +'; -- SQLite

.. _aggregation-functions:

Aggregation functions
---------------------

.. currentmodule:: django.db.models

Django provides the following aggregation functions in the
``django.db.models`` module. For details on how to use these
aggregate functions, see :doc:`the topic guide on aggregation
</topics/db/aggregation>`. See the :class:`~django.db.models.Aggregate`
documentation to learn how to create your aggregates.

.. warning::

    SQLite can't handle aggregation on date/time fields out of the box.
    This is because there are no native date/time fields in SQLite and Django
    currently emulates these features using a text field. Attempts to use
    aggregation on date/time fields in SQLite will raise
    ``NotImplementedError``.

.. admonition:: Note

    Aggregation functions return ``None`` when used with an empty
    ``QuerySet``. For example, the ``Sum`` aggregation function returns ``None``
    instead of ``0`` if the ``QuerySet`` contains no entries. An exception is
    ``Count``, which does return ``0`` if the ``QuerySet`` is empty.

All aggregates have the following parameters in common:

``expression``
~~~~~~~~~~~~~~

A string that references a field on the model, or a :doc:`query expression
</ref/models/expressions>`.

``output_field``
~~~~~~~~~~~~~~~~

An optional argument that represents the :doc:`model field </ref/models/fields>`
of the return value

.. note::

    When combining multiple field types, Django can only determine the
    ``output_field`` if all fields are of the same type. Otherwise, you
    must provide the ``output_field`` yourself.

``**extra``
~~~~~~~~~~~

Keyword arguments that can provide extra context for the SQL generated
by the aggregate.

``Avg``
~~~~~~~

.. class:: Avg(expression, output_field=FloatField(), **extra)

    Returns the mean value of the given expression, which must be numeric
    unless you specify a different ``output_field``.

    * Default alias: ``<field>__avg``
    * Return type: ``float`` (or the type of whatever ``output_field`` is
      specified)

    .. versionchanged:: 1.9

        The ``output_field`` parameter was added to allow aggregating over
        non-numeric columns, such as ``DurationField``.

``Count``
~~~~~~~~~

.. class:: Count(expression, distinct=False, **extra)

    Returns the number of objects that are related through the provided
    expression.

    * Default alias: ``<field>__count``
    * Return type: ``int``

    Has one optional argument:

    .. attribute:: distinct

        If ``distinct=True``, the count will only include unique instances.
        This is the SQL equivalent of ``COUNT(DISTINCT <field>)``. The default
        value is ``False``.

``Max``
~~~~~~~

.. class:: Max(expression, output_field=None, **extra)

    Returns the maximum value of the given expression.

    * Default alias: ``<field>__max``
    * Return type: same as input field, or ``output_field`` if supplied

``Min``
~~~~~~~

.. class:: Min(expression, output_field=None, **extra)

    Returns the minimum value of the given expression.

    * Default alias: ``<field>__min``
    * Return type: same as input field, or ``output_field`` if supplied

``StdDev``
~~~~~~~~~~

.. class:: StdDev(expression, sample=False, **extra)

    Returns the standard deviation of the data in the provided expression.

    * Default alias: ``<field>__stddev``
    * Return type: ``float``

    Has one optional argument:

    .. attribute:: sample

        By default, ``StdDev`` returns the population standard deviation. However,
        if ``sample=True``, the return value will be the sample standard deviation.

    .. admonition:: SQLite

        SQLite doesn't provide ``StdDev`` out of the box. An implementation
        is available as an extension module for SQLite. Consult the `SQlite
        documentation`_ for instructions on obtaining and installing this
        extension.

``Sum``
~~~~~~~

.. class:: Sum(expression, output_field=None, **extra)

    Computes the sum of all values of the given expression.

    * Default alias: ``<field>__sum``
    * Return type: same as input field, or ``output_field`` if supplied

``Variance``
~~~~~~~~~~~~

.. class:: Variance(expression, sample=False, **extra)

    Returns the variance of the data in the provided expression.

    * Default alias: ``<field>__variance``
    * Return type: ``float``

    Has one optional argument:

    .. attribute:: sample

        By default, ``Variance`` returns the population variance. However,
        if ``sample=True``, the return value will be the sample variance.

    .. admonition:: SQLite

        SQLite doesn't provide ``Variance`` out of the box. An implementation
        is available as an extension module for SQLite. Consult the `SQlite
        documentation`_ for instructions on obtaining and installing this
        extension.

.. _SQLite documentation: https://www.sqlite.org/contrib

Query-related tools
===================

This section provides reference material for query-related tools not documented
elsewhere.

``Q()`` objects
---------------

.. class:: Q

A ``Q()`` object, like an :class:`~django.db.models.F` object, encapsulates a
SQL expression in a Python object that can be used in database-related
operations.

In general, ``Q() objects`` make it possible to define and reuse conditions.
This permits the :ref:`construction of complex database queries
<complex-lookups-with-q>` using ``|`` (``OR``) and ``&`` (``AND``) operators;
in particular, it is not otherwise possible to use ``OR`` in ``QuerySets``.

``Prefetch()`` objects
----------------------

.. class:: Prefetch(lookup, queryset=None, to_attr=None)

The ``Prefetch()`` object can be used to control the operation of
:meth:`~django.db.models.query.QuerySet.prefetch_related()`.

The ``lookup`` argument describes the relations to follow and works the same
as the string based lookups passed to
:meth:`~django.db.models.query.QuerySet.prefetch_related()`. For example:

    >>> from django.db.models import Prefetch
    >>> Question.objects.prefetch_related(Prefetch('choice_set')).get().choice_set.all()
    <QuerySet [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]>
    # This will only execute two queries regardless of the number of Question
    # and Choice objects.
    >>> Question.objects.prefetch_related(Prefetch('choice_set')).all()
    <QuerySet [<Question: Question object>]>

The ``queryset`` argument supplies a base ``QuerySet`` for the given lookup.
This is useful to further filter down the prefetch operation, or to call
:meth:`~django.db.models.query.QuerySet.select_related()` from the prefetched
relation, hence reducing the number of queries even further:

    >>> voted_choices = Choice.objects.filter(votes__gt=0)
    >>> voted_choices
    <QuerySet [<Choice: The sky>]>
    >>> prefetch = Prefetch('choice_set', queryset=voted_choices)
    >>> Question.objects.prefetch_related(prefetch).get().choice_set.all()
    <QuerySet [<Choice: The sky>]>

The ``to_attr`` argument sets the result of the prefetch operation to a custom
attribute:

    >>> prefetch = Prefetch('choice_set', queryset=voted_choices, to_attr='voted_choices')
    >>> Question.objects.prefetch_related(prefetch).get().voted_choices
    <QuerySet [<Choice: The sky>]>
    >>> Question.objects.prefetch_related(prefetch).get().choice_set.all()
    <QuerySet [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]>

.. note::

    When using ``to_attr`` the prefetched result is stored in a list. This can
    provide a significant speed improvement over traditional
    ``prefetch_related`` calls which store the cached result within a
    ``QuerySet`` instance.

``prefetch_related_objects()``
------------------------------

.. function:: prefetch_related_objects(model_instances, *related_lookups)

.. versionadded:: 1.10

Prefetches the given lookups on an iterable of model instances. This is useful
in code that receives a list of model instances as opposed to a ``QuerySet``;
for example, when fetching models from a cache or instantiating them manually.

Pass an iterable of model instances (must all be of the same class) and the
lookups or :class:`Prefetch` objects you want to prefetch for. For example::

    >>> from django.db.models import prefetch_related_objects
    >>> restaurants = fetch_top_restaurants_from_cache()  # A list of Restaurants
    >>> prefetch_related_objects(restaurants, 'pizzas__toppings')
