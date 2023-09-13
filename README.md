# Django REST Framework Tutorial

1. Create a virtual environment(optional): 
```bash
# Create a virtual environment named 'venv'
python -m venv venv

# Activate the virtual environment
source venv/Scripts/activate
```

2. Install Django and Django REST Framework
```bash
pip install django
pip install djangorestframework
```

3. Create a django project and navigate to its directory
```bash
# Create django project
django-admin startproject todo_project

# Navigate to the django project created
cd todo_project
```

4. Create a Django app for your API
```bash
# Create django app
django-admin startapp todo_app
```

5. Run initial migrations of the built-in model
```bash
# Run initial migrations
py manage.py migrate
```

6. Add rest_framework and todo_app to the INSTALLED_APPS (list) inside todo_project/todo_project/settings.py
```py
# todo_project/todo_project/settings.py
INSTALLED_APPs = [
    # ...other default installed_apps
    'rest_framework',
    'todo_app'
]
```

7. Create a serializers.py and urls.py inside the todo_app directory
```bash
├── todo_project
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
├── db.sqlite3
├── manage.py
└── todo_app
    ├── admin.py
    ├── serializers.py # Create this file
    ├── __init__.py
    ├── models.py
    ├── urls.py # Create this file
    └── views.py
```

8. Navigate to the urls.py of the project (todo_project/urls.py)
```py
# import the urls from the todo_app and name it as todo_urls
from django.contrib import admin
from django.urls import path, include
from todo_app import urls as todo_urls

# include rest_framework and todo_urls
urlpatterns = [
    path('admin/', admin.site.urls),
    path('api-auth/', include('rest_framework.urls')),
    path('todos/', include(todo_urls)),
]
```

9. Create a superuser
```bash
py manage.py createsuperuser
# Username: user
# Email: brianmaysebastian@gmail.com
# Password: bri123
```

10. Create a model for the todo_app
```py 
# todo_app/models.py
from django.db import models
from django.contrib.auth.models import User

# Create your models here.
class Todo(models.Model):
    task = models.CharField(max_length=180)
    completed = models.BooleanField(default=False, blank=True)
    timestamp = models.DateTimeField(auto_now_add=True, auto_now=False, blank=True)
    updated = models.DateTimeField(auto_now=True, blank=True)
    user = models.ForeignKey(User, on_delete=models.CASCADE, blank=True, null=True)

    def __str__(self) -> str:
        return self.task
```

11. Migrate the model to the database
```bash 
py manage.py makemigrations
py manage.py migrate
```

12. Serialize the model(todo_app/serializers.py)

**note:Django REST framework uses ModelSerializer to convert any model to serialized JSON object
```py
from rest_framework import serializers
from .models import Todo

class TodoSerializer(serializers.ModelSerializer):
    class Meta:
        model = Todo
        fields = ["task", "completed", "timestamp", "updated", "user"]
```

13. List view

**note:
- GET: listing all to-dos of a given requested user
- POST: creating a new to-do

The GET() method first fetches all the objects from the model by filtering with the requested user ID. Then, it serializes from the model object to a JSON serialized object. Next, it returns the response with serialized data and status as 200_OK.

The POST() method fetches the requested data and adds the requested user ID in the data dictionary. Next, it creates a serialized object and saves the object if it’s valid. If valid, it returns the serializer.data, which is a newly created object with status as 201_CREATED. Otherwise, it returns the serializer.errors with status as 400_BAD_REQUEST.
```py
# todo_project/todo_app/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework import permissions
from .models import Todo
from .serializers import TodoSerializer

# Create your views here.
class TodoListApiView(APIView):
    # add permission to check if user is authenticated
    permission_classes = [permissions.IsAuthenticated]

    # 1. List all
    def get(self, request, *args, **kwargs):
        '''
        List all the todo items for given requested user
        '''

        todos = Todo.objects.filter(user = request.user.id)
        serializer = TodoSerializer(todos, many=True)
        return Response(serializer.data, status=status.HTTP_200_OK)

    # 2. Create
    def post(self, request, *args, **kwargs):    
        '''
        Create the Todo with given Todo data
        '''

        data = {
            'task': request.data.get('task'),
            'completed': request.data.get('completed'),
            'user': request.user.id
        }

        serializer = TodoSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

14. Create an endpoint for the class-based view above
```py
# todo_project/todo_app/urls.py
from django.urls import path
from .views import(
    TodoListApiView,
)

