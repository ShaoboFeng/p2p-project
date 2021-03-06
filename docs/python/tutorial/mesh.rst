Mesh Socket
~~~~~~~~~~~

Basic Usage
-----------

The mesh schema is used as an alert and messaging network. Its primary purpose is to ensure message delivery to every participant in the network.

To connect to a mesh network, use the :py:class:`~py2p.mesh.mesh_socket` object. This is instantiated as follows:

.. code-block:: python

    >>> from py2p import mesh
    >>> sock = mesh.mesh_socket('0.0.0.0', 4444)

Using ``'0.0.0.0'`` will automatically grab your LAN address. Using an outbound internet connection requires a little more work. First, ensure that you have a port forward set up (NAT busting is not in the scope of this project). Then specify your outward address as follows:

.. code-block:: python

    >>> from py2p import mesh
    >>> sock = mesh.mesh_socket('0.0.0.0', 4444, out_addr=('35.24.77.21', 44565))

In addition, SSL encryption can be enabled if `cryptography <https://cryptography.io/en/latest/installation/>`_ is installed. This works by specifying a custom :py:class:`~py2p.base.protocol` object, like so:

.. code-block:: python

    >>> from py2p import mesh, base
    >>> sock = mesh.mesh_socket('0.0.0.0', 4444, prot=base.protocol('mesh', 'SSL'))

Eventually that will be the default, but while things are being tested it will default to plaintext. If `cryptography <https://cryptography.io/en/latest/installation/>`_ is not installed, this will generate an :py:exc:`ImportError`

Specifying a different protocol object will ensure that the node *only* can connect to people who share its object structure. So if someone has ``'mesh2'`` instead of ``'mesh'``, it will fail to connect. You can see the current default by looking at :py:data:`py2p.mesh.default_protocol`.

Unfortunately, this failure is currently silent. Because this is asynchronous in nature, raising an :py:exc:`Exception` is not possible. Because of this, it's good to perform the following check after connecting:

.. code-block:: python

    >>> from py2p import mesh
    >>> import time
    >>> sock = mesh.mesh_socket('0.0.0.0', 4444)
    >>> sock.connect('192.168.1.14', 4567)
    >>> time.sleep(1)
    >>> assert sock.routing_table

To send a message, use the :py:meth:`~py2p.mesh.mesh_socket.send` method. Each argument supplied will correspond to a packet that the peer receives. In addition, there is a keyed argument you can use. ``flag`` will specify how other nodes relay this. These flags are defined in :py:class:`py2p.base.flags`. ``broadcast`` will indicate that other nodes are supposed to relay it. ``whisper`` will indicate that your peers are *not* supposed to relay it.

.. code-block:: python

    >>> sock.send('this is', 'a test')

Receiving is a bit simpler. When the :py:meth:`~py2p.mesh.mesh_socket.recv` method is called, it returns a :py:class:`~py2p.base.message` object (or ``None`` if there are no messages). This has a number of methods outlined which you can find by clicking its name. Most notably, you can get the packets in a message with :py:attr:`.message.packets`, and reply directly with :py:meth:`.message.reply`.

.. code-block:: python

    >>> sock.send('Did you get this?')  # A peer then replies
    >>> msg = sock.recv()
    >>> print(msg)
    message(type=b'whisper', packets=[b'yes', b'I did'], sender=b'6VnYj9LjoVLTvU3uPhy4nxm6yv2wEvhaRtGHeV9wwFngWGGqKAzuZ8jK6gFuvq737V')
    >>> print(msg.packets)
    [b'whisper', b'yes', b'I did']
    >>> for msg in sock.recv(10):
    ...     msg.reply("Replying to a list")

Advanced Usage
--------------

In addition to this, you can register a custom handler for incoming messages. This is appended to the end of the default handlers. These handlers are then called in a similar way to Javascripts ``Array.some()``. In other words, when a handler returns something true-like, it stops calling handlers.

When writing your handler, keep in mind that you are only passed a :py:class:`~py2p.base.message` object and a :py:class:`~py2p.mesh.mesh_connection`. Fortunately you can get access to everything you need from these objects.

.. code-block:: python

    >>> from py2p import mesh, base
    >>> def register_1(msg, handler):   # Takes in a message and mesh_connection
    ...     packets = msg.packets       # This grabs a copy of the packets. Slightly more efficient to store this once.
    ...     if packets[1] == b'test':   # This is the condition we want to act under
    ...         msg.reply(b"success")   # This is the response we should give
    ...         return True             # This tells the daemon we took an action, so it should stop calling handlers
    ...
    >>> def register_2(msg, handler):   # This is a slightly different syntax
    ...     packets = msg.packets
    ...     if packets[1] == b'test':
    ...         handler.send(base.flags.whisper, base.flags.whisper, b"success")  # One could instead reply to the node who relayed the message
    ...         return True
    ...
    >>> sock = mesh.mesh_socket('0.0.0.0', 4444)
    >>> sock.register_handler(register_1)  # The handler is now registered

If this does not take two arguments, :py:meth:`~py2p.base.base_socket.register_handler` will raise a :py:exc:`ValueError`.

To help debug these services, you can specify a :py:attr:`~py2p.base.base_socket.debug_level` in the constructor. Using a value of 5, you can see when it enters into each handler, as well as every message which goes in or out.
