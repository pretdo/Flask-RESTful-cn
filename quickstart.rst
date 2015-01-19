.. _quickstart:

快速入门
==========

.. currentmodule:: flask.ext.restful

是时候编写你第一个 REST API。本指南假设你对 `Flask <http://flask.pocoo.org>`_ 有一定的认识，并且已经安装了 Flask 和 Flask-RESTful。如果还没有安装的话，可以依照 :ref:`installation` 章节的步骤安装。


一个最小的 API
---------------

一个最小的 Flask-RESTful API 像这样::

    from flask import Flask
    from flask.ext import restful

    app = Flask(__name__)
    api = restful.Api(app)

    class HelloWorld(restful.Resource):
        def get(self):
            return {'hello': 'world'}

    api.add_resource(HelloWorld, '/')

    if __name__ == '__main__':
        app.run(debug=True)


把上述代码保存为 api.py 并且在你的 Python 解释器中运行它。需要注意地是我们已经启用了 `Flask 调试 <http://flask.pocoo.org/docs/quickstart/#debug-mode>`_ 模式，这种模式提供了代码的重载以及更好的错误信息。调试模式绝不能在生产环境下使用。 ::

    $ python api.py
     * Running on http://127.0.0.1:5000/

现在打开一个新的命令行窗口使用 curl 测试你的 API::

    $ curl http://127.0.0.1:5000/
    {"hello": "world"}



资源丰富的路由(Resourceful Routing)
-------------------------------------
Flask-RESTful 提供的最主要的基础就是资源(resources)。资源(Resources)是构建在 `Flask 可拔插视图 <http://flask.pocoo.org/docs/views/>`_ 之上，只要在你的资源(resource)上定义方法就能够容易地访问多个 HTTP 方法。一个待办事项应用程序的基本的 CRUD 资源看起来像这样: ::

    from flask import Flask, request
    from flask.ext.restful import Resource, Api

    app = Flask(__name__)
    api = Api(app)

    todos = {}

    class TodoSimple(Resource):
        def get(self, todo_id):
            return {todo_id: todos[todo_id]}

        def put(self, todo_id):
            todos[todo_id] = request.form['data']
            return {todo_id: todos[todo_id]}

    api.add_resource(TodoSimple, '/<string:todo_id>')

    if __name__ == '__main__':
        app.run(debug=True)

你可以尝试这样: ::

    $ curl http://localhost:5000/todo1 -d "data=Remember the milk" -X PUT
    {"todo1": "Remember the milk"}
    $ curl http://localhost:5000/todo1
    {"todo1": "Remember the milk"}
    $ curl http://localhost:5000/todo2 -d "data=Change my brakepads" -X PUT
    {"todo2": "Change my brakepads"}
    $ curl http://localhost:5000/todo2
    {"todo2": "Change my brakepads"}


或者如果你安装了 requests 库的话，可以从 python shell 中运行::

     >>> from requests import put, get
     >>> put('http://localhost:5000/todo1', data={'data': 'Remember the milk'}).json()
     {u'todo1': u'Remember the milk'}
     >>> get('http://localhost:5000/todo1').json()
     {u'todo1': u'Remember the milk'}
     >>> put('http://localhost:5000/todo2', data={'data': 'Change my brakepads'}).json()
     {u'todo2': u'Change my brakepads'}
     >>> get('http://localhost:5000/todo2').json()
     {u'todo2': u'Change my brakepads'}

Flask-RESTful 支持视图方法多种类型的返回值。同 Flask 一样，你可以返回任一迭代器，它将会被转换成一个包含原始 Flask 响应对象的响应。Flask-RESTful 也支持使用多个返回值来设置响应代码和响应头，如下所示: ::

    class Todo1(Resource):
        def get(self):
            # Default to 200 OK
            return {'task': 'Hello world'}

    class Todo2(Resource):
        def get(self):
            # Set the response code to 201
            return {'task': 'Hello world'}, 201

    class Todo3(Resource):
        def get(self):
            # Set the response code to 201 and return custom headers
            return {'task': 'Hello world'}, 201, {'Etag': 'some-opaque-string'}


端点(Endpoints)
----------------

很多时候在一个 API 中，你的资源可以通过多个 URLs 访问。你可以把多个 URLs 传给 Api 对象的 :py:meth:`Api.add_resource` 方法。每一个 URL 都能访问到你的 :py:class:`Resource` ::

    api.add_resource(HelloWorld,
        '/',
        '/hello')

你也可以
You can also match parts of the path as variables to your resource methods. ::

    api.add_resource(Todo,
        '/todo/<int:todo_id>', endpoint='todo_ep')

参数解析
----------------

While Flask provides easy access to request data (i.e. querystring or POST
form encoded data), it's still a pain to validate form data. Flask-RESTful
has built-in support for request data validation using a library similar to
`argparse <http://docs.python.org/dev/library/argparse.html>`_. ::

    from flask.ext.restful import reqparse

    parser = reqparse.RequestParser()
    parser.add_argument('rate', type=int, help='Rate to charge for this resource')
    args = parser.parse_args()


Note that unlike the argparse module,
:py:meth:`reqparse.RequestParser.parse_args` returns a Python dictionary
instead of a custom data structure.

Using the :py:class:`reqparse` module also gives you sane error messages for
free. If an argument fails to pass validation, Flask-RESTful will respond with
a 400 Bad Request and a response highlighting the error. ::

    $ curl -d 'rate=foo' http://127.0.0.1:5000/
    {'status': 400, 'message': 'foo cannot be converted to int'}


The :py:class:`inputs` module provides a number of included common conversion
functions such as :py:meth:`inputs.date` and :py:meth:`inputs.url`.

