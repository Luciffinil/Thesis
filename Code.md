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
缓存容器
```java
public class CacheHolder extends ConcurrentHashMap<String, Object>{
	private static final CacheHolder cacheHolder = new CacheHolder();
	
	private CacheHolder() {}
	
	public static CacheHolder getInstance() {
		return cacheHolder;
	}
	
	public <T> T getCache(Object key) {
		return (T)super.get(key);
	}
	
	public <T> T get(String cacheKey, String bizKey) {
		Map<String, Object> bizCache = (Map<String, Object>) super.get(cacheKey);
		if(bizCache == null) {
			throw new RuntimeException("No cache object find from " + CacheHolder.class.getSimpleName() + " by cacheKey:" + cacheKey);
		}
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

	}

	@Override
	public void remove(Map<String, Object> option) {

	}

	@Override
	public void reload(Map<String, Object> option) {

	}
	
}

```


