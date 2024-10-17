
## 一 refresh\_token刷新access\_token


Keycloak会话管理中，获取到accessToken和refreshToken后，基于accessToken交换用户数据或者参与KeycloakAPI的请求，当accessToken过期的时候，可使用refreshToken去交换新的accessToken和refreshToken。


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016171842904-1212444323.png)


这块根据之前的refresh\_token就得到了一个新的token对象


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016171904659-1739715429.png)


## refresh\_token和access\_token有效期配置


* refresh\_token的有效期一般比access\_token的长，这也就是通过refresh\_token来换取新的access\_token的一个前提，下面来配置一个这两个token的超时时间。
* refresh\_token超时时间refresh\_expires\_in，在realms settings中，选择tokens进行配置，对SSO Session Max进行设置


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016171944757-603663122.png)


access\_token超时时间expires\_in,在realms settings中，选择tokens进行配置，对Access Token Lifespan进行设置。


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172000797-168226680.png)


这个用户会话，对应的sessionId(session\_state)可以在浏览器cookie中找到，或者在kc管理后台的用户\-》会话中查看，这个sessionId被客户端访问，都会刷新这个“开始”时间，和“最后访问”时间，当你的access\_token过期后，`你通过refresh_token去刷新access_token时，这个“最后访问”时间也会更新`，如图


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172037151-565495828.png)



> 如果用户访问资源，在token过期，而refresh\_token(sso session max)未过期时，你可以通过refresh\_token来获取新的token，这时会有新的会话产生；但如果refresh\_token也过期时，它将跳转到登录页，从新进行认证，会话也就被删除了。


## 三 refresh\_token过期时间的配置


领域设置\-\>Tokens中，有四个选项用来控制refresh\_token的超时时间


* SSO Session Idle
* SSO Session Max
* Client Session Idle
* Client Session Max


下图的4个选项，配置是有问题的，正确的配置应该是最长时间大于空闲时间，下面配置无意义，这时有效性使用4个选项中最小的值【sso会话空闲时间，sso会话最长时间，client session Idle和client session Max】


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172128544-1289775826.png)


如果正常配置的话，当空闲时间和最长时间不相同时，`真实的refresh_token_expire时间将取决于client Session Idle的值`，如下配置


当refresh\_token到期之后到达 （ session max的时间 ） ，session就失效了，而`它并不会立即清除`，它会交给keycloak进行维护，`最长是session max时间后自动清除`，而用户如果在这个时间之前进行refresh\_token时，会提示token是不活动的，这时会话也会也被清空，表示令牌过期了，如下面两张图：


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172237596-1769032318.png)


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172250633-1473718504.png)


当session idle和session max不相同时(sso session max和client session max)，用户的会话会在sso session max到期时删除，`而sso session max是全局的`，不能在客户端单独配置，一个会话是在什么时间被系统回收，主要由以下6个参数决定，SSO Session Max和Client Sesssion Max我们设置一个即可，它在keycloak后台清理session时会以最长的为准，而当session达到session idle时间时，如果用户主动刷新token，session也会被主动删除，不会等session max时间达了再删。


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172322070-1808951507.png)


## 四 Session Idle和Session Max的作用


会话的空闲时间(Idle)，是指在多长时间之内没有使用refresh\_token进行刷新，这个会话(session\_state)就过期，无法再直接用refresh\_token去换新的token了，这时用户就需要重新回到登录页，完成新的认证；这主要针对长时间不操作的用户，kc需要让它重新完成用户名密码的确认。



> 注意：如果开启了“记住我”这个功能，因为如果开启“记住我”功能之后，`你的会话空闲时间等于“记住我空闲时间”`，你的”sso session idle”配置将失效，如果记住我配置了最大时间和空闲时间，那么token的生成和校验都将使用记住我的时间，如图keycloak14\.0\.0\.\-services里AuthenticationManage.isSessionValid的源码。



> 【`session idle在判断上有2分钟的误差`，主要考虑DC集群的数据同步，比如idle有效期5分钟，那么真正过期就是5\+2为7分钟】当到7分钟后，你获取session是否在线时，结果会返回false.


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172634571-1219904598.png)


上面代码中，isSessionValid方法会在`验证token和刷新token时`都会进行执行，我们如果希望将session idle和session max去正确使用，还需要修改kc源代码中的org.keycloak.protocol.oidc
.TokenManager.refreshAccessToken()方法中的代码,将verifyRefreshToken方法参数中的checkExpiration改成false，如图：


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172655844-1573294330.png)


最后，下图配置了access\_token有效期2分钟，refresh\_token最长30天，会话空闲为7天；配置的作用为：用户每2分钟access\_token会过期，然后用户通过refresh\_token去换新的access\_token，如果用户7天没有换token，这个会话就过期，如果会话已经产生了30天，则会话也过期，用户就会返回登录页，重新认证。


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172720185-287105828.png)


事实上，当session idle 和session max相等时，你的refresh\_token的过期时间会一直递减，从第一次申请这个refresh\_token开始，这个过期时间就固定了，它和你换新token是无关系的；但如果session idle和 session max不相等时（max\>idle），你每换新token，你的新换的refresh\_token的过期时间都从头开始算，它的大小等于session idle的大小，这也是你在session idle时间内没有去刷新token而会话就会过期的原因，请注意：我们要用新换回的refresh\_token把之前的refresh\_token也替换掉，因为老的已经过期了。


## 五 offline\_access角色让refresh\_token永不过期


对于用户登录后，如果授权码模式，如果希望refresh\_token永不过期，可以使用offline\_access这种scope ，前提是你在认证接口调用时，scope地方需要添加offline\_access这个选项，并且你是授权码的认证方式，如图：


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172751173-1018485078.png)


当前客户端模板里，也是需要添加这个offline\_access的客户端模板


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172806003-1253310979.png)


为指定的用户添加offline\_access角色，如果没有这个角色，需要手动添加。


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172817819-358587507.png)


当没有开启Offline session Max limit时，你的刷新token就是永不过期的，如图


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172831993-1741100512.png)


如果希望控制refresh\_token的有效期，可以开启限制


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172845434-1170636048.png)


生成的refresh\_token的超时时间将是5分钟，300秒，还是上面4个配置，谁小用谁，如图：


![](https://img2024.cnblogs.com/blog/118538/202410/118538-20241016172858252-181417286.png)



> Refresh\_token的JWT串，解析后Typ有两种类型，Refresh和Offline，前者的是通过SSO Session Max来控制它的有效期，而后者Offline就是申请token时，使用的scope包含了offline\_access，它对应的refresh\_token是无不效期的。


 本博客参考[飞数机场](https://ze16.com)。转载请注明出处！
