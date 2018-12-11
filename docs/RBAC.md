## November 27, 2018 7:16 PM
## RBAC基础上实现不同的用户不同的界面.md

# 实现不同的用户不同的界面
## 在权限表设置字段区分不同的权限
**is_menu 决定这个ＵＲＬ在不在视图上显示　**

```
# 权限表
class UserPurview(models.Model):
    title = models.CharField(max_length=32)
    url = models.CharField(max_length=80)
    is_menu = models.BooleanField(default=False, verbose_name='菜单')
    icon = models.CharField(max_length=24, null=True, blank=True, verbose_name='图标')

```

#　设置session 
查询表注意去重
## QuerySet对象.distinct()

## 这里把设置session 的代码封装了工具
#### 把相同可以实现多次的调用的代码片段实现封装，面对函数编程，领悟面对对象式编程

```
'''
    设置session　数据
'''
from SZcrm import settings


def init(request, user_obj):
    # 1.查询用户权限
    # 2.把查询的url 添加到session 里面
    url_list = user_obj.roles.all().values(
        'permissions__url', 'permissions__title', 'permissions__is_menu',
        'permissions__icon').distinct()  # 返回的QuerySet对象，没个元素都是元组
    # 3.把查询的ｕｒｌ加入列表
    # 取到url_list
    #  permissions_list = [permissions['permissions__url'] for permissions in url_list]
    permissions_list = []
    # 存放菜单列表
    menu_list = []
    for i in url_list:
        permissions_list.append(i['permissions__url'])
        if i.get('permissions__is_menu'):
            menu_list.append({
                'url': i['permissions__url'],
                'title': i['permissions__title'],
                'icon': i['permissions__icon'],
            })
    session_key = getattr(settings, 'PERMISSION_URL_KEY', 'permissions_url')
    menu_key = getattr(settings, 'SECRET_MENU', 'menu_list')

    # 设置session, 存取menu列表
    request.session[menu_key] = menu_list
    # 设置session, 存取ｕｒｌ列表
    request.session[session_key] = permissions_list


```

# 前端的代码实现
## 在写程序之前想想如何规范代码的健壮型。
前端接受数据依赖后端，有一些变量我们写在了配置文件里面，　防止后期变动，影响整个项目。那么前端如何实现呢
### 自定义django 模板函数　inclusion_tag
```
	1.在当前的ａｐｐ下建立templatetags包
	2.	from django import template
    	register = template.Library()
		自己的业务逻辑

```
## 代码实现
```
from django import template
from SZcrm import settings
import re
register = template.Library()

@register.inclusion_tag(filename='rbac/menu.html')
def meun_list(request):
    # １． 获取当前的url
    new_url = request.path_info
    # ２． 获取session　储存的session　key
    meun_key = getattr(settings, 'SECRET_MENU', 'menu_list')
    # 3. 获取session 里面的权限限定
    meun_list = request.session.get(meun_key)
    # 4.　循环判断：如果当前的ｕｒｌ在权限里面加一个css样式，
    for meun in meun_list:
        if re.match(r'^{}$'.format(meun['url']), new_url):
            meun['class'] = 'active'
    return {'menu_list': meun_list}

```
## inclusion_tag html 代码　与不同的前端代码无区别
```
<ul class="nav nav-sidebar navbar-inverse">
    {% for menu in menu_list %}
        <li class="{{ menu.class }}"><a href="{{ menu.url }}"><span
                class="icon-wrap"><i
                class="fa {{ menu.icon }}"></i></span>{{ menu.title }}</a></li>
    {% endfor %}
</ul>

```
### **时刻注意文件路径问题，　数据库导入　最为重要的是抄代码单词问题**

