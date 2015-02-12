.. _reqparse:

请求解析
===============

.. currentmodule:: flask.ext.restful

Flask-RESTful 的请求解析接口是模仿 ``argparse`` 接口。它设计成提供简单并且统一的访问 Flask 中 :py:class:`flask.request` 对象里的任何变量的入口。

基本参数
---------------

这里是请求解析一个简单的例子。它寻找在 :py:attr:`flask.Request.values` 字典里的两个参数。一个类型为 ``int``，另一个的类型是 ``str`` ::

    from flask.ext.restful import reqparse

    parser = reqparse.RequestParser()
    parser.add_argument('rate', type=int, help='Rate cannot be converted')
    parser.add_argument('name', type=str)
    args = parser.parse_args()

如果你指定了 help 参数的值，在解析的时候当类型错误被触发的时候，它将会被作为错误信息给呈现出来。如果你没有指定 help 信息的话，默认行为是返回类型错误本身的信息。


默认下，arguments **不是** 必须的。另外，在请求中提供的参数不属于 RequestParser 的一部分的话将会被忽略。

另请注意：在请求解析中声明的参数如果没有在请求本身设置的话将默认为 ``None``。

必需的参数
------------------

要求一个值传递的参数，只需要添加 ``required=True`` 来调用 :py:meth:`~reqparse.RequestParser.add_argument`。 ::

    parser.add_argument('name', type=str, required=True,
    help="Name cannot be blank!")

多个值&列表
-----------------------

如果你要接受一个键多个值的话，你可以传入 ``action='append'`` ::

    parser.add_argument('name', type=str, action='append')

这将让你做出这样的查询 ::

    curl http://api.example.com -d "Name=bob" -d "Name=sue" -d "Name=joe"

你的参数将会像这样 ::

    args = parser.parse_args()
    args['name']    # ['bob', 'sue', 'joe']

其它目标（Destinations）
--------------------------

If for some reason you'd like your argument stored under a different name once
it's parsed, you can use the ``dest`` kwarg. ::

    parser.add_argument('name', type=str, dest='public_name')

    args = parser.parse_args()
    args['public_name']

参数位置
------------------

By default, the :py:class:`~reqparse.RequestParser` tries to parse values
from :py:attr:`flask.Request.values`, and :py:attr:`flask.Request.json`.

Use the ``location`` argument to :py:meth:`~reqparse.RequestParser.add_argument`
to specify alternate locations to pull the values from. Any variable on the
:py:class:`flask.Request` can be used. For example: ::

    # Look only in the POST body
    parser.add_argument('name', type=int, location='form')

    # Look only in the querystring
    parser.add_argument('PageSize', type=int, location='args')

    # From the request headers
    parser.add_argument('User-Agent', type=str, location='headers')

    # From http cookies
    parser.add_argument('session_id', type=str, location='cookies')

    # From file uploads
    parser.add_argument('picture', type=werkzeug.datastructures.FileStorage, location='files')

多个位置
------------------

Multiple argument locations can be specified by passing a list to ``location``::

    parser.add_argument('text', location=['headers', 'values'])

The last location listed takes precedence in the result set.

继承解析
------------------

Often you will make a different parser for each resource you write.
The problem with this is if parsers have arguments in common. Instead of
rewriting arguments you can write a parent parser containing all the
shared arguments and then extend the parser with
:py:meth:`~reqparse.RequestParser.copy`. You can also overwrite any argument
in the parent with :py:meth:`~reqparse.RequestParser.replace_argument`, or remove
it completely with :py:meth:`~reqparse.RequestParser.remove_argument`.
For example: ::

    from flask.ext.restful import RequestParser

    parser = RequestParser()
    parser.add_argument('foo', type=int)

    parser_copy = parser.copy()
    parser_copy.add_argument('bar', type=int)

    # parser_copy has both 'foo' and 'bar'

    parser_copy.replace_argument('foo', type=str, required=True, location='json')
    # 'foo' is now a required str located in json, not an int as defined
    #  by original parser

    parser_copy.remove_argument('foo')
    # parser_copy no longer has 'foo' argument
