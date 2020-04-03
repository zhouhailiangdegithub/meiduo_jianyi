# Python脚本静态化详情页

> 提示：
* 商品详情页查询数据量大，而且是用户频繁访问的页面。
* 类似首页广告，为了减少数据库查询次数，提升页面响应效率，我们也要对详情页进行静态化处理。

> 静态化说明：
* 首页广告的数据变化非常的频繁，所以我们最终使用了**`定时任务`**进行静态化。
* 详情页的数据变化的频率没有首页广告那么频繁，而且是当SKU信息有改变时才要更新的，所以我们采用新的静态化方案。
    * 方案一：通过Python脚本手动一次性批量生成所有商品静态详情页。
    * 方案二：后台运营人员修改了SKU信息时，异步的静态化对应的商品详情页面。
    * 我们在这里先使用方案一来静态详情页。当有运营人员参与时才会补充方案二。
    
> 注意：
* 用户数据和购物车数据不能静态化。
* 热销排行和商品评价不能静态化。
    
### 1. 定义批量静态化详情页脚本文件

> **1.准备脚本目录和Python脚本文件**

<img src="/goods/images/65脚本文件夹.png" style="zoom:50%">

> **2.指定Python脚本环境**

```python
#!/usr/bin/env python
```

> **3.添加Python脚本导包路径**

```python
#!/usr/bin/env python

import sys
sys.path.insert(0, '../')
```

> **4.设置Python脚本Django环境**

```python
#!/usr/bin/env python

import sys
sys.path.insert(0, '../')

import os
if not os.getenv('DJANGO_SETTINGS_MODULE'):
    os.environ['DJANGO_SETTINGS_MODULE'] = 'meiduo_mall.settings.dev'

import django
django.setup()
```

> **5.编写静态化详情页Python脚本代码**

```python
#!/usr/bin/env python

import sys
sys.path.insert(0, '../')

import os
if not os.getenv('DJANGO_SETTINGS_MODULE'):
    os.environ['DJANGO_SETTINGS_MODULE'] = 'meiduo_mall.settings.dev'

import django
django.setup()

from django.template import loader
from django.conf import settings

from goods import models
from contents.utils import get_categories
from goods.utils import get_breadcrumb


def generate_static_sku_detail_html(sku_id):
    """
    生成静态商品详情页面
    :param sku_id: 商品sku id
    """
    # 获取当前sku的信息
    sku = models.SKU.objects.get(id=sku_id)

    # 查询商品频道分类
    categories = get_categories()
    # 查询面包屑导航
    breadcrumb = get_breadcrumb(sku.category)

    # 构建当前商品的规格键
    sku_specs = sku.specs.order_by('spec_id')
    sku_key = []
    for spec in sku_specs:
        sku_key.append(spec.option.id)
    # 获取当前商品的所有SKU
    skus = sku.spu.sku_set.all()
    # 构建不同规格参数（选项）的sku字典
    spec_sku_map = {}
    for s in skus:
        # 获取sku的规格参数
        s_specs = s.specs.order_by('spec_id')
        # 用于形成规格参数-sku字典的键
        key = []
        for spec in s_specs:
            key.append(spec.option.id)
        # 向规格参数-sku字典添加记录
        spec_sku_map[tuple(key)] = s.id
    # 获取当前商品的规格信息
    goods_specs = sku.spu.specs.order_by('id')
    # 若当前sku的规格信息不完整，则不再继续
    if len(sku_key) < len(goods_specs):
        return
    for index, spec in enumerate(goods_specs):
        # 复制当前sku的规格键
        key = sku_key[:]
        # 该规格的选项
        spec_options = spec.options.all()
        for option in spec_options:
            # 在规格参数sku字典中查找符合当前规格的sku
            key[index] = option.id
            option.sku_id = spec_sku_map.get(tuple(key))
        spec.spec_options = spec_options

    # 上下文
    context = {
        'categories': categories,
        'breadcrumb': breadcrumb,
        'sku': sku,
        'specs': goods_specs,
    }

    template = loader.get_template('detail.html')
    html_text = template.render(context)
    file_path = os.path.join(settings.STATICFILES_DIRS[0], 'detail/'+str(sku_id)+'.html')
    with open(file_path, 'w') as f:
        f.write(html_text)

if __name__ == '__main__':
    skus = models.SKU.objects.all()
    for sku in skus:
        print(sku.id)
        generate_static_sku_detail_html(sku.id)
```

### 2. 执行批量静态化详情页脚本文件

> **1.添加`regenerate_detail_html.py`脚本文件可执行权限**

```bash
$ chmod +x regenerate_detail_html.py
```

<img src="/goods/images/66可执行权限1.png" style="zoom:50%">
**********************************************************
<img src="/goods/images/66可执行权限2.png" style="zoom:50%">

> **2.执行批量静态化详情页脚本文件**

```bash
$ python regenerate_detail_html.py
```

<img src="/goods/images/67详情页静态化结果.png" style="zoom:50%">

> **提示**：跟测试静态首页一样的，使用Python自带的http.server模块来模拟静态服务器，提供静态首页的访问测试。

```bash
# 进入到static上级目录
$ cd ~/projects/meiduo_project/meiduo_mall/meiduo_mall
# 开启测试静态服务器
$ python -m http.server 8080 --bind 127.0.0.1
```