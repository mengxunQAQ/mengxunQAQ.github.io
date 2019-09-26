---
layout:     post
title:      Django安装app需要注意的问题
subtitle:   从一个BUG说起
date:       2018-04-22
author:     mengxun
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Django
---


## 前言

这几天在写项目的时候遇见了一个BUG，就是application明明安装了，却总是报错，后来进源码研究了一下，发现了里面的玄机。

### 问题的由来

先看看项目布局，目录类似下面这种：
![](https://i.loli.net/2018/07/14/5b4966f6eb711.png)
然后配置文件里是这样安装application的：
![](https://i.loli.net/2018/07/14/5b4968249bb61.png)
好，如果现在这种情况直接运行会报如下的错：
![](https://i.loli.net/2018/07/14/5b4968c5ad6f1.png)

这个报错信息什么意思呢，就是说`模型类没有声明app_label并且没在安装的application里面定义`。

那么现在问题来了，我是没在模型类里写app_label，而且文档里有这样说：

> If a model is defined outside of an application in [`INSTALLED_APPS`](https://docs.djangoproject.com/en/2.1/ref/settings/#std:setting-INSTALLED_APPS), it must declare which app it belongs to:
>
> ```
> app_label = 'myapp'
> ```

这波说的我不亏，可是问题我application明明装了呀，而且模型类也是在application里面定义的，还报错，那就只能去源码里看看是什么问题了

### 发现问题

进入到最后报错的来源，django的源码根目录下的`db/models/base.py`里，代码如下（介于篇幅，这里只贴主要代码）：

```python
class ModelBase(type):
    """
    Metaclass for all models.
    """
    def __new__(cls, name, bases, attrs):
        super_new = super(ModelBase, cls).__new__

        # Also ensure initialization is only performed for subclasses of Model
        # (excluding Model class itself).
        parents = [b for b in bases if isinstance(b, ModelBase)]
        if not parents:
            return super_new(cls, name, bases, attrs)

        # Create the class.
        module = attrs.pop('__module__')
        new_attrs = {'__module__': module}
        classcell = attrs.pop('__classcell__', None)
        if classcell is not None:
            new_attrs['__classcell__'] = classcell
        new_class = super_new(cls, name, bases, new_attrs)
        attr_meta = attrs.pop('Meta', None)
        abstract = getattr(attr_meta, 'abstract', False)
        if not attr_meta:
            meta = getattr(new_class, 'Meta', None)
        else:
            meta = attr_meta
        base_meta = getattr(new_class, '_meta', None)

        app_label = None

        # Look for an application configuration to attach the model to.
        app_config = apps.get_containing_app_config(module)

        if getattr(meta, 'app_label', None) is None:
            if app_config is None:
                if not abstract:
                    raise RuntimeError(
                        "Model class %s.%s doesn't declare an explicit "
                        "app_label and isn't in an application in "
                        "INSTALLED_APPS." % (module, name)
                    )

            else:
                app_label = app_config.label

```

如果你仔细看的话会发现报错信息就在其中，那么是什么导致报错呢，开了debug探究一下原因：

- 首先，报错信息上面有三个判断，这三个判断同时成立才会发生报错，很显然，我的代码让这三个判断同时成立了(っ °Д °;)っ.

- 然后由于没定义`app_label`第一个判断肯定成立，然后我也没有定义`abstract = True`使类变成抽象基类，那真相就只有一个了，就是我让`app_config`是`None`了......

- 那么，`app_config`为什么为`None`呢，再往上面有一行代码:

  ```python
  app_config = apps.get_containing_app_config(module)

  ```

- 好，那现在切入get_containing_app_config这个方法（看看这命名，很语义化，值得偷师一波）：

  ```python
  def get_containing_app_config(self, object_name):
      """
      Look for an app config containing a given object.

      object_name is the dotted Python path to the object.

      Returns the app config for the inner application in case of nesting.
      Returns None if the object isn't in any registered app config.
      """
      self.check_apps_ready()
      candidates = []
      for app_config in self.app_configs.values():
          if object_name.startswith(app_config.name):
              subpath = object_name[len(app_config.name):]
              if subpath == '' or subpath[0] == '.':
                  candidates.append(app_config)
      if candidates:
      	return sorted(candidates, key=lambda ac: -len(ac.name))[0]
  ```

- 既然这个方法会返回空，那就说明`candidates`这个list是空，也就说明没有append进对象，然后DEBUG发现，在check了几个Django自带的app的models之后，后面的`object_name`参数会传递自己扩展的models，而在这里这个`object_name`参数为`"apps.assets.models"`，然后`app_config`根本就没有以`"apps.assets.models"`开头的`name`属性，所以就算遍历完也没办法让`starstwith`返回`True`，直接导致`candidates`是个空`list`，随后引发连锁反应，进一步导致报错。

### 解决问题

既然问题的症结已经发现，那下一步自然是解决这个问题，那么如何解决呢：

- 问题的根就在于object_name为`"apps.assets.models"`的时候，app_config.name为`"assets"`，无法通过startswith的验证。
- 那么就可以从以下两个方面入手解决：
  - 让`object_name` 从`"apps.assets.models"`变成`assets.models`：可以在模型类的加app_label属性，而每个都加app_label太过麻烦。不想每个都加，就得写个抽象基类，然后再让每个类继承，也很麻烦，这种方法排除；
  - 让`app_config.name`从`"assets"`变成`"apps.assets"`：可以直接在安装application的时候，将名字前面加上`apps.`；也可以写成类似于`assets.apps.AssetsConfig`的形式。强烈推荐后者，因为在你用IDE变更目录的时候（比如说把app1，app2从项目根目录移到项目根目录的apps目录下），这个子类里的name会随之改变，避免手工更改（Jetbrains家的东西就是好用）
