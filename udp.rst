.. include:: replace.txt
.. highlight:: cpp

UDP model in ns-3
-----------------

This chapter describes the UDP model available in |ns3|.

Generic support for UDP
***********************

|ns3| supports a native implementation of UDP. It provides a connectionless,
unreliable datagram packet service. Packets may be reordered or duplicated before
they arrive. UDP calculates and checks checksums to catch transmission errors.

This implementation inherits from a few common header classes in the ``src/network``
directory, so that user code can swap out implementations with minimal changes to
the scripts.

Here are the important abstract base classes:

* class :cpp:class:`UdpSocket`: This is defined in:
  ``src/internet/model/udp-socket.{cc,h}``
  This is an abstract base class of all UDP sockets. This class exists solely for
  hosting ``UdpSocket`` attributes that can be reused across different implementations,
  and for declaring UDP-specific multicast API.

* class :cpp:class:`UdpSocketFactory`: This is used by the layer-4 protocol
  instance to create UDP sockets.

ns-3 UDP
********

This is an implementation of the User Datagram Protocol described in RFC 768.
UDP uses a simple connectionless communication model with a minimum of protocol
mechanism. The implementation provides checksums for data integrity, and port
numbers for addressing different functions at the source and destination of the
datagram. It has no handshaking dialogues, and thus exposes the user's program to
any unreliability of the underlying network. There is no guarantee of delivery,
ordering, or duplicate protection.

Usage
+++++

In many cases, usage of UDP is set at the application layer by telling
the |ns3| application which kind of socket factory to use.

Using the helper functions defined in ``src/applications/helper`` and
``src/network/helper``, here is how one would create a UDP receiver::

  // Create a packet sink on the receiver
  uint16_t port = 50000;
  Address sinkLocalAddress(InetSocketAddress (Ipv4Address::GetAny (), port));
  PacketSinkHelper sinkHelper ("ns3::UdpSocketFactory", sinkLocalAddress);
  ApplicationContainer sinkApp = sinkHelper.Install (serverNode);
  sinkApp.Start (Seconds (1.0));
  sinkApp.Stop (Seconds (10.0));

Similarly, the below snippet configures OnOffApplication traffic source to use
UDP::

  // Create the OnOff applications to send UDP to the server
  OnOffHelper clientHelper ("ns3::UdpSocketFactory", Address ());
  clientHelper.SetAttribute ("Remote", remoteAddress);
  ApplicationContainer clientApps = (clientHelper.Install (clientNode);
  clientApps.Start (Seconds (1.0));
  clientApps.Stop (Seconds (10.0));

For users who wish to have a pointer to the actual socket (so that
socket operations like Bind(), setting socket options, etc. can be
done on a per-socket basis), UDP sockets can be created by using the 
``Socket::CreateSocket()`` method. 

Once a UDPsocket is created, we do not need an explicit connection setup before
sending and receiving data. Being a connectionless protocol, all we need to do
is to create a socket and bind it a known port. For a client, simply create a
socket and start sending data.The bind() call allows an application to specify
a port number and an address on the local machine. It Allocates a local IPv4
endpoint for this socket.

At the end of the protocol, the connection is closed using the Socket::close()
which closes the socket.It returns a 0 on success and -1 on failure. 

Please note that applications usually create the sockets they use automatically,
and so is not straightforward to connect direcly to them using pointers. Please
refer to the source code of your preferred application to discover how and when
it creates the socket.


UDP Socket interaction and interface with Application layer
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
In the following, there is an analysis on the public interface of the UDP socket,
and how it can be used to interact with the socket itself.
 
**Socket APIs for UDP connections**:

*Connect()*
  Set the remote endpoint, and try to connect to it. If the connection to address
  succeeds, the UDP will call the callback *ConnectionSucceeded*.

*Bind()*
  Bind the socket to an address, or to a general endpoint. A general endpoint
  is an endpoint with an ephemeral port allocation (that is, a random port
  allocation) on the 0.0.0.0 IP address. For instance, in current applications,
  data senders usually binds automatically after a *Connect()* over a random
  port. Consequently, the connection will start from this random port towards
  the well-defined port of the receiver. The IP 0.0.0.0 is then translated by
  lower layers into the real IP of the device.

*Bind6()*
  Same as *Bind()*, but for IPv6.

*BindToNetDevice()*
  Bind the socket to the specified NetDevice, creating a general endpoint.

*ShutdownSend()*
  Signal a termination of send, or in other words revents data from being added
  to the buffer.

*Recv()*
  Grab data from the UDP socket and forwards it to the application layer. If no
  data is present (i.e. *m_deliveryQueue.empty ()* returns 0) an empty packet is
  returned.

*RecvFrom()*
  Same as *Recv()*, but with the source address as parameter.

*SendTo()*
  The *SendTo()* API is the UDP counterpart of the TCP API *Send()*. It
  additionally specifies the address to which the message is to be sent
  because no prior connection is established in UDP communication. It returns
  the number of bytes sent or -1 in case of failure.

*Close()*
  The close API is used to close a socket and terminate the connection. This
  results in freeing all the kernel data structures previously allocated.

-----------------------------------------

**Public callbacks**

These callbacks are called by the UDP socket to notify the application of
interesting events. We will refer to these with the protected name used in
socket.h, but we will provide the API function to set the pointers to these
callback as well.

*NotifyConnectionSucceeded*: *SetConnectCallback*, 1st argument
  Called when the *Connect()* succeeds and the connection is established.

*NotifyConnectionFailed*: *SetConnectCallback*, 2nd argument
  Called in *Connect()* when the connection establishment fails.

*NotifyDataSent*: *SetDataSentCallback*
  The Socket notifies the application that some bytes has been transmitted on
  the IP level. These bytes could still be lost in the node (traffic control
  layer) or in the network.

*NotifySend*: *SetSendCallback*
  Invoked to get the space available in the tx buffer when a packet (that can
  carry data) is sent.

*NotifyDataRecv*: *SetRecvCallback*
  Called when the socket receive a packet (that can carry data) in the receiver
  buffer.


Validation
++++++++++

The following test cases are found in the ``src/internet/test/udp-test.cc`` file.

* **UdpSocketImplTest:** Checks data received via UDP Socket over IPv4.
* **UdpSocketLoopbackTest:** Checks data received via UDP Socket Loopback over IPv4.
* **Udp6SocketImplTest :** Checks data received via UDP Socket over IPv6.
* **Udp6SocketLoopbackTest :** Checks data received via UDP Socket Loopback over IPv6 Test.


Limitations
+++++++++++


* UDP_CORK, MSG_DONTROUTE, path MTU discovery control (e.g. IP_MTU_DISCOVER)
  are not presently part of this implementation.
* MTU handling is also weak in ns-3 for the moment; it is best to send
  datagrams that do not exceed 1500 byte MTU (e.g. 1472 byte UDP datagrams).
* Not all socket API callbacks are supported.
