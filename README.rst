RIPE Atlas Cousteau |Build Status| |Code Health|
================================================

A python wrapper around RIPE ATLAS API.

Installation
------------

You can install by either cloning the repo and run the following inside
the repo:

.. code:: bash

    $ python setup.py install

or via pip using:

.. code:: bash

    $ pip install ripe.atlas.cousteau

Usage
-----

Creating a Measurement
~~~~~~~~~~~~~~~~~~~~~~

Creating two new RIPE Atlas UDMs is as easy as:

.. code:: python

    from datetime import datetime
    from ripe.atlas.cousteau import (
      Ping,
      Traceroute,
      AtlasSource,
      AtlasCreateRequest
    )

    ATLAS_API_KEY = ""

    ping = Ping(af=4, target="www.google.gr", description="testing new wrapper")

    traceroute = Traceroute(
        af=4,
        target="www.ripe.net",
        description="testing",
        protocol="ICMP",
    )

    source = AtlasSource(type="area", value="WW", requested=5)

    atlas_request = AtlasCreateRequest(
        start_time=datetime.utcnow(),
        key=ATLAS_API_KEY,
        measurements=[ping, traceroute],
        sources=[source]
    )

    (is_success, response) = atlas_request.create()

Keep in mind that this library is trying to comply with what is stated
in the `documentation pages`_. This means that if you try to create a
request that is missing a field stated as required, the library won’t go
ahead and do the HTTP query. On the contrary, it will raise an exception
with some info in it.

.. _documentation pages: https://atlas.ripe.net/docs/measurement-creation-api/

Changing Measurement Sources
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Similarly if you want to change (remove in the following example) probes
for an existing measurement you can do:

.. code:: python

    from ripe.atlas.cousteau import AtlasChangeSource, AtlasChangeRequest

    ATLAS_MODIFY_API_KEY = ""

    source = AtlasChangeSource(
        value="1,2,3",
        requested=3,
        action="remove"
    )

    atlas_request = AtlasChangeRequest(
        key=ATLAS_MODIFY_API_KEY,
        msm_id=1000001,
        sources=[source]
    )

    (is_success, response) = atlas_request.create()

Same applies if you want to add a list of probes, you just have to
change “action” key to “add” as stated on the `docs`_.

.. _docs: https://atlas.ripe.net/docs/rest/#participation-request

Stopping Measurement
~~~~~~~~~~~~~~~~~~~~

You can stop a measurement with:

.. code:: python

    from ripe.atlas.cousteau import AtlasStopRequest

    ATLAS_STOP_API_KEY = ""

    atlas_request = AtlasStopRequest(msm_id=1000001, key=ATLAS_STOP_API_KEY)

    (is_success, response) = atlas_request.create()

In order to be able to successfully create most of the above you need to
create an `API key`_.

.. _API key: https://atlas.ripe.net/docs/keys/

Make Any API Get Requests
~~~~~~~~~~~~~~~~~~~~~~~~~

If you know the url path you can make any request easily towards ATLAS
API.

.. code:: python

    msm_id = 2016892
    url_path = (
        "api/v1/measurement/{0}/?fields=participant_count&"
        "format=json"
    ).format(msm_id)
    request = AtlasRequest(**{"url_path": url_path})
    result = namedtuple('Result', 'success response')
    (is_success, response) = request.get()
    if not is_success:
        return False

    return result.response["participant_count"]

Fetch Results
~~~~~~~~~~~~~

You can fetch results for any measurements using cousteau. In the
following example we are getting all results for measurement ID 2016892
and for probe IDs 1,2,3,4 between 2015-05-19 and 2015-05-20. Times can
be python datetime objects, Unix timestamps or string representations of
dates.

.. code:: python

    from ripe.atlas.cousteau import AtlasResultsRequest

    kwargs = {
        "msm_id": 2016892,
        "start": datetime(2015, 05, 19)
        "stop": datetime(2015, 05, 20)
        "probe_ids": [1,2,3,4]
    }

    is_success, results = AtlasResultsRequest(**kwargs).create()

    if is_success:
        print(results)

Fetch real time results
~~~~~~~~~~~~~~~~~~~~~~~

Besides fetching results from main API it is possible to get results
though `streaming API`_.

.. code:: python

    from ripe.atlas.cousteau import AtlasStream

    def on_result_response(*args):
        """
        Function that will be called every time we receive a new result.
        Args is a tuple, so you should use args[0] to access the real message.
        """
        print args[0]

    atlas_stream = AtlasStream()
    atlas_stream.connect()
    # Measurement results
    stream_type = "result"
    # Bind function we want to run with every result message received
    atlas_stream.bind_stream(stream_type, on_result_response)
    # Subscribe to new stream for 1001 measurement results
    stream_parameters = {"msm": 1001}
    atlas_stream.start_stream(stream_type=stream_type, **stream_parameters)

    # Probe's connection status results
    stream_type = "probestatus"
    atlas_stream.bind_stream(stream_type=stream_type, on_result_response)
    stream_parameters = {"enrichProbes": True}
    atlas_stream.start_stream(stream_type=stream_type, **stream_parameters)

    # Timeout all subscriptions after 5 secs. Leave seconds empty for no timeout.
    # Make sure you have this line after you start *all* your streams
    atlas_stream.timeout(seconds=5)
    # Shut down everything
    atlas_stream.disconnect()

The available stream parameters for every stream type are described in
the `streaming results docs`_

.. _streaming API: https://atlas.ripe.net/docs/result-streaming/
.. _streaming results docs: https://atlas.ripe.net/docs/result-streaming/

Fetch Probes/Measurements Meta data
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is a helpful feature that hides all the complexity of traversing
the API using the next url each time there are more objects. It returns
a python generator that you can use to access each object.

Fetches all probes from NL with asn\_v4 3333 and with tag NAT

.. code:: python

    from ripe.atlas.cousteau import ProbeRequest

    filters = {"tags": "NAT", "country_code": "NL", "asn_v4": "3333"}
    probes = ProbeRequest(**filters)

    for probe in probes:
        print(probe["id"])

    # Print total count of found probes
    print(probes.total_count)

Fetches all specified measurements.

.. code:: python

    from ripe.atlas.cousteau import MeasurementRequest

    filters = {"status": 1}
    measurements = MeasurementRequest(**filters)

    for msm in measurements:
        print(msm["msm_id"])

    # Print total count of found measurements
    print(measurements.total_count)

Colophon
========

But why `Cousteau`_? The RIPE Atlas team decided to name all of its
modules after explorers, and this is not an exception :)

.. _Cousteau: http://en.wikipedia.org/wiki/Jacques_Cousteau
.. |Build Status| image:: https://travis-ci.org/RIPE-NCC/ripe-atlas-cousteau.png?branch=master
   :target: https://travis-ci.org/RIPE-NCC/ripe-atlas-cousteau
.. |Code Health| image:: https://landscape.io/github/RIPE-NCC/ripe-atlas-cousteau/master/landscape.png
   :target: https://landscape.io/github/RIPE-NCC/ripe-atlas-cousteau/master
