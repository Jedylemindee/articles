# How to test a Flask API and its common extensions with Pytest.

If you plan to use Flask to build a json API (for instance a REST API), 
there are high chances you end up using the following add-ons :
* SQLAlchemy to work with SQL databases (with Flask-SQLAlchemy)
* Flask-Migrate to work with database migrations 
* Marshmallow to validate incoming data, and serialize the requests outcomes

If you don't use these packages, I strongly recommend you to have a look at them and not reinvent the wheel.

This article will give you some insights about how to efficiently test a Flask application alongside these 
extensions. In particular, we will create a set of pytest *fixtures* to cover most of the possible test cases.

### Prerequisites

I assume that you already have some knowledge of pytest and particularly the notion of *fixtures*. Basically, a fixture
defines a resource that a test can use. If you are not familiar with that, I suggest you to read 
[pytest's documentation](https://docs.pytest.org/en/latest/fixture.html) 

I also assume that your Flask project follows the application factory pattern. It means you should have a set of extensions declared in one (or multiple) module(s) :

```python
# extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_marshmallow import Marshmallow
from flask_migrate import Migrate

db = SQLAlchemy()
migrate = Migrate()
marsh = Marshmallow()
```
    
Then you should define an application factory in another module : 

```python
# app.py
from flask import Flask
from .extensions import db, migrate, ma

def create_app(config):
    app = Flask(__name__)
    app.config.update(**config)
    db.init_app(app)
    migrate.init_app(app)
    marsh.init_app(app)
    # import sqlalchemy models
    from .models import *    
    return app
```        

If this project structure sounds like greek to you, you should definitely check 
Flask's doc about [app factory patterns](https://flask.palletsprojects.com/en/1.1.x/patterns/appfactories/).

## First fixtures : setup the environment

First, we need some fixtures for our Flask app:

```python
# conftest.py
import pytest
from .app import create_app

TEST_CONFIG = {
    'TESTING': True,
    'DEBUG': True
}

@pytest.fixture(scope="session")
def app():    
    app = create_app(config=TEST_CONFIG)    
    with app.app_context():
        yield app

@pytest.fixture(scope="function")
def test_client(app):
    return app.test_client()
```

This way, you can easily test your routes :

```python
# test_routes.py
def test_hello_world(test_client):
    res = test_client.post("/hello_world", data={})
    assert res.status_code == 201
    assert "hello" in res.json
```

Moreover, if you need some special authentication, you can easily add other client fixtures by modifying
headers in app.test_client, e.g a *user_client* fixture that will automatically log you as a user.


## Setting up database fixtures

For the next section you need to have a database for your tests.
This database can be any local database that is safe to use for testing, and is therefore assumed to be empty.

First, let's change our test config to include our database

```python
# conftest.py    
TEST_CONFIG = {
    'TESTING': True,
    'DEBUG': True,
    'SQLALCHEMY_DATABASE_URI': 'my_test_database_uri'
}
```

Then let's add a *db* fixture:

```python
# conftest.py
from .extensions import db as _db
from flask_migrate import upgrade as flask_migrate_upgrade

@pytest.fixture(scope="session")
def db(app, request):
    """Session-wide test database."""

    def teardown():
        _db.drop_all()
    _db.app = app

    flask_migrate_upgrade(directory="migrations")
    request.addfinalizer(teardown)
    return _db
```
        
This fixture has a session-wide scope, which means it will be computed only once for all the tests.
What it basically does is it binds our 'db' object to our app fixture, then executes all the migrations
with *flask_migrate_upgrade()*. This line could have been replaced by a simple *_db.create_all()*, 
but this call to Flask-Migrate allows to test migrations on the fly.

The next step is to create a *session* fixture. This fixture will be used in each of our tests are ensure 
they are isolated from each other. 

```python
@pytest.fixture(scope="function")
def session(db, request):
    db.session.begin_nested()

    def commit():
        db.session.flush()
    
    # patch commit method
    old_commit = db.session.commit
    db.session.commit = commit

    def teardown():
        db.session.rollback()
        db.session.close()
        db.session.commit = old_commit

    request.addfinalizer(teardown)
    return db.session
```

Each time this fixture is computed (i.e during each test), the following things happen : 
* Before the test, the database is empty
* The call to *begin_nested()* creates a nested transaction that expires only when a *session.rollback()* 
or a *session.commit()* is called. However, we do not want to allow commits to be triggered, since it would actually push 
changes in the database and therefore impact other tests. In order to prevent this, we simply
patch the *session.commit* method with *session.flush*, which is transaction-safe. This way we can call
*db.session.commit* in our project without worrying about tests.
* When the test ends, a rollback is issued, clearing the database for further tests.


## Testing models and serializers with FactoryBoy

Now let's be more practical with an example.

Say we have a model we want to test:

```python
# models.py

from .extensions import db

class MyModel(db.Model):
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.Text, nullable=True)
    is_active = db.Boolean(db.Boolean, nullable=False, default=True)
    
    def save(self):
        db.session.add(self)
    
    def __str__(self):
        return self.name
        
 ### Usage: 
 # instance = MyModel(name="...", description="...", is_active=True)
 # db.session.add(instance)
 # db.session.commit()
```    
            
And a corresponding serializer created with Marshmallow-SQLAlchemy:

```python
# serializers.py
from .extensions import marsh as ma

class MyModelSerializer(ma.ModelSchema):
    class Meta:
        model = MyModel
        
### Deserialization 
# db_instance = MyModelSerializer().load(dictionnary_data)
# <MyModel "foo">
### Serialization
# dictionnary_data = MyModelSerializer().dump(db_instance)
# {"name": "foo", "description": "bar", "is_active": True}
```

With the previous configuration, you could test your model and serializer this way :

```python
# test_mymodel.py
from .models import MyModel
from .serializers import MyModelSerializer

def test_mymodel_str_method(session):
    instance = MyModel(name="Test", description="Ipso Lorim", is_active=True)
    assert str(instance) == instance.name

def test_mymodel_save_method(session):
    instance = MyModel(name="Test", description="Lorem Ipsum...", is_active=True)
    instance.save()
    session.flush()
    assert MyModel.query.filter_by(name="Test").first() == instance
    
def test_mymodel_serializer():
    # we do not need the session fixture since we don't need to save anything to the db
    form_data = {
        "name": "Test3",
        "description": "Ipsum Lorim",
        "is_active": False
    }
    serializer = MyModelSerializer().load(form_data)
    assert serializer.errors = {}
    assert serializer.data.name == form_data["name"]
    assert serializer.data.description == form_data["description"]
```
        
However, this method is not flawless. First, it would be nice to test our serializer
with our model, so that if we need to add another attribute to the model, we don't
need to add it in 'form_data' too. Moreover, we always have to specify some dummy data
when instanciating a model, such as a dummy 'name' attribute, whereas we just need to launch
our tests with 'some dummy instance of MyModel', whatever its attributes are.


#### Better testing serializers with a Dictionnary Mixin

The first problem can be easily resolved with an 'as_dict' method to add to our models :

```python
# mixins.py

class DictMixin:

    def as_dict(self, exclude=None):
        if exclude is None:
            exclude = []
        table = self.__table__
        return {
            field: getattr(self, field)
            for field in table.columns.keys()
            if field not in exclude
        }
        
# models.py

from .mixins import DictMixin

class MyModel(db.Model, DictMixin);
    # ...
```
        
Then we can rewrite our last test :

```python
# test_models.py    
def test_mymodel_serializer():
    form_data = MyModel(name="Test", description="Test desc").as_dict()
    serializer = MyModelSerializer().load(form_data)
    assert serializer.errors = {}
    assert serializer.data.name == form_data["name"]
    assert serializer.data.description == form_data["description"]
```        
        
#### Lightening our tests with FactoryBoy

Let's summarize our second problem. Instead of writing :

```python
instance = MyModel(name="Test", description="Lorem Ipsum...")
```
  
We would like something like :

```python
instance = SomeInstanceOfMyModelWithDummyParams() 
```
    
With different attributes each time we call it. We could also want a batch of different instances to make some scenario tests,
etc. 

Fortunately, we have a solution for that, it's called FactoryBoy :tada: let's discover it
with our example. 

A Factory for MyModel would look like this:

```python
# factories.py
import factory.fuzzy

class MyModelFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = MyModel
        sqlalchemy_session = db.session
        
    is_active = True # we want this attribute to be False by default
    name = factory.faker.Faker("name") # each call will generate a fake name
    description = factory.fuzzy.FuzzyText() # same with another method
```

And we can use it in our tests     
    
```python
# test_models.py
from .factories import MyModelFactory

def test_mymodel_str_method(session):
    # this instance will have new attributes each time this test is executed
    instance = MyModelFactory()
    session.flush()
    assert str(instance) == instance.name

def test_mymodel_save_method(session):
    # calling build() will create the python object without saving it to the database
    instance = MyModelFactory.build()
    instance.save()
    session.flush()
    assert MyModel.query.filter_by(instance.name).first() == instance

def test_mymodel_serializer():
    form_data = MyModelFactory.build().as_dict()
    serializer = MyModelSerializer().load(form_date)
    assert serializer.errors = {}
    assert serializer.data.name == form_data["name"]
    assert serializer.data.description == form_data["description"]
```

And that's it !