urlpatterns = [
    path('api/', TodoListApiView.as_view()),
]
```

14. Migrate the model and run the server
```bash
py manage.py makemigrations
py manage.py migrate
py manage.py runserver
```

15. Test the api on your browser or on postman
```bash
1. Using browser 
# login first to get an authentication to access the todo list
http://127.0.0.1:8000/api-auth/login/

# GET - to get all the records of to-dos 
http://127.0.0.1:8000/todos/api/

# POST - to create a new record
# type this on the text field to create a new record
{
    "task": "New Task",
    "completed": false
}

2. Using Postman
# navigate to the Authorization tab and input your username and password
# Type: Basic Auth
Username = user
Password = bri123

# GET - to get all the records of to-dos
http://127.0.0.1:8000/todos/api/

# POST - to create a new record
# navigate to the Body tab and select "raw" as the type and type
{
    "task": "New Task",
    "completed": false
}
```

16. Detail view - lets create the second endpoint todos/api/<int: todo_id>

The GET() method first fetches the object with the ID todo_id and user as request user from the to-do model. If the requested object is not available, it returns the response with the status as 400_BAD_REQUEST. Otherwise, it serializes the model object to a JSON serialized object and returns the response with serializer.data and status as 200_OK.

The PUT() method fetches the to-do object if it is available in the database, updates its data with requested data, and saves the updated data in the database.

The DELETE() method fetches the to-do object if is available in the database, deletes it, and responds with a response.
```py
# update the todo_app/urls.py code and add this new class
class TodoDetailApiView(APIView):
    # add permission to check if user is authenticated
    permission_classes = [permissions.IsAuthenticated]

    def get_object(self, todo_id, user_id):
        '''
        Helper method to get the object with given todo_id and user_id
        '''

        try:
            return Todo.objects.get(id=todo_id, user=user_id)
        except Todo.DoesNotExist:
            return None
        
        
    # 3. Retrieve
    def get(self, request, todo_id, *args, **kwargs):
        '''
        Retrieves the Todo with given todo_id
        '''

        todo_instance = self.get_object(todo_id, request.user.id)
        if not todo_instance:
            return Response(
                {'response': 'Object with todo_id does not exist'}, 
                status=status.HTTP_400_BAD_REQUEST
            )

        serializer = TodoSerializer(todo_instance)

        return Response(serializer.data, status=status.HTTP_200_OK)


    # 4. Update 
    def put(self, request, todo_id, *args, **kwargs):
        '''
        Updates the todo item if given todo_id exists
        '''

        todo_instance = self.get_object(todo_id, request.user.id)
        if not todo_instance:
            return Response(
                {'response': 'Object with todo_if does not exist'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        data = {
            'task': request.data.get('task'),
            'completed': request.data.get('completed'),
            'user': request.user.id
        }

        serializer = TodoSerializer(instance=todo_instance, data=data, partial=True)

        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_200_OK)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    

    # 5. Delete
    def delete(self, request, todo_id, *args, **kwargs):
        '''
        Deletes the todo item if given todo_id exists
        '''

        todo_instance = self.get_object(todo_id, request.user.id)
        if not todo_instance:
            return Response(
                {'response': 'Object with todo_id does not exist'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        todo_instance.delete()

        return Response(
            {'response': 'Object deleted'}, 
            status=status.HTTP_200_OK
        )
```

17. Update the todo_app/urls.py for the second endpoint
```py
# todo_project/todo_app/urls.py
from django.urls import path
from .views import(
    TodoListApiView,
    TodoDetailApiView,
)

urlpatterns = [
    path('api/', TodoListApiView.as_view()),
    path('api/<int:todo_id>/', TodoDetailApiView.as_view()),
]
```