Calling ``parse_args`` with ``strict=True`` ensures that an error is thrown if
the request includes arguments your parser does not define.

    args = parser.parse_args(strict=True)

数据格式化
---------------

By default, all fields in your return iterable will be rendered as-is. While
this works great when you're just dealing with Python data structures,
it can become very frustrating when working with objects. To solve this
problem, Flask-RESTful provides the :py:class:`fields` module and the
:py:meth:`marshal_with` decorator. Similar to the Django ORM and WTForm, you
use the fields module to describe the structure of your response. ::

    from collections import OrderedDict
    from flask.ext.restful import fields, marshal_with

    resource_fields = {
        'task':   fields.String,
        'uri':    fields.Url('todo_ep')
    }

    class TodoDao(object):
        def __init__(self, todo_id, task):
            self.todo_id = todo_id
            self.task = task

            # This field will not be sent in the response
            self.status = 'active'

    class Todo(Resource):
        @marshal_with(resource_fields)
        def get(self, **kwargs):
            return TodoDao(todo_id='my_todo', task='Remember the milk')

The above example takes a python object and prepares it to be serialized. The
:py:meth:`marshal_with` decorator will apply the transformation described by
``resource_fields``. The only field extracted from the object is ``task``. The
:py:class:`fields.Url` field is a special field that takes an endpoint name
and generates a URL for that endpoint in the response. Many of the field types
you need are already included. See the :py:class:`fields` guide for a complete
list.

完整的例子
------------

Save this example in api.py ::

    from flask import Flask
    from flask.ext.restful import reqparse, abort, Api, Resource

    app = Flask(__name__)
    api = Api(app)

    TODOS = {
        'todo1': {'task': 'build an API'},
        'todo2': {'task': '?????'},
        'todo3': {'task': 'profit!'},
    }


    def abort_if_todo_doesnt_exist(todo_id):
        if todo_id not in TODOS:
            abort(404, message="Todo {} doesn't exist".format(todo_id))

    parser = reqparse.RequestParser()
    parser.add_argument('task', type=str)


    # Todo
    #   show a single todo item and lets you delete them
    class Todo(Resource):
        def get(self, todo_id):
            abort_if_todo_doesnt_exist(todo_id)
            return TODOS[todo_id]

        def delete(self, todo_id):
            abort_if_todo_doesnt_exist(todo_id)
            del TODOS[todo_id]
            return '', 204

        def put(self, todo_id):
            args = parser.parse_args()
            task = {'task': args['task']}
            TODOS[todo_id] = task
            return task, 201


    # TodoList
    #   shows a list of all todos, and lets you POST to add new tasks
    class TodoList(Resource):
        def get(self):
            return TODOS

        def post(self):
            args = parser.parse_args()
            todo_id = int(max(TODOS.keys()).lstrip('todo')) + 1
            todo_id = 'todo%i' % todo_id
            TODOS[todo_id] = {'task': args['task']}
            return TODOS[todo_id], 201

    ##
    ## Actually setup the Api resource routing here
    ##
    api.add_resource(TodoList, '/todos')
    api.add_resource(Todo, '/todos/<todo_id>')


    if __name__ == '__main__':
        app.run(debug=True)


Example usage ::

    $ python api.py
     * Running on http://127.0.0.1:5000/
     * Restarting with reloader

GET the list ::

    $ curl http://localhost:5000/todos
    {"todo1": {"task": "build an API"}, "todo3": {"task": "profit!"}, "todo2": {"task": "?????"}}

GET a single task ::

    $ curl http://localhost:5000/todos/todo3
    {"task": "profit!"}

DELETE a task ::

    $ curl http://localhost:5000/todos/todo2 -X DELETE -v

    > DELETE /todos/todo2 HTTP/1.1
    > User-Agent: curl/7.19.7 (universal-apple-darwin10.0) libcurl/7.19.7 OpenSSL/0.9.8l zlib/1.2.3
    > Host: localhost:5000
    > Accept: */*
    >
    * HTTP 1.0, assume close after body
    < HTTP/1.0 204 NO CONTENT
    < Content-Type: application/json
    < Content-Length: 0
    < Server: Werkzeug/0.8.3 Python/2.7.2
    < Date: Mon, 01 Oct 2012 22:10:32 GMT

Add a new task ::

    $ curl http://localhost:5000/todos -d "task=something new" -X POST -v

    > POST /todos HTTP/1.1
    > User-Agent: curl/7.19.7 (universal-apple-darwin10.0) libcurl/7.19.7 OpenSSL/0.9.8l zlib/1.2.3
    > Host: localhost:5000
    > Accept: */*
    > Content-Length: 18
    > Content-Type: application/x-www-form-urlencoded
    >
    * HTTP 1.0, assume close after body
    < HTTP/1.0 201 CREATED
    < Content-Type: application/json
    < Content-Length: 25
    < Server: Werkzeug/0.8.3 Python/2.7.2
    < Date: Mon, 01 Oct 2012 22:12:58 GMT
    <
    * Closing connection #0
    {"task": "something new"}

Update a task ::

    $ curl http://localhost:5000/todos/todo3 -d "task=something different" -X PUT -v

    > PUT /todos/todo3 HTTP/1.1
    > Host: localhost:5000
    > Accept: */*
    > Content-Length: 20
    > Content-Type: application/x-www-form-urlencoded
    >
    * HTTP 1.0, assume close after body
    < HTTP/1.0 201 CREATED
    < Content-Type: application/json
    < Content-Length: 27
    < Server: Werkzeug/0.8.3 Python/2.7.3
    < Date: Mon, 01 Oct 2012 22:13:00 GMT
    <
    * Closing connection #0
    {"task": "something different"}

