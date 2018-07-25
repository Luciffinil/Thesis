```Javascript
Ext.Ajax.request({
    url: contextPath + '/security/acl/login',
    method: 'POST',
    params: {
      'userCode': userCode,
      'password': password
    },
    success: function(response){
        var respText = JSON.parse(response.responseText);
        var code = respText.code;
        if(code=='40031' || code=='40041') {

          var sessionId = respText.customSessionId;
          Ext.util.Cookies.set('Custom-SessionId', sessionId);	//TODO set period, path

          Ext.create('IFSP.index.view.ChangePassword').show();
        } else if(code != '0') {
          var msg = Ext.ComponentQuery.query("label[name=msg]", form)[0];
          msg.update(respText.msg);
        } else {
          var sessionId = respText.customSessionId;
          Ext.util.Cookies.set('Custom-SessionId', sessionId);	//TODO set period, path
          Ext.create('IFSP.index.view.RoleSelection').show();
        }
    }
});
```
