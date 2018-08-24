# Django DBRouting

## Defining your databases
In settings.py file (I am using mysql)
```
DATABASES = {
    'default':{},
    'db1': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'Database_1',
        'USER': 'root',
        'PASSWORD': 'root',
    },
    'db2':{
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'Database_2',
        'USER': 'root',
        'PASSWORD': 'root',
    }
}
```

## Synchronizing your databases
The migrate management command operates on one database at a time. By default, it operates on the default database, but by providing the --database option, you can tell it to synchronize a different database. So, to synchronize all models onto all databases.
```
$ ./manage.py migrate --database=db1
$ ./manage.py migrate --database=db2
```
## Manually selecting a database for a QuerySet
In the urls.py file (sample URL patterns)
```
http://example.com/login/
http://test.example/com/login
```
```
urlpatterns = [
    url(r'^/login/$', views.xzy, {'database': 'db1'}),
    url(r'^com/login/$', views.xzy, {'database': 'db2'})

]
```

You can select the database for a QuerySet at any point in the QuerySet “chain.” Just call using() on the QuerySet to get another QuerySet that uses the specified database.

using() takes a single argument: the alias of the database on which you want to run the query. For example:
In xzy(request, database), you can do the following:
```
records = Model.objects.using(database).all()
```

Selecting a database for save()
my_object.save(using='legacy_users')

## Automatic database routing
In settings.py file, we need to define DATABASE_ROUTERS. So that we can automatically select database
```DATABASE_ROUTERS = ['path.to.DBRouter']```

We can select database in two ways

First Way to select database is by defining which database to use for which host.
```
DB_Mapping = {
    "HOST1": "DB1",
    "HOST2": "DB2",
}
```

Create a middleware to get the database name from host name. [docs](https://docs.djangoproject.com/en/1.8/topics/http/middleware/)
```
from django.conf import settings
from django.utils.deprecation import MiddlewareMixin

class RouterMiddleware(MiddlewareMixin):
    def process_view( self, request, view_func, args, kwargs ):
        HOST_NAME = request.get_host()

    def process_request(self, request):
        HOST_NAME = request.get_host()
```

Create a database router to auto select.
```
class DatabaseRouter(object):
    def _default_db( self ):
        if HOST_NAME in settings.DATABASES.keys():
            return DB_Mapping[HOST_NAME]
        else:
            return 'default'

    def db_for_read( self, model, **hints ):
        return self._default_db()

    def db_for_write( self, model, **hints ):
        return self._default_db()
```

Second Way is to define separate apps for each database.

```
class AppRouter:
    """
    A router to control all database operations on models in the
    respective apps application.
    """

    def db_for_read(self, model, **hints):
        """
        Attempts to read app1 models go to app1_db.
        """
        if model._meta.app_label == 'app1':
            return 'app1_db'
        return None

    def db_for_write(self, model, **hints):
        """
        Attempts to write app1 models go to app1_db.
        """
        if model._meta.app_label == 'app1':
            return 'app1_db'
        return None

    def allow_relation(self, obj1, obj2, **hints):
        """
        Allow relations if a model in the app1 app is involved.
        """
        if obj1._meta.app_label == 'app1' or \
           obj2._meta.app_label == 'app1':
           return True
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """
        Make sure the app only appears in the 'app1_db'
        database.
        """
        if app_label == 'app1':
            return db == 'app1_db'
        return None
```

You can readup more [here](https://docs.djangoproject.com/en/dev/topics/db/multi-db/)
