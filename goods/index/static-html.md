# 页面静态化

> 思考：
* 美多商城的首页访问频繁，而且查询数据量大，其中还有大量的循环处理。

> 问题：
* 用户访问首页会耗费服务器大量的资源，并且响应数据的效率会大大降低。

> 解决：
* 页面静态化

### 1. 页面静态化介绍

> **1.为什么要做页面静态化**

* 减少数据库查询次数。
* 提升页面响应效率。

> **2.什么是页面静态化**

* 将动态渲染生成的页面结果保存成html文件，放到静态文件服务器中。
* 用户直接去静态服务器，访问处理好的静态html文件。

<img src="/goods/images/48静态文件访问示意图.png" style="zoom:35%">

> **3.页面静态化注意点**

* 用户相关数据不能静态化：
    * 用户名、购物车等不能静态化。
* 动态变化的数据不能静态化：
    * 热销排行、新品推荐、分页排序数据等等。
* 不能静态化的数据处理：
    * 可以在用户得到页面后，在页面中向后端发送Ajax请求获取相关数据。
    * 直接使用模板渲染出来。
    * 其他合理的处理方式等等。
    
### 2. 首页页面静态化实现

> **1.首页页面静态化实现步骤**

* 查询首页相关数据
* 获取首页模板文件
* 渲染首页html字符串
* 将首页html字符串写入到指定目录，命名'index.html'

> **2.首页页面静态化实现**

<img src="/goods/images/49封装首页静态化过程.png" style="zoom:40%">

```python
def generate_static_index_html():
    """
    生成静态的主页html文件
    """
    print('%s: generate_static_index_html' % time.ctime())

    # 获取商品频道和分类
    categories = get_categories()

    # 广告内容
    contents = {}
    content_categories = ContentCategory.objects.all()
    for cat in content_categories:
        contents[cat.key] = cat.content_set.filter(status=True).order_by('sequence')

    # 渲染模板
    context = {
        'categories': categories,
        'contents': contents
    }

    # 获取首页模板文件
    template = loader.get_template('index.html')
    # 渲染首页html字符串
    html_text = template.render(context)
    # 将首页html字符串写入到指定目录，命名'index.html'
    file_path = os.path.join(settings.STATICFILES_DIRS[0], 'index.html')
    with open(file_path, 'w', encoding='utf-8') as f:
        f.write(html_text)
```

> **3.首页页面静态化测试效果**

<img src="/goods/images/50首页页面静态化效果1.png" style="zoom:40%">

<img src="/goods/images/50首页页面静态化效果2.png" style="zoom:40%">

> **提示**：使用Python自带的`http.server`模块来模拟静态服务器，提供静态首页的访问测试。

```bash
# 进入到static上级目录
$ cd ~/projects/meiduo_project/meiduo_mall/meiduo_mall
# 开启测试静态服务器
$ python -m http.server 8080 --bind 127.0.0.1
```

<img src="/goods/images/50首页页面静态化效果3.png" style="zoom:50%">

### 3. 定时任务crontab静态化首页

> 重要提示：

* 对于首页的静态化，考虑到页面的数据可能由多名运营人员维护，并且经常变动，所以将其做成定时任务，即定时执行静态化。
* 在Django执行定时任务，可以通过`django-crontab`扩展来实现。

> **1.安装 django-crontab**

```bash
$ pip install django-crontab
```

> **2.注册 django-crontab 应用**

```python
INSTALLED_APPS = [    
    'django_crontab', # 定时任务
]
```

> **3.设置定时任务**

```
定时时间基本格式 :

*  *  *  *  *

分 时 日 月 周    命令

M: 分钟（0-59）。每分钟用 * 或者 */1 表示
H：小时（0-23）。（0表示0点）
D：天（1-31）。
m: 月（1-12）。
d: 一星期内的天（0~6，0为星期天）。
```

> 定时任务分为三部分定义：

* 任务时间
* 任务方法
* 任务日志

```python
CRONJOBS = [
    # 每1分钟生成一次首页静态文件
    ('*/1 * * * *', 'contents.crons.generate_static_index_html', '>> ' + os.path.join(os.path.dirname(BASE_DIR), 'logs/crontab.log'))
]
```

> 解决 crontab 中文问题
* 在定时任务中，如果出现非英文字符，会出现字符异常错误

```python
CRONTAB_COMMAND_PREFIX = 'LANG_ALL=zh_cn.UTF-8'
```

> **4.管理定时任务**

```bash
# 添加定时任务到系统中
$ python manage.py crontab add

# 显示已激活的定时任务
$ python manage.py crontab show

# 移除定时任务
$ python manage.py crontab remove
```

<img src="/goods/images/51定时任务效果.png" style="zoom:50%">
