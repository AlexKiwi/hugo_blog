---
title: "Django前后端的图片存储问题"
date: 2018-10-23T10:09:56+08:00
draft: false
---

### 问题提出

问题是这样的，我需要将自己用PIL包创建的图片存储到数据库中

### 解决过程

在项目设置中文件的存储位置是FTP服务器

django的默认存储位置由DEFAULT_FILE_STORAGE参数给出，我们也可以编写自定义的存储系统（本篇暂时不讲）

```python
DEFAULT_FILE_STORAGE = 'storages.backends.ftp.FTPStorage'
FTP_STORAGE_LOCATION = 'ftp://ftpfile:用户名@ip/lecare-admin_data/media/'
```

这个时候我们PIL创建的图片存储在本机磁盘就不行了需要存储在FTP服务器上的磁盘地址才能显示和正常使用,我们可以使用default_storage,将文件存储到默认的存储位置

```python
>>> from django.core.files.base import ContentFile
>>> from django.core.files.storage import default_storage

>>> path = default_storage.save('/path/to/file', ContentFile('new content'))
>>> path
'/path/to/file'

>>> default_storage.size(path)
11
>>> default_storage.open(path).read()
'new content'

>>> default_storage.delete(path)
>>> default_storage.exists(path)
False
```

我们将PIL创建的图片存储到默认位置，但是需要转换成输入输出流在存储

```python
buf = BytesIO()  # 构建一个输入输出流
im.save(buf, "jpeg")  # 将图片保存到输入输出流,也就是内存中
bur_str = buf.getvalue()  # 获得输入输出流里面的内容
path = default_storage.save(file_name, ContentFile(bur_str))
```

完整代码如下：

```python
def create_img_for_text(text, file_name, is_supplement=False):
    im = Image.new("RGB", (1000, 1000), (255, 255, 255))
    dr = ImageDraw.Draw(im)
    font = ImageFont.truetype("/opt/uwsgi/log/lecare-api/yahei.ttf", 18)
    font_title = ImageFont.truetype("/opt/uwsgi/log/lecare-api/yahei.ttf", 30)
    font_supplement = ImageFont.truetype('/opt/uwsgi/log/lecare-api/yahei.ttf', 100)
    dr.text((450, 5), u'申请信息', font=font_title, fill="#000000")
    dr.text((10, 100), text, font=font, fill="#000000")
    if is_supplement:
        dr.text((800, 200), u'补', font=font_supplement, fill="#ff0000")
    buf = BytesIO()  # 构建一个输入输出流
    im.save(buf, "jpeg")  # 将图片保存到输入输出流,也就是内存中
    bur_str = buf.getvalue()  # 获得输入输出流里面的内容

    path = default_storage.save(file_name, ContentFile(bur_str))
```



### 拓展思考

写到这里我想起之前写一个跟换头像的需求，前端传来的加密的图片

 ```html
//在html代码img标签里的写法
<img src="data:image/gif;base64,R0lGODlhHAAmAKIHAKqqqsvLy0hISObm5vf394uLiwAAAP///yH5B…EoqQqJKAIBaQOVKHAXr3t7txgBjboSvB8EpLoFZywOAo3LFE5lYs/QW9LT1TRk1V7S2xYJADs=">

 ```

使用base64加密的图片不需要单独发送http请求来请求图片，可以直接和页面一起下载到浏览器，在css中的写法是这样的

```css
//在css里的写法
#fkbx-spch, #fkbx-hspch {
  background: url(data:image/gif;base64,R0lGODlhHAAmAKIHAKqqqsvLy0hISObm5vf394uLiwAAAP///yH5B…EoqQqJKAIBaQOVKHAXr3t7txgBjboSvB8EpLoFZywOAo3LFE5lYs/QW9LT1TRk1V7S2xYJADs=) no-repeat center;
}
```

当然base64加密不仅用于图片加密上，迅雷的专用地址也是base64加密的

```
thunder://QUFodHRwOi8vZG93bi5zYW5kYWkubmV0L3RodW5kZXI3L1RodW5kZXI3LjEuNS4yMTUyLmV4ZVpa
```

* 为什么要使用base64加密呢

  其实base64加密的图片并不比base64字符串大，并且为了减少html请求我们也可以用**CssSprites**，所以base64加密图片的用法会在以下情况下：

  **如果图片足够小且因为用处的特殊性无法被制作成雪碧图（CssSprites），在整个网站的复用性很高且基本不会被更新**。

  那么此时使用 base64 编码传输图片就可谓好钢用在刀刃上，思前想后，符合这个规则的，有一个是我们经常会遇到的，就是页面的背景图 



* 前端和后端图片转换

我们从前端拿到的base64编码后的图片怎在后端存储呢？ 只需要去掉 data:image/gif;base64,剩下的就是图片base64的内容了，相关代码如下

```python 
class AvatarChangeView(View):
    def get(self, request):
        return render(request, 'user/upload.html')

    def post(self, request):
        image_base64 = request.POST.get('image')
        base64_data = re.sub('^data:image/.+;base64,', '', image_base64)
        binary_data = base64.b64decode(base64_data)

        try:
            filename = utils.update_pic(binary_data, settings.MEDIA_ROOT + '/user/avatars')
            request.user.profile.image = 'user/avatars/' + filename
            request.user.profile.save()
            return JsonResponse({'res': 1})
        except Exception as e:
            return JsonResponse({'res': 0, 'msg': e.message})
```

同样的后端的图片如果想base64放在前端需要吧图片转换成输入输出流再base64加密 在加上data:image/gif;base64就可以了。





原创文章，文笔有限，才疏学浅，文中若有不正之处，万望告知。

参考资料：

https://docs.djangoproject.com/en/2.1/topics/files/

https://blog.csdn.net/rongDang/article/details/81225879

https://www.cnblogs.com/coco1s/p/4375774.html
