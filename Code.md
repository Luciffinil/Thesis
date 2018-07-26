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

