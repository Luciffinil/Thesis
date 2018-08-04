```Javascript
Ext.Ajax.request({
    url: contextPath + '/acl/login',
    method: 'POST',
    params: {
      'userCode': userCode,
      'password': CryptoJS.DES.encrypt(pwd).toString()
    },
    success: function(response){
        var respText = JSON.parse(response.responseText);
        var code = respText.code;
        if(code=='40031' || code=='40041') {
            // 初次登录或密码过期，进行修改密码
        } else if(code != '0') {
            // 其他异常情况，显示异常详情
        } else {
            var sessionId = respText.customSessionId;
			Ext.util.Cookies.set('Custom-SessionId', sessionId);	// 设定登录用户的SessionId
            // 正常登录，进行选择角色
        }
    }
});
```

```Java
User user = sqlSessionTemplate.selectOne("security.user.find", userCode);
```

缓存模块
```java
CacheUpdaterHolder cacheUpdaterHolder = CacheUpdaterHolder.getInstance();
cacheUpdaterHolder.put("CACHE_UPDATER_USER", new UserCacheUpdater());
```
缓存容器
```java
public class CacheHolder extends ConcurrentHashMap<String, Object>{
	private static final CacheHolder cacheHolder = new CacheHolder();
	
	private CacheHolder() {}
	
	public static CacheHolder getInstance() {
		return cacheHolder;
	}
	
	public <T> T getCache(String key) {
		return (T)super.get(key);
	}
	
	public <T> T get(String cacheKey, String bizKey) {
		Map<String, Object> bizCache = (Map<String, Object>) super.get(cacheKey);
		return (T)bizCache.get(bizKey);
	}
}
```

接口 CacheUpdater.java
```java
public interface CacheUpdater {
	void update(Object record);
}
```

抽象类  AbstractCacheUpdater.java
```java
public abstract class AbstractCacheUpdater implements CacheUpdater{
	@Override
	public void update(Object object) {
		Map<String, Object> option = (Map<String, Object>)object;
		String cacheOp = (String)option.get("CACHE_OP");
		if(StringUtils.equals(cacheOp, "CACHE_OP_PUT")) {
			put(option);
		}else if(StringUtils.equals(cacheOp, "CACHE_OP_REMOVE")) {
			remove(option);
		}else if(StringUtils.equals(cacheOp, "CACHE_OP_RELOAD")) {
			reload(option);
		}else {
			throw new RuntimeException("未知CACHE_OP:" + cacheOp);
		}
	}
	
	abstract public void put(Map<String, Object> option);
	abstract public void remove(Map<String, Object> option);
	abstract public void reload(Map<String, Object> option);
}
```

实现类
```java
public class UserCacheUpdater extends AbstractCacheUpdater {
	@Override
	public void put(Map<String, Object> option) {
		...
	}

	@Override
	public void remove(Map<String, Object> option) {
		...
	}

	@Override
	public void reload(Map<String, Object> option) {
		...
	}
	
}

```

缓存更新容器
```java
public class CacheUpdaterHolder extends ConcurrentHashMap<String, CacheUpdater>{
	private static final CacheUpdaterHolder cacheUpdaterHolder = new CacheUpdaterHolder();	
	
	private CacheUpdaterHolder() {}
	
	public static CacheUpdaterHolder getInstance() {
		return cacheUpdaterHolder;
	}
	
	public static void reload() {
		ExecutorHolder.getCachedExecutorService().execute(new Runnable() {
			@Override
			public void run() {
				CacheUpdater cacheUpdater = null;	
				try {
					Map<String, Object> param = new HashMap<>();
					param.put("CACHE_OP", "CACHE_OP_RELOAD");				
					CacheUpdaterHolder cacheUpdaterHolder = CacheUpdaterHolder.getInstance();
					for (String key:cacheUpdaterHolder.keySet()) {
						cacheUpdater = cacheUpdaterHolder.get(key);
						cacheUpdater.update(param);
					}
				} catch (Throwable t) {
					logger.error("Cache reload failed for " + cacheUpdater, t);
				}
			}
		});
	}
	
	public static void reload(final String cacheUpdaterKey) {
		Map<String, Object> param = new HashMap<>();
		param.put("CACHE_OP", "CACHE_OP_RELOAD");
		
		CacheUpdaterHolder cacheUpdaterHolder = CacheUpdaterHolder.getInstance();
		CacheUpdater cacheUpdater = cacheUpdaterHolder.get(cacheUpdaterKey);
		if(cacheUpdater != null) {
			cacheUpdater.update(param);
		}else {
			logger.warn("Can not find CacheUpdater, cacheUpdaterKey:" + cacheUpdaterKey);
		}
	}
}
```


Socket
线程池
```java
public class ExecutorHolder {
	private static Logger logger = LogManager.getLogger(ExecutorHolder.class);

	private static ExecutorService executorService = Executors.newFixedThreadPool(30);
	private static ExecutorService cachedExecutorService = Executors.newCachedThreadPool();
	
	public static ExecutorService getExecutorService() {
		return executorService;
	}
	
	public static ExecutorService getCachedExecutorService() {
		return cachedExecutorService;
	}
	
	public static void destroyExecutorService() {
		executorService.shutdown();
		executorService = null;
		logger.info("线程池已关闭!");
		
		cachedExecutorService.shutdown();
		cachedExecutorService = null;
		logger.info("缓存线程池已关闭!");
	}
}
```

