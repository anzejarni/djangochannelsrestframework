Custom actions
==============


Consumer that aren't bound to a Model.
--------------------------------------

We may want a consumer for handling certain actions that are not related to any Django model. For example,
fetching data from an external api service, using ``httpx`` or another async request library.


.. code-block:: python

    # consumers.py
    from djangochannelsrestframework.decorators import action
    from djangochannelsrestframework.consumers import AsyncAPIConsumer
    from rest_framework import status

    class MyConsumer(AsyncAPIConsumer):

        @action()
        async def an_async_action(self, some=None, **kwargs):
            # do something async
            return {'response with': 'some message'}, status.HTTP_RESPONSE_OK

        @action()
        def a_sync_action(self, pk=None, **kwargs):
            # do something sync
            return {'response with': 'some message'}, status.HTTP_RESPONSE_OK

Consumer that is bound to a Model.
----------------------------------

Inheriting from ``GenericAsyncAPIConsumer``, we have access to methods like ``get_queryset`` and ``get_object``.
This way we can perform operations in our Django models though custom actions.

.. code-block:: python

    # serializers.py
    from rest_framework import serializers
    from django.contrib.auth.models import User

    class UserSerializer(serializers.ModelSerializer):
        
        class Meta:
            model = User
            fields = ["id", "username", "email", "password"]
            extra_kwargs = {'password': {'write_only': True}}
        
        def create(self, validated_data):
            user = User(
                email=validated_data['email'],
                username=validated_data['username']
            )
            user.set_password(validated_data['password'])
            user.save()
            return user

.. code-block:: python

    # consumers.py
    from django.contrib.auth.models import User
    from .serializers import UserSerializer
    from djangochannelsrestframework.generics import GenericAsyncAPIConsumer
    from djangochannelsrestframework.decorators import action

    class UserConsumer(GenericAsyncAPIConsumer):
        queryset = User.objects.all()
        serializer_class = UserSerializer

        @action()
        async def send_email(self, pk=None, to=None, **kwargs):
            user = await database_sync_to_async(self.get_object)(pk=pk)
            # ... do some stuff
            # remember to wrap all db actions in `database_sync_to_async`
            return {}, 200  # return the content and the response code.

        @action()  # if the method is not async it is already wrapped in `database_sync_to_async`
        def publish(self, pk=None, **kwargs):
            user = self.get_object(pk=pk)
            # ...
            return {'pk': pk}, 200

