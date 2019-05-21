# Shiro

安全校验框架,是一个绕不过的东西.

相信我,我绕过,然后又泪流满面的绕回来了.

---

## 1. Shrio 各种角色区别

### 1.1 shiro 校验类型

origin: [cloudsky 的博客](http://www.cnblogs.com/skyme/archive/2011/09/24/2189364.html)

| 过滤器名称 | 过滤器类                                                         | 描述                                                                                                                 |
| ---------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| anon       | org.apache.shiro.web.filter.authc.AnonymousFilter                | 匿名过滤器                                                                                                           |
| authc      | org.apache.shiro.web.filter.authc.FormAuthenticationFilter       | 如果继续操作,需要做对应的表单验证否则不能通过                                                                        |
| authcBasic | org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter  | 基本 http 验证过滤,如果不通过,跳转屋登录页面                                                                         |
| logout     | org.apache.shiro.web.filter.authc.LogoutFilter                   | 登录退出过滤器 noSessionCreation org.apache.shiro.web.filter.session.NoSessionCreationFilter 没有 session 创建过滤器 |
| perms      | org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter | 权限过滤器                                                                                                           |
| port       | org.apache.shiro.web.filter.authz.PortFilter                     | 端口过滤器,可以设置是否是指定端口如果不是跳转到登录页面                                                              |
| rest       | org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter     | http 方法过滤器,可以指定如 post 不能进行访问等                                                                       |
| roles      | org.apache.shiro.web.filter.authz.RolesAuthorizationFilter       | 角色过滤器,判断当前用户是否指定角色                                                                                  |
| ssl        | org.apache.shiro.web.filter.authz.SslFilter                      | 请求需要通过 ssl,如果不是跳转回登录页                                                                                |
| user       | org.apache.shiro.web.filter.authc.UserFilter                     | 如果访问一个已知用户,比如记住我功能,走这个过滤器                                                                     |

### 1.2 前端应用

from:[shiro 权限验证标签](https://www.cnblogs.com/jifeng/p/4500410.html)

引入校验标签

```html
<%@ taglib prefix="shiro" uri="http://shiro.apache.org/tags" %>
```

#### 1.2.1 guest

guest:验证当前用户是否为“访客”,即未认证(包含未记住)的用户.

```html
<shiro:guest>
  Hi there! Please <a href="login.jsp">Login</a> or
  <a href="signup.jsp">Signup</a> today!
</shiro:guest>
```

#### 1.2.2 user

user: 认证通过或已记住的用户.

```html
<shiro:user>
    Welcome back John!  Not John? Click <a href="login.jsp">here<a> to login.
</shiro:user>
```

#### 1.2.3 authenticated

authenticated 标签 :已认证通过的用户.不包含已记住的用户,这是与 user 标签的区别所在.

```html
<shiro:authenticated>
  <a href="updateAccount.jsp">Update your contact information</a>.
</shiro:authenticated>
```

#### 1.2.4 notAuthenticated

notAuthenticated 标签 :未认证通过用户,与 authenticated 标签相对应.与 guest 标签的区别是,该标签包含已记住用户.

```html
<shiro:notAuthenticated>
  Please <a href="login.jsp">login</a> in order to update your credit card
  information.
</shiro:notAuthenticated>
```

#### 1.2.5 principal

principal 标签 :输出当前用户信息,通常为登录帐号信息.

```html
Hello, <shiro:principal />, how are you today?
```

#### 1.2.6 hasRole

hasRole 标签 :验证当前用户是否属于该角色.

```html
<shiro:hasRole name="administrator">
  <a href="admin.jsp">Administer the system</a>
</shiro:hasRole>
```

#### 1.2.7 lacksRole

lacksRole 标签 :与 hasRole 标签逻辑相反,当用户不属于该角色时验证通过.

```html
<shiro:lacksRole name="administrator">
  Sorry, you are not allowed to administer the system.
</shiro:lacksRole>
```

#### 1.2.8 hasAnyRole

hasAnyRole 标签 :验证当前用户是否属于以下任意一个角色.

```html
<shiro:hasAnyRoles name="developer, project manager, administrator">
    You are either a developer, project manager, or administrator.
</shiro:lacksRole>
```

#### 1.2.9 hasPermission

hasPermission 标签 :验证当前用户是否拥有指定权限.

```html
<shiro:hasPermission name="user:create">
  <a href="createUser.jsp">Create a new User</a>
</shiro:hasPermission>
```

#### 1.2.10 lacksPermission

lacksPermission 标签 :与 hasPermission 标签逻辑相反,当前用户没有制定权限时,验证通过.

```html
<shiro:hasPermission name="user:create">
  <a href="createUser.jsp">Create a new User</a>
</shiro:hasPermission>
```

---

## 2. Shiro 源码解读

---

## 3. 参考文档

a. [前端 Shrio 权限标签](http://www.cnblogs.com/cocosili/p/7103025.html)
