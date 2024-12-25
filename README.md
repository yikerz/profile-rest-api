### Objective

In this project, I am using Django to create an API for user profile creation and profile feed item management.

### Create Vagrant Development Server

Vagrant is a tool to recreate a virtual machine (VM) under the same conditions every time. It ensures consistency in the environment required to run the software.

1. Initialize the `ubuntu/bionic64` box with Vagrant. ([cmd 1](#commands))
2. Replace the default `Vagrantfile` with the one provided in the project.
3. Start the VM. ([cmd 2](#commands))
4. Connect to the VM via SSH. ([cmd 3](#commands))

### Creaet Virtual Environment in VM

In the Vagrant VM, the directory `/vagrant` is synced with the project folder on your host machine.

1. Create a new virtual environment at `~/env`. ([cmd 4](#commands))
2. Activate the virtual environment. ([cmd 5](#commands))
3. Create a `requirements.txt` file in `/vagrant` with the following dependencies

- `django==2.2`
- `djangorestframework==3.9.2`

4. Install the dependencies. ([cmd 6](#commands))

### Create Django App

1. Create a new Django project called `profiles_project`. ([cmd 7](#commands))
2. Create a new Django app called `profiles_api`. ([cmd 8](#commands))
3. Add the following apps to the `INSTALLED_APPS` section in `profiles_project/settings.py`

- `rest_framework`
- `rest_framework.authtoken`
- `profiles_api`

### Model, Serializer, and View

The API functionality is built around three main components: **Model**, **Serializer**, and **View**.

#### Model

Represents the database table schema. Define all required fields and their types. Django models include methods for database operations (`models.Model`). For custom database actions, override default methods or define a custom Manager.

#### View

Handles user interaction. Key components of a view are the **endpoint** and **request type**. Each view corresponds to an endpoint, defined in `urls.py`. A single endpoint can handle multiple request types, with their behavior defined in view functions or classes.

#### Serializer

Bridges data transfer between the view and database. It converts user input (e.g., JSON) into a format suitable for the database and vice versa. The serializer typically includes a **Meta** class to specify:

1. The corresponding model.
2. Fields to map.
3. Additional configurations (`extra_kwargs`)

#### How These Components Work Together

1. **Request**: The user sends a request (e.g., JSON data) to an endpoint.
2. **View**: The view processes the request and determines the logic.
3. **Serializer**:

- For input: Converts JSON into Python data and validates it.
- For output: Converts Python data or model instances into JSON.

4. **Model**: Performs database actions like querying or saving data.

### Model, Serializer and View for `UserProfile`

#### `UserProfile` Model

1. Define the `UserProfile` class:
   In `profiles_api/models.py`, create the `UserProfile` class that inherits from `AbstractBaseUser` and `PermissionsMixin`. Using `AbstractBaseUser` allows you to fully customize the user model (e.g., changing the primary identifier to email instead of a username). `PermissionsMixin` provides essential fields and methods for handling user permissions, such as `is_superuser`.

   - Fields: `email`, `name`, `is_active` and `is_staff`
   - Assign `UserProfileManager` to `objects` (This connects the custom manager to the model for handling user creation; explained below)
   - Set `USERNAME_FIELD` (`email`) and `REQUIRED_FIELDS` (`name`)

2. Create `UserProfileManager`
   In `profiles_api/models.py`, define a custom model manager to override the default user creation methods. The default manager doesn't handle custom fields like email or name. By overriding `create_user`, you can define the behavior for creating users and ensure necessary fields are provided.

   - Create function `create_user`
     - Ensures `email` is provided
     - Normalizes the email to a standard format
     - Creates a user instance and set its password securely using `set_password()`
     - Save and return user
   - Create function `create_superuser`
     - Calls `create_user` to create a base user instance.
     - Sets `is_superuser` and `is_staff` to True.
     - Save and return user

3. Configure `AUTH_USER_MODEL` as `profiles_api.UserProfile` in `profiles_project/settings.py`
4. Register `models.UserProfile` model to the admit site

#### `UserProfile` Serializer

1. Define a serializer class `UserProfileSerializer` inheriting from `serializers.ModelSerializer` to handle user data in `profiles_api/serializers.py`.
2. In the `Meta` class:

   - Link the serializer to the `UserProfile` model.
   - Specify which fields (`id`, `email`, `name`, `password`) should be included.
   - Configure `extra_kwargs` for the `password` field to make it `write-only` and display it as a masked input in forms.

3. Implement a `create` method:

   - Use the custom manager method `create_user` to handle user creation and password hashing.

4. Implement an `update` method:

   - Check if the `password` is in the incoming data.
   - If present, remove it from the data and securely update the user's password.
   - Update the remaining fields and save the instance.

#### `UserProfile` ViewSet

1. Define a `ViewSet` class `UserProfileViewSet` that inherits from `viewsets.ModelViewSet` to provide CRUD functionality in `profiles_api/views.py`.
2. Set the `serializer_class` to link the viewset to the serializer `UserProfileSerializer`.
3. Define the `queryset` to operate on all user profile objects.

#### Register the ViewSet in URLs

1. Create `urls.py` in `profiles_api`
   - Use a router (e.g., `DefaultRouter`) to simplify endpoint management.
   - Register the `UserProfileViewSet` with the router, specifying a base name (e.g., `profile`).
   - Include the router's URLs in the main urlpatterns to expose the endpoints (e.g., `/profile/`, `/profile/<id>/`).
2. Include `profiles_api/urls.py` in `profiles_project/urls.py`.

#### Authentications and Permissions

1. Create a permission class `UpdateOwnProfile` by inheriting from `permissions.BasePermission` in a newly created file `profiles_api/permissions.py`

   - Implement the `has_object_permission` method, which returns `True` for safe HTTP methods (e.g., GET, HEAD, OPTIONS) by checking against `permissions.SAFE_METHODS`.
   - For other methods (e.g., PUT, PATCH, DELETE), ensure that the authenticated user's id matches the object's id.

2. Add `TokenAuthentication` to the `authentication_classes` in the `UserProfileViewSet`.
3. Add `UpdateOwnProfile` to the `permission_classes` in the `UserProfileViewSet`.

#### Search Profile Feature

1. Add `filters.SearchFilter` to the `filter_backends` in the `UserProfileViewSet`.
2. Define `search_fields` to specify which fields can be searched (e.g., `name`, `email`).

### Create Login API

1. Create a class named `UserLoginApiView` by inheriting from `ObtainAuthToken` in `profiles_api/views.py`.
2. Add `renderer_classes` to the `UserLoginApiView` class to be `api_settings.DEFAULT_RENDERER_CLASSES` (ensures that the login endpoint works with web-based interfaces and the admin site).
3. Add a new path in `profiles_api/urls.py` for the `UserLoginApiView` class.

### Create Profile Feed API

#### Define the `ProfileFeedItem` Model

1. Create a `ProfileFeedItem` class in `models.py` with fields `user_profile`, `status_text` and `created_on`.
2. Override the `__str__` method to return a meaningful representation of the model (e.g., `status_text`).
3. In `admin.py`, register the `ProfileFeedItem` model to make it manageable through the Django admin interface.

#### Define the Serializer

1. Create a `ProfileFeedItemSerializer` class in `serializers.py` by inheriting from `serializers.ModelSerializer`.
2. Define `Meta` class
   - Specify `ProfileFeedItem` as the model.
   - Include necessary fields such as `id`, `user_profile`, `status_text`, and `created_on`.
   - Set `user_profile` in `extra_kwargs` as `read_only` since it will be automatically assigned to the authenticated user.

#### Define the ViewSet

1. Create a `UserProfileFeedViewSet` class in `views.py` by inheriting from `viewsets.ModelViewSet`.
   - Define `serializer_class` as `ProfileFeedItemSerializer`.
   - Set `queryset` to `ProfileFeedItem.objects.all()`.
   - Set `authentication_classes` to include `TokenAuthentication`.
2. Override `perform_create` method by using the serializer to save the object and assigning `user_profile` to the currently authenticated user.

#### Register ViewSet to Router

1. In `urls.py`, register the `UserProfileFeedViewSet` to the router with an appropriate base name (e.g., `feed`).

#### Feed API Permissions

1. In `permissions.py`, create a class `UpdateOwnStatus` by inheriting from `permissions.BasePermission`.
   - Return `True` if request method is `SAFE_METHODS`.
   - If not `SAFE_METHODS`, return boolean `obj.user_profile.id == request.user.id`.
2. In `UserProfileFeedViewSet`, update the `permission_classes` to include:
   - `UpdateOwnStatus` for object-level permissions.
   - `IsAuthenticated` to restrict access to authenticated users only.

### Commands

1. `vagrant init ubuntu/bionic64`
2. `vagrant up`
3. `vagrant ssh`
4. `python -m venv ~/env`
5. `source ~/env/bin/activate`
6. `pip install -r requirements.txt`
7. `django-admin startproject profiles_project`
8. `python manage.py startapp profiles_api`
