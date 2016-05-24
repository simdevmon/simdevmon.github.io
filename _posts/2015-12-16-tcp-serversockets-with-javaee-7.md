---
title: TCP server sockets with JavaEE 7
updated: 2015-09-06 18:00
---

Usually JavaEE applications provide a REST interface to communicate with other systems. 
Some legacy systems do not support REST and I had the requirement to integrate TCP ServerSockets. 
It is a bad idea to start a server socket directly in an EJB or to even start a new thread, because this can cause malfunctions in the application server (since they are not managed).
With *ManagedExecutorService* it is possible to get a managed instance.

The socket pool is invoked only once as a singleton and creates all socket listeners on startup. 
In this example I created two listeners manually.

```java
@Startup
@Singleton
public class SocketPool
{

    @Inject
    private Logger logger;

    @Inject
    private Instance<SocketListener> sockets;

    @Resource
    private ManagedExecutorService mes;

    @PostConstruct
    public void init()
    {
        startSocket(5556);
        startSocket(5557);
    }

    private void startSocket(int port)
    {
        SocketListener listener = this.sockets.get();
        listener.setPort(port);
        this.mes.submit(listener);
    }

    @PreDestroy
    public void stop()
    {
        for (SocketListener socket : this.sockets)
        {
            try
            {
                socket.stop();
            }
            catch (IOException ex)
            {
                this.logger.error(ex.getMessage(), ex);
            }
        }
    }
}
```

The socket listener binds a single port and forwards incoming connections to the socket connection.

```java
public class SocketListener implements Runnable
{

    @Inject
    private Logger logger;

    @Inject
    private Instance<SocketConnection> handles;

    @Resource
    private ManagedExecutorService mes;

    private int port;

    private ServerSocket serverSocket;

    @Override
    public void run()
    {
        try
        {
            this.logger.info("Start server socket at {}", this.port);
            this.serverSocket = new ServerSocket(this.port);
            while (true)
            {
                SocketConnection connection = this.handles.get();
                connection.setSocket(this.serverSocket.accept());
                this.mes.submit(connection);
            }
        }
        catch (IOException ex)
        {
            this.logger.error(ex.getMessage(), ex);
        }
    }

    public void stop() throws IOException
    {
        this.logger.info("Stop server socket at {}", this.port);
        this.serverSocket.close();
    }

    public int getPort()
    {
        return port;
    }

    public void setPort(int port)
    {
        this.port = port;
    }
}
```

The socket connection does the actual communication with the client. 

```java
public class SocketConnection implements Runnable
{

    @Inject
    private Logger logger;

    private Socket socket;

    public Socket getSocket()
    {
        return socket;
    }

    public void setSocket(Socket socket)
    {
        this.socket = socket;
    }

    @Override
    public void run()
    {
        this.logger.info("Accepting client at {}", this.socket.getPort());
        try
        {
            try (OutputStream output = socket.getOutputStream())
            {
                output.write(
                    ("HTTP/1.1 200 OK\n\nCurrentTime: " + LocalDateTime.now())
                    .getBytes()
                );
            }
        }
        catch (IOException ex)
        {
            this.logger.error(ex.getMessage(), ex);
        }
    }
}
```