初始化
```java
private void initSocketListener() {
	try {
		server = new SocketServer();
		ExecutorHolder.getCachedExecutorService().execute(server);
	} catch (IOException e) {
		logger.error("服务端启动错误!",e);
	}
}
```

socket服务端
```java
public class SocketServer implements Runnable{	
	private ServerSocket serverSocket;
	private boolean isStart = true;
	
	public SocketServer() throws IOException {
		serverSocket = new ServerSocket(10429);
	}

	@Override
	public void run() {
		logger.info("服务端已启动,正在监听端口： " + serverSocket.getLocalPort());
		while(isStart) {
			Socket server = null;
			try {
				server = serverSocket.accept();
				logger.info("(" + server.getInetAddress().getHostAddress() + ")已连接本机...");
				ExecutorHolder.getCachedExecutorService().execute(new SocketMessageReceiver(server));
			} catch (SocketException e) {
				logger.warn("Socket服务端已关闭");
			} catch (IOException e) {
				logger.error("客户端连接失败!", e);
			}
		}
	}
	
	public void stop() {
		isStart  = false;
		try {
			serverSocket.close();
		} catch (IOException e) {
			logger.error("服务端关闭监听端口失败!", e);
		}
	}
}
```

Socket消息接收器
```java
public class SocketMessageReceiver implements Runnable{
	private Socket server;

	public SocketMessageReceiver(Socket server) {
		this.server = server;
	}
	
	@Override
	public void run() {
		ObjectInputStream ois = null;
		ObjectOutputStream oos = null;
		try {
			ois = new ObjectInputStream(socket.getInputStream());
			String type = (String) ois.readObject();
			SocketServerHandlerHolder socketServerHandlerHolder = SocketServerHandlerHolder.getInstance();
			SocketServerHandler socketServerHandler = socketServerHandlerHolder.get(type);
			socketServerHandler.handle(ois);
		} catch (Exception e) {
			logger.error("消息转发异常!", e);
		} 
	}
	...
}
```

Socket消息处理接口
```java
public interface SocketServerHandler{
	void handle(ObjectInputStream ois);
}
```

Socket缓存消息处理抽象类
```java
public abstract class CacheSocketServerHandler implements SocketServerHandler{
	@Override
	public void handle(ObjectInputStream ois) {
		try {
			Object object = readObject(ois);
			Map<String, Object> option = (Map<String, Object>)object;
			String type = (String) option.get("CACHE_UPDATER");
			CacheUpdater cacheUpdater = CacheUpdaterHolder.getInstance().get(type);
			if(cacheUpdater == null) {
				throw new RuntimeException("No cache updater find by type:" + type);
			}
			cacheUpdater.update(object);
		} catch (Exception e) {
			logger.error("缓存更新失败!", e);
		}
	}
	
	public abstract Object readObject(ObjectInputStream ois) throws Exception;
}
```

Socket缓存消息实现类
```java
public class BaseCacheSocketServerHandler extends CacheSocketServerHandler{

	@Override
	public Object readObject(ObjectInputStream ois) throws Exception {
		return ois.readObject();
	}
	
}
```

Socket消息处理实现类容器
```java
public class SocketServerHandlerHolder extends ConcurrentHashMap<String, SocketServerHandler>{
	private static final SocketServerHandlerHolder socketServerHandlerHolder = new SocketServerHandlerHolder();
	
	private SocketServerHandlerHolder() {}
	
	public static SocketServerHandlerHolder getInstance() {
		return socketServerHandlerHolder;
	}
}
```

Controller发送消息
```java
Map<String, Object> option = new HashMap<>();
option.put("CACHE_OP", cacheOp);
option.put("CACHE_UPDATER", updater);
option.put("CACHE_OBJECT", record);
SocketMessage socketMessage = new SocketMessage("SOCKET_SERVER_HANDLER_BASE_CACHE", option);
SocketUtil.broadcast(socketMessage);
```

```java
public SocketMessage(String type, Object object) {
	this.type = type;
	this.object = object;
}
```

```java
public class SocketUtil {
	public static void broadcast(SocketMessage socketMessage) {
		List<Host> hosts = sqlSessionTemplate.selectList("base.host.find");
		if (CollectionUtils.isNotEmpty(hosts)) {
			for (Host host : hosts) {
				ExecutorHolder.getCachedExecutorService().execute(new SocketMessageSender(host, socketMessage));
			}
		} 
	}
}	
```

Socket消息发送器
```java
public class SocketMessageSender implements Runnable{
	private Host host;
	private SocketMessage message;
	
	public SocketMessageSender(Host host, SocketMessage message) {
		this.host = host;
		this.message = message;
	}

	@Override
	public void run() {
		Socket client = null;
		try {
			client = new Socket(host.getHost(), 10429);
		} catch (IOException e) {
			logger.warn("请求连接" + host.getHost() + "失败!");
			return;
		}
		sendSocketMessage(client, message);
	}
	
	private Socket connect(String host, int port) throws IOException {
		return new Socket(host, port);
	}

	private void sendSocketMessage(Socket client, SocketMessage socketMessage){
		ObjectOutputStream oos = null;
		try {
			oos = new ObjectOutputStream(socket.getOutputStream());
			oos.writeObject(socketMessage.getType());
			oos.writeObject(socketMessage.getObject());
			client.close();
		} catch (Exception e) {
			logger.error("客户端发送消息失败!",e);
		}
	}
}
```
