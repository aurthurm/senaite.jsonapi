UPDATE
------

Running this test from the buildout directory:

    bin/test test_doctests -t update


Test Setup
~~~~~~~~~~

Needed Imports:

    >>> import json
    >>> import transaction
    >>> import urllib
    >>> from plone.app.testing import setRoles
    >>> from plone.app.testing import TEST_USER_ID
    >>> from plone.app.testing import TEST_USER_PASSWORD

    >>> from bika.lims import api

Functional Helpers:

    >>> def get(url):
    ...     browser.open("{}/{}".format(api_url, url))
    ...     return browser.contents

    >>> def post(url, data):
    ...     url = "{}/{}".format(api_url, url)
    ...     browser.post(url, urllib.urlencode(data, doseq=True))
    ...     return browser.contents

    >>> def get_item_object(response):
    ...     assert("items" in response)
    ...     response = json.loads(response)
    ...     items = response.get("items")
    ...     assert(len(items)==1)
    ...     item = response.get("items")[0]
    ...     assert("uid" in item)
    ...     return api.get_object(item["uid"])

    >>> def create(data):
    ...     response = post("create", data)
    ...     return get_item_object(response)

Variables:

    >>> portal = self.portal
    >>> portal_url = portal.absolute_url()
    >>> api_url = "{}/@@API/senaite/v1".format(portal_url)
    >>> setup = api.get_setup()
    >>> browser = self.getBrowser()
    >>> setRoles(portal, TEST_USER_ID, ["LabManager", "Manager"])
    >>> transaction.commit()

Initialize the instance with some objects for testing:

    >>> clients = api.get_portal().clients
    >>> data = {"portal_type": "Client",
    ...         "parent_path": api.get_path(clients),
    ...         "title": "Chicken corp",
    ...         "ClientID": "CC"}
    >>> client1 = create(data)

    >>> data = {"portal_type": "Client",
    ...         "parent_path": api.get_path(clients),
    ...         "title": "Beef Corp",
    ...         "ClientID": "BC"}
    >>> client2 = create(data)

    >>> data = {"portal_type": "Client",
    ...         "parent_path": api.get_path(clients),
    ...         "title": "Octopus Corp",
    ...         "ClientID": "OC"}
    >>> client3 = create(data)


Update by resource and uid
~~~~~~~~~~~~~~~~~~~~~~~~~~

We can update an object by providing the resource and the uid of the object:

    >>> client_uid = api.get_uid(client1)
    >>> data = {"ClientID": "CC1"}
    >>> response = post("client/update/{}".format(client_uid), data)
    >>> obj = get_item_object(response)
    >>> obj.getClientID()
    'CC1'

Update by uid without resource
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Even easier, we can update with only the uid:

    >>> data = {"ClientID": "CC2"}
    >>> response = post("update/{}".format(client_uid), data)
    >>> obj = get_item_object(response)
    >>> obj.getClientID()
    'CC2'

Update via post only
~~~~~~~~~~~~~~~~~~~~

When updating by resource (without an UID explicitly set), the system expects a
the data to passed via POST to contain the item to be updated.

The object to be updated can be send in the HTTP POST body by using the `uid`:

    >>> data = {"uid": client_uid,
    ...         "ClientID": "CC3"}
    >>> response = post("update", data)
    >>> obj = get_item_object(response)
    >>> obj.getClientID()
    'CC3'

By using the `path`, as the physical path of the object:

    >>> data = {"path": api.get_path(client1),
    ...         "ClientID": "CC4"}
    >>> response = post("update", data)
    >>> obj = get_item_object(response)
    >>> obj.getClientID()
    'CC4'

Or by using the `id` of the object together with `parent_path`, as the physical
path of the container object:

    >>> data = {"id": api.get_id(client1),
    ...         "parent_path": api.get_path(clients),
    ...         "ClientID": "CC5"}
    >>> response = post("update", data)
    >>> obj = get_item_object(response)
    >>> obj.getClientID()
    'CC5'

Do a transition
~~~~~~~~~~~~~~~

We can transition the objects by using the keyord `transition` in the data sent
via POST:

    >>> api.is_active(client1)
    True
    >>> data = {"uid": api.get_uid(client1),
    ...         "transition": "deactivate"}
    >>> response = post("update", data)
    >>> obj = get_item_object(response)
    >>> api.is_active(obj)
    False

We can update and transition at same time:

    >>> data = {"uid": api.get_uid(client1),
    ...         "ClientID": "CC6",
    ...         "transition": "activate"}
    >>> response = post("update", data)
    >>> obj = get_item_object(response)
    >>> api.is_active(obj)
    True
    >>> obj.getClientID()
    'CC6'

Update restrictions
~~~~~~~~~~~~~~~~~~~

We get a 401 error if we try to update an object from inside portal root:

    >>> data = {"title": "My clients folder",
    ...         "uid": api.get_uid(clients),}
    >>> post("update", data)
    Traceback (most recent call last):
    [...]
    HTTPError: HTTP Error 401: Unauthorized

We get a 401 error if we try to update an object from inside setup folder:

    >>> cats_uid = api.get_uid(portal.setup.analysiscategories)
    >>> data = {"title": "My Analysis Categories folder",
    ...         "uid": cats_uid,}
    >>> post("update", data)
    Traceback (most recent call last):
    [...]
    HTTPError: HTTP Error 401: Unauthorized

We cannot update the `id` of an object:

    >>> original_id = api.get_id(client1)
    >>> data = {"id": "client-123123",
    ...         "uid": client_uid }
    >>> response = post("update", data)
    >>> obj = get_item_object(response)
    >>> api.get_id(obj) == original_id
    True
