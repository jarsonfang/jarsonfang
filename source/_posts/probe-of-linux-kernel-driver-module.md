---
title: linux设备模型深探
tags:
  - 设备模型
categories:
  - kernel
  - 设备驱动
date: 2014-02-27 15:23:42
---

原文链接：<http://blog.chinaunix.net/uid-20543183-id-1930813.html>

源码文件：
```c
./lib/kobject.c
./drivers/base/bus.c
./drivers/base/driver.c
./drivers/base/core.c
```
目录：  
[一：前言](#1)  
[二：kobject、kset和ktype](#2)  
[三：kobject、kset和ktype的操作](#3)  

*   [kobject 操作](#3.1)
*   [kset 操作](#3.2)

[四：bus、device和device_driver](#4)

*   [总线注册](#4.1)
*   [设备注册](#4.2)
*   [驱动注册](#4.3)

[五：小结](#5)
<!--more-->

## <a name=1>一：前言</a>

Linux设备模型是一个极其复杂的结构体系，在编写驱动程序的时候，通常不会用到这方面的东西，但是，理解这部份内容，对于我们理解linux设备驱动的结构是大有裨益的。我们不但可以在编写程序程序的时候知其然亦知其所以然，又可以学习到一种极其精致的架构设计方法。由于之前已经详细分析了`sysfs`文件系统，所以本节的讨论主要集中在设备模型的底层实现上。上层的接口，如`pci`、`usb`、网络设备都可以看成是底层的封装。

## <a name=2>二：kobject、kset和ktype </a>

`kobject`,`kset`,`ktype`这三个结构是设备模型中的下层架构。模型中的每一个元素都对应一个`kobject`，`kset`和`ktype`可以看成是`kobject`在层次结构与属性结构方面的扩充。将三者之间的关系用图的方示描述如下：

 {% asset_img kobject-kset-ktype.jpg %}

如上图所示：我们知道，在`sysfs`中每一个目录都对应一个`kobject`，这些`kobject`都有自己的`parent`，在没有指定`parent`的情况下，都会指向它所属的`kset->object`；其次，`kset`也内嵌了`kobject`，这个`kobject`又可以指它上一级的`parent`。就这样，构成了一个空间层次关系。

其实，每个对象都有属性，例如，电源管理，执插拨事性管理等等。因为大部份的同类设备都有相同的属性，因此将这个属性隔离开来，存放在`ktype`中。这样就可以灵活的管理了。记得在分析`sysfs`的时候，对于`sysfs`中的普通文件读写操作都是由`kobject->ktype->sysfs_ops`来完成的。

经过上面的分析，我们大概了解了`kobject`、`kset`与`ktype`的大概架构与相互之间的关系。下面我们从linux源代码中的分析来详细研究他们的操作。

## <a name=3>三：kobject、kset和ktype的操作</a>

为了说明`kobject`的操作，先写一个测试模块，代码如下：
```c
#include <linux/device.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/string.h>
#include <linux/sysfs.h>
#include <linux/stat.h>

MODULE_AUTHOR("eric xiao");
MODULE_LICENSE("Dual BSD/GPL");

void obj_test_release(struct kobject *kobject);
ssize_t eric_test_show(struct kobject *kobject, struct attribute *attr,char *buf);
ssize_t eric_test_store(struct kobject *kobject,struct attribute *attr,const char *buf, size_t count);

struct attribute test_attr = {
        .name = "eric_xiao",
        .mode = S_IRWXUGO,
};

static struct attribute *def_attrs[] = {
        &test_attr,
        NULL,
};

struct sysfs_ops obj_test_sysops =
{
        .show = eric_test_show,
        .store = eric_test_store,
};

struct kobj_type ktype =
{
        .release = obj_test_release,
        .sysfs_ops=&obj_test_sysops,
        .default_attrs=def_attrs,
};

void obj_test_release(struct kobject *kobject)
{
        printk("eric_test: release .\n");
}

ssize_t eric_test_show(struct kobject *kobject, struct attribute *attr,char *buf)
{
        printk("have show.\n");
        printk("attrname:%s.\n", attr->name);
        sprintf(buf,"%s\n",attr->name);
        return strlen(attr->name)+2;
}

ssize_t eric_test_store(struct kobject *kobject,struct attribute *attr,const char *buf, size_t count)
{
        printk("havestore\n");
        printk("write: %s\n",buf);
        return count;
}

struct kobject kobj;
static int kobject_test_init()
{
        printk("kboject test init.\n");
        kobject_init_and_add(&kobj,&ktype,NULL,"eric_test");
        return 0;
}

static int kobject_test_exit()
{
        printk("kobject test exit.\n");
        kobject_del(&kobj);
        return 0;
}

module_init(kobject_test_init);
module_exit(kobject_test_exit);
```
加载模块之后，会发现，在`/sys`下多了一个`eric_test`目录。该目录下有一个叫`eric_xiao`的文件。如下所示：
```bash
[root@localhost eric_test]# ls
eric_xiao
```
用`cat`察看此文件：
```bash
[root@localhost eric_test]# cat eric_xiao
eric_xiao
```
再用`echo`往里面写点东西；
```bash
[root@localhost eric_test]# echo  hello > eric_xiao
```
Dmesg的输出如下：
```bash
have show.
attrname:eric_xiao.
havestore
write: hello
```
如上所示，我们看到了`kobject`的大概建立过程。

### <a name=3.1>kobject 操作</a>

我们来看一下`kobject_init_and_add()`的实现，在这个函数里，包含了对`kobject`的大部份操作。
```c
int kobject_init_and_add(struct kobject *kobj, struct kobj_type *ktype,
               struct kobject *parent, const char *fmt, ...)
{
     va_list args;
     int retval;
     //初始化kobject
     kobject_init(kobj, ktype);

     va_start(args, fmt);
     //为kobjcet设置名称，在sysfs中建立相关信息
     retval = kobject_add_varg(kobj, parent, fmt, args);
     va_end(args);

     return retval;
}
```
上面的流程主要分为两部份：一部份是`kobject`的初始化，在这一部份，它将`kobject`与给定的`ktype`关联起来，初始化`kobject`中的各项结构；另一部份是`kobject`的名称设置，空间层次关系的设置，具体表现在`sysfs`文件系统中。

对于第一部份，代码比较简单，这里不再赘述。跟踪第二部份，也就是`kobject_add_varg()`的实现。
```c
static int kobject_add_varg(struct kobject *kobj, struct kobject *parent,
                  const char *fmt, va_list vargs)
{
     va_list aq;
     int retval;

     va_copy(aq, vargs);
     //设置kobject的名字。即kobject的name成员
     retval = kobject_set_name_vargs(kobj, fmt, aq);
     va_end(aq);
     if (retval) {
         printk(KERN_ERR "kobject: can not set name properly!\n");
         return retval;
     }
     //设置kobject的parent。在上面的例子中，我们没有给它指定父结点
     kobj->parent = parent;
     //在sysfs中添加kobjcet信息
     return kobject_add_internal(kobj);
}
```
设置好`kobject->name`后，转入`kobject_add_internal()`，在sysfs中创建空间结构。代码如下：
[cc lang="c"]
static int kobject_add_internal(struct kobject *kobj)
{
     int error = 0;
     struct kobject *parent;

     if (!kobj)
         return -ENOENT;
     //如果kobject的名字为空.退出
     if (!kobj->name || !kobj->name[0]) {
         pr_debug("kobject: (%p): attempted to be registered with empty "
               "name!\n", kobj);
         WARN_ON(1);
         return -EINVAL;
     }

     //取kobject的父结点
     parent = kobject_get(kobj->parent);
     //如果kobject的父结点没有指定，就将kset->kobject做为它的父结点
     /* join kset if set, use it as parent if we do not already have one */
     if (kobj->kset) {
         if (!parent)
              parent = kobject_get(&kobj->kset->kobj);
         kobj_kset_join(kobj);
         kobj->parent = parent;
     }
     //调试用
     pr_debug("kobject: '%s' (%p): %s: parent: '%s', set: '%s'\n",
          kobject_name(kobj), kobj, __FUNCTION__,
          parent ? kobject_name(parent) : "<NULL>",
          kobj->kset ? kobject_name(&kobj->kset->kobj) : "<NULL>");

     //在sysfs中创建kobject的相关元素
     error = create_dir(kobj);
     if (error) {
         //如果创建失败。减少相关的引用计数
         kobj_kset_leave(kobj);
         kobject_put(parent);
         kobj->parent = NULL;

         /* be noisy on error issues */
         if (error == -EEXIST)
              printk(KERN_ERR "%s failed for %s with "
                     "-EEXIST, don't try to register things with "
                     "the same name in the same directory.\n",
                     __FUNCTION__, kobject_name(kobj));
         else
              printk(KERN_ERR "%s failed for %s (%d)\n",
                     __FUNCTION__, kobject_name(kobj), error);
         dump_stack();
     } else
         //如果创建成功。将state_in_sysfs建为1。表示该object已经在sysfs中了
         kobj->state_in_sysfs = 1;

     return error;
}
[/cc]
这段代码比较简单，它主要完成kobject父结点的判断和选定，然后再调用create_dir()在sysfs创建相关信息。该函数代码如下：
[cc lang="c"]
static int create_dir(struct kobject *kobj)
{
     int error = 0;
     if (kobject_name(kobj)) {
          //为kobject创建目录
         error = sysfs_create_dir(kobj);
         if (!error) {
              //为kobject->ktype中的属性创建文件
              error = populate_dir(kobj);
              if (error)
                   sysfs_remove_dir(kobj);
         }
     }
     return error;
}
[/cc]
我们在上面的示例中看到的/sys下的eric_test目录，以及该目录下面的eric_xiao的这个文件就是这里被创建的。我们先看一下kobject所表示的目录创建过程，这是在sysfs_create_dir()中完成的。代码如下：
[cc lang="c"]
int sysfs_create_dir(struct kobject * kobj)
{
     struct sysfs_dirent *parent_sd, *sd;
     int error = 0;

     BUG_ON(!kobj);
     /*
      *如果kobject的parnet存在，就在目录点的目录下创建这个目录;
      *如果父结点不存在，就在/sys下面创建结点。
      *在上面的流程中，我们可能并没有为其指定父结点，也没有为其指定kset。
      */
     if (kobj->parent)
         parent_sd = kobj->parent->sd;
     else
         parent_sd = &sysfs_root;

     //在sysfs中创建目录
     error = create_dir(kobj, parent_sd, kobject_name(kobj), &sd);
     if (!error)
         kobj->sd = sd;
     return error;
}
[/cc]
在这里，我们就要联系之前分析过的sysfs文件系统的研究了。如果不太清楚的，可以再找到那篇文章仔细的研读一下。create_dir()就是在sysfs中创建目录的接口，在之前已经详细分析过了。这里不再讲述。

接着看为kobject->ktype中的属性创建文件，这是在populate_dir()中完成的。代码如下：
[cc lang="c"]
static int populate_dir(struct kobject *kobj)
{
     struct kobj_type *t = get_ktype(kobj);
     struct attribute *attr;
     int error = 0;
     int i;

     if (t && t->default_attrs) {
         for (i = 0; (attr = t->default_attrs[i]) != NULL; i++) {
              error = sysfs_create_file(kobj, attr);
              if (error)
                   break;
         }
     }
     return error;
}
[/cc]
这段代码比较简单，它遍历ktype中的属性，然后为其建立文件。请注意：文件的操作最后都会回溯到ktype->sysfs_ops的show和store这两个函数中。

Kobject的创建已经分析完了，接着分析怎么将一个kobject注销掉。注意过程是在kobject_del()中完成的。代码如下：
[cc lang="c"]
void kobject_del(struct kobject *kobj)
{
     if (!kobj)
         return;

     sysfs_remove_dir(kobj);
     kobj->state_in_sysfs = 0;
     kobj_kset_leave(kobj);
     kobject_put(kobj->parent);
     kobj->parent = NULL;
}
[/cc]
该函数会将在sysfs中的kobject对应的目录删除。请注意，属性文件是建立在这个目录下面的，只需要将这个目录删除，属性文件也随之删除。
最后，减少相关的引用计数，如果kobject的引用计数为零。则将其所占空间释放.

<a name=3.2>

### kset 操作
</a>
kset的操作与kobject类似，因为kset中内嵌了一个kobject结构，所以，大部份操作都是集中在kset->kobject上。具体分析一下kset_create_and_add()这个接口,类似上面分析的kobject接口，这个接口也包括了kset的大部分操作。代码如下:
[cc lang="c"]
struct kset *kset_create_and_add(const char *name,
                    struct kset_uevent_ops *uevent_ops,
                    struct kobject *parent_kobj)
{
     struct kset *kset;
     int error;
     //创建一个kset
     kset = kset_create(name, uevent_ops, parent_kobj);
     if (!kset)
         return NULL;
     //注册kset
     error = kset_register(kset);
     if (error) {
         //如果注册失败,释放kset
         kfree(kset);
         return NULL;
     }
     return kset;
}
[/cc]
Kset_create()用来创建一个struct kset结构。代码如下:
[cc lang="c"]
static struct kset *kset_create(const char *name,
                   struct kset_uevent_ops *uevent_ops,
                   struct kobject *parent_kobj)
{
     struct kset *kset;

     kset = kzalloc(sizeof(*kset), GFP_KERNEL);
     if (!kset)
         return NULL;
     kobject_set_name(&kset->kobj, name);
     kset->uevent_ops = uevent_ops;
     kset->kobj.parent = parent_kobj;

     kset->kobj.ktype = &kset_ktype;
     kset->kobj.kset = NULL;

     return kset;
}
[/cc]
我们注意，在这里创建kset时，为其内嵌的kobject指定其ktype结构为kset_ktype。这个结构的定义如下:
[cc lang="c"]
static struct kobj_type kset_ktype = {
     .sysfs_ops    = &kobj_sysfs_ops,
     .release = kset_release,
};
[/cc]
属性文件的读写操作全部都包含在sysfs_ops成员里，kobj_sysfs_ops的定义如下:
[cc lang="c"]
struct sysfs_ops kobj_sysfs_ops = {
     .show    = kobj_attr_show,
     .store   = kobj_attr_store,
};
[/cc]
show,store成员对应的函数代码如下所示:
[cc lang="c"]
static ssize_t kobj_attr_show(struct kobject *kobj, struct attribute *attr,
                    char *buf)
{
     struct kobj_attribute *kattr;
     ssize_t ret = -EIO;

     kattr = container_of(attr, struct kobj_attribute, attr);
     if (kattr->show)
         ret = kattr->show(kobj, kattr, buf);
     return ret;
}

static ssize_t kobj_attr_store(struct kobject *kobj, struct attribute *attr,
                     const char *buf, size_t count)
{
     struct kobj_attribute *kattr;
     ssize_t ret = -EIO;

     kattr = container_of(attr, struct kobj_attribute, attr);
     if (kattr->store)
         ret = kattr->store(kobj, kattr, buf, count);
     return ret;
}
[/cc]
从上面的代码看以看出，会将struct attribute结构转换为struct kobj_attribte结构，也就是说struct kobj_attribte内嵌了一个struct attribute。实际上，这是和宏__ATTR配合在一起使用的，经常用于group中，在这里并不打算研究group，原理都是一样的，这里列出来只是做个说明而已。

创建好了kset之后，会调用kset_register()，这个函数就是kset操作的核心代码了。如下:
[cc lang="c"]
int kset_register(struct kset *k)
{
     int err;

     if (!k)
         return -EINVAL;

     kset_init(k);
     err = kobject_add_internal(&k->kobj);
     if (err)
         return err;
     kobject_uevent(&k->kobj, KOBJ_ADD);
     return 0;
}
[/cc]
在kset_init()里会初始化kset中的其它字段，然后调用kobject_add_internal()为其内嵌的kobject结构建立空间层次结构，之后因为添加了kset，会产生一个事件，这个事件是通过用户空间的hotplug程序处理的，这就是kset明显不同于kobject的地方。详细研究一下这个函数，这对于我们研究hotplug的深层机理是很有帮助的，它的代码如下：
[cc lang="c"]
int kobject_uevent(struct kobject *kobj, enum kobject_action action)
{
     return kobject_uevent_env(kobj, action, NULL);
}
[/cc]
之后，会调用kobject_uevent_env()。这个函数中的三个参数含义分别为:引起事件的kobject，事件类型(add,remove,change,move,online,offline等)，第三个参数是要添加的环境变量。

代码篇幅较长，我们效仿情景分析的做法，分段分析如下:
[cc lang="c"]
int kobject_uevent_env(struct kobject *kobj, enum kobject_action action,
                char *envp_ext[])
{
     struct kobj_uevent_env *env;
     const char *action_string = kobject_actions[action];
     const char *devpath = NULL;
     const char *subsystem;
     struct kobject *top_kobj;
     struct kset *kset;
     struct kset_uevent_ops *uevent_ops;
     u64 seq;
     int i = 0;
     int retval = 0;

     pr_debug("kobject: '%s' (%p): %s\n",
          kobject_name(kobj), kobj, __FUNCTION__);

     /* search the kset we belong to */
     top_kobj = kobj;
     while (!top_kobj->kset && top_kobj->parent)
         top_kobj = top_kobj->parent;

     if (!top_kobj->kset) {
         pr_debug("kobject: '%s' (%p): %s: attempted to send uevent "
               "without kset!\n", kobject_name(kobj), kobj,
               __FUNCTION__);
         return -EINVAL;
     }
[/cc]
因为对事件的处理函数包含在kobject->kset->uevent_ops中，要处理事件，就必须要找到上层的一个不为空的kset。上面的代码就是顺着kobject->parent找不到一个不为空的kset，如果不存在这样的kset，就退出
[cc lang="c"]
     kset = top_kobj->kset;
     uevent_ops = kset->uevent_ops;

     /* skip the event, if the filter returns zero. */
     if (uevent_ops && uevent_ops->filter)
         if (!uevent_ops->filter(kset, kobj)) {
              pr_debug("kobject: '%s' (%p): %s: filter function "
                    "caused the event to drop!\n",
                    kobject_name(kobj), kobj, __FUNCTION__);
              return 0;
         }

     /* originating subsystem */
     if (uevent_ops && uevent_ops->name)
         subsystem = uevent_ops->name(kset, kobj);
     else
         subsystem = kobject_name(&kset->kobj);
     if (!subsystem) {
         pr_debug("kobject: '%s' (%p): %s: unset subsystem caused the "
               "event to drop!\n", kobject_name(kobj), kobj,
               __FUNCTION__);
         return 0;
     }
[/cc]
找到了不为空的kset，就跟kset->uevent_ops->filter()匹配，看这个事件是否被过滤。如果没有被过滤掉，就会调用kset->uevent_ops->name()得到子系统的名称。如果不存在kset->uevent_ops->name()，就会以kobject->name做为子系统名称。
[cc lang="c"]
     /* environment buffer */
     env = kzalloc(sizeof(struct kobj_uevent_env), GFP_KERNEL);
     if (!env)
         return -ENOMEM;

     /* complete object path */
     devpath = kobject_get_path(kobj, GFP_KERNEL);
     if (!devpath) {
         retval = -ENOENT;
         goto exit;
     }

     /* default keys */
     retval = add_uevent_var(env, "ACTION=%s", action_string);
     if (retval)
         goto exit;
     retval = add_uevent_var(env, "DEVPATH=%s", devpath);
     if (retval)
         goto exit;
     retval = add_uevent_var(env, "SUBSYSTEM=%s", subsystem);
     if (retval)
         goto exit;

     /* keys passed in from the caller */
     if (envp_ext) {
         for (i = 0; envp_ext[i]; i++) {
              retval = add_uevent_var(env, envp_ext[i]);
              if (retval)
                   goto exit;
         }
     }
[/cc]
接下来,就应该设置为调用hotplug设置环境变量了。首先,分配一个struct kobj_uevent_env结构用来存放环境变量的值；然后调用kobject_get_path()用来获得引起事件的kobject在sysfs中的路径；再调用add_uevent_var()将动作代表的字串、kobject路径、子系统名称填充到struct kobj_uevent_env中。如果有指定环境变量,也将其添加进去。 kobject_get_path()和add_uevent_var()都比较简单.这里不再详细分析了.请自行查看源代码
[cc lang="c"]
     /* let the kset specific function add its stuff */
     if (uevent_ops && uevent_ops->uevent) {
         retval = uevent_ops->uevent(kset, kobj, env);
         if (retval) {
              pr_debug("kobject: '%s' (%p): %s: uevent() returned "
                    "%d\n", kobject_name(kobj), kobj,
                    __FUNCTION__, retval);
              goto exit;
         }
     }

     /*
      * Mark "add" and "remove" events in the object to ensure proper
      * events to userspace during automatic cleanup. If the object did
      * send an "add" event, "remove" will automatically generated by
      * the core, if not already done by the caller.
      */
     if (action == KOBJ_ADD)
         kobj->state_add_uevent_sent = 1;
     else if (action == KOBJ_REMOVE)
         kobj->state_remove_uevent_sent = 1;

     /* we will send an event, so request a new sequence number */
     spin_lock(&sequence_lock);
     seq = ++uevent_seqnum;
     spin_unlock(&sequence_lock);
     retval = add_uevent_var(env, "SEQNUM=%llu", (unsigned long long)seq);
     if (retval)
         goto exit;
[/cc]
在这里还会调用kobject->kset->uevent_ops->uevent()，让产生事件的kobject添加环境变量，最后将事件序列添加到环境变量中去。
[cc lang="c"]
#if defined(CONFIG_NET)
     /* send netlink message */
     if (uevent_sock) {
         struct sk_buff *skb;
         size_t len;

         /* allocate message with the maximum possible size */
         len = strlen(action_string) + strlen(devpath) + 2;
         skb = alloc_skb(len + env->buflen, GFP_KERNEL);
         if (skb) {
              char *scratch;

              /* add header */
              scratch = skb_put(skb, len);
              sprintf(scratch, "%s@%s", action_string, devpath);

              /* copy keys to our continuous event payload buffer */
              for (i = 0; i < env->envp_idx; i++) {
                   len = strlen(env->envp[i]) + 1;
                   scratch = skb_put(skb, len);
                   strcpy(scratch, env->envp[i]);
              }

              NETLINK_CB(skb).dst_group = 1;
              netlink_broadcast(uevent_sock, skb, 0, 1, GFP_KERNEL);
         }
     }
#endif

     /* call uevent_helper, usually only enabled during early boot */
     if (uevent_helper[0]) {
         char *argv [3];

         argv [0] = uevent_helper;
         argv [1] = (char *)subsystem;
         argv [2] = NULL;
         retval = add_uevent_var(env, "HOME=/");
         if (retval)
              goto exit;
         retval = add_uevent_var(env,
                        "PATH=/sbin:/bin:/usr/sbin:/usr/bin");
         if (retval)
              goto exit;

         call_usermodehelper(argv[0], argv, env->envp, UMH_WAIT_EXEC);
     }

exit:
     kfree(devpath);
     kfree(env);
     return retval;
}
[/cc]
忽略一段选择编译的代码，再后就是调用用户空间的hotplug了。添加最后两个环境变量：HOME和PATH。然后调用hotplug，以子系统名称为参数。
现在我们终于知道hotplug处理程序中的参数和环境变量是怎么来的了.^_^

使用完了kset，再调用kset_unregister()将其注销。这个函数很简单,请自行查阅代码.
为了印证一下上面的分析，写一个测试模块。如下：
[cc lang="c"]
#include <linux/device.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/string.h>
#include <linux/sysfs.h>
#include <linux/stat.h>
#include <linux/kobject.h>

MODULE_AUTHOR("eric xiao");
MODULE_LICENSE("Dual BSD/GPL");

int kset_filter(struct kset *kset, struct kobject *kobj);
const char *kset_name(struct kset *kset, struct kobject *kobj);
int kset_uevent(struct kset *kset, struct kobject *kobj,
                      struct kobj_uevent_env *env);

struct kset kset_p;
struct kset kset_c;

struct kset_uevent_ops uevent_ops =
{
        .filter = kset_filter,
        .name   = kset_name,
        .uevent = kset_uevent,
};

int kset_filter(struct kset *kset, struct kobject *kobj)
{
        printk("UEVENT: filter. kobj %s.\n",kobj->name);
        return 1;
}

const char *kset_name(struct kset *kset, struct kobject *kobj)
{
        static char buf[20];
        printk("UEVENT: name. kobj %s.\n",kobj->name);
        sprintf(buf,"%s","kset_test");
        return buf;
}

int kset_uevent(struct kset *kset, struct kobject *kobj,
                      struct kobj_uevent_env *env)
{
        int i = 0;
        printk("UEVENT: uevent. kobj %s.\n",kobj->name);

        while( i< env->envp_idx){
                printk("%s.\n",env->envp[i]);
                i++;
        }

        return 0;
}

int kset_test_init()
{
        printk("kset test init.\n");
        kobject_set_name(&kset_p.kobj,"kset_p");
        kset_p.uevent_ops = &uevent_ops;
        kset_register(&kset_p);

       kobject_set_name(&kset_c.kobj,"kset_c");
        kset_c.kobj.kset = &kset_p;
        kset_register(&kset_c);
        return 0;
}

int kset_test_exit()
{
        printk("kset test exit.\n");
        kset_unregister(&kset_p);
        kset_unregister(&kset_c);
        return 0;
}

module_init(kset_test_init);
module_exit(kset_test_exit);
[/cc]
在这里，定义并注册了二个kset，第二个kset的kobj->kset域指向第一个kset。这样，当第二个kset注册或者卸载的时候就会调用第一个kset中的uevent_ops的相关操作.
kset_p.uevent_ops->filter函数中，使其返回1.使其匹配成功。
在kset_p.uevent_ops->name中，使其返回的子系统名为引起事件的kobject的名称，即：kset_c.
最后在kset_p.uevent_ops->uevent中将环境变量全部打印出来。
下面是dmesg的输出结果：
[cc]
kset test init.
UEVENT: filter. kobj kset_c.
UEVENT: name. kobj kset_c.
UEVENT: uevent. kobj kset_c.
ACTION=add.
DEVPATH=/kset_p/kset_c.
SUBSYSTEM=kset_test.
[/cc]
输出结果跟我们的分析是吻合的，在这里，值得我们注意的是：注册一个kobject不会产生事件，只有注册kset才会。

<a name=4>

## 四：bus、device和device_driver
</a>
上面分析了kobject、kset、ktype，这三个结构联合起来一起构成了整个设备模型的基石。而bus、device、device_driver，则是基于kobject、kset、ktype之上的架构。在这里,总线、设备、驱动被有序的组合在一起。Bus、device、device_driver三者之间的关系如下图所示:
![080710160335](http://jarson.in/wp-content/uploads/2014/02/080710160335.jpg)
如上图所示，struct bus_type的p->drivers_kset指向注册在上面的驱动程序，它的p->device_kset上挂着注册在上面的设备。每次有一个新的设备注册到上面，都会去匹配右边的驱动，看是否能匹配上。如果匹配成功，则将设备结构的is_registerd域置为0，然后将设备添加到驱动的p->klist_devices域。同理,每注册一个驱动,都会去匹配左边的设备。如果匹配成功,将则设备加到驱动的p->klist_devices域，再将设备的is_registerd置为0。
这就是linux设备模型用来管理设备和驱动的基本架构，我们来跟踪一下代码来看下详细的操作。

<a name=4.1>

### 总线注册
</a>
注册一个总线的接口为bus_register()，我们照例分段分析:
[cc lang="c"]
int bus_register(struct bus_type *bus)
{
     int retval;
     struct bus_type_private *priv;

     priv = kzalloc(sizeof(struct bus_type_private), GFP_KERNEL);
     if (!priv)
         return -ENOMEM;

     priv->bus = bus;
     bus->p = priv;

     BLOCKING_INIT_NOTIFIER_HEAD(&priv->bus_notifier);

     retval = kobject_set_name(&priv->subsys.kobj, "%s", bus->name);
     if (retval)
         goto out;

     priv->subsys.kobj.kset = bus_kset;
     priv->subsys.kobj.ktype = &bus_ktype;
     priv->drivers_autoprobe = 1;

     retval = kset_register(&priv->subsys);
     if (retval)
         goto out;
[/cc]
首先,先为struct bus_type的私有区分配空间,然后将其和struct bus_type关联起来。由于struct bus_type也要在sysfs文件中表示一个节点,因此,它也内嵌一个kset的结构，这就是priv->subsys。

首先,它为这个kset的名称赋值为bus的名称，然后将priv->subsys.kobj.kset指向bus_kset，priv->subsys.kobj.ktype指向bus_ktype;然后调用kset_reqister()将priv->subsys注册。这里涉及到的接口都在之前分析过，注册过后，应该会在bus_kset所表示的目录下创建一个总线名称的目录，并且用户空间的hotplug应该会检测到一个add事件。我们来看一下bus_kset到底指向的是什么:
[cc lang="c"]bus_kset = kset_create_and_add("bus", &bus_uevent_ops, NULL);[/cc]
从此可以看出，这个bus_kset在sysfs中的结点就是/sys/bus，在这里注册的struct bus_types就会在/sys/bus/下面出现。
[cc lang="c"] 
     retval = bus_create_file(bus, &bus_attr_uevent);
     if (retval)
         goto bus_uevent_fail;
[/cc]
bus_create_file()就是在priv->subsys.kobj的这个kobject上建立一个普通属性的文件，这个文件的属性对应在bus_attr_uevent，读写操作对应在priv->subsys.ktype中，我们到后面才统一分析bus注册时候的文件创建。
[cc lang="c"] 
     priv->devices_kset = kset_create_and_add("devices", NULL,
                             &priv->subsys.kobj);
     if (!priv->devices_kset) {
         retval = -ENOMEM;
         goto bus_devices_fail;
     }

     priv->drivers_kset = kset_create_and_add("drivers", NULL,
                             &priv->subsys.kobj);
     if (!priv->drivers_kset) {
         retval = -ENOMEM;
         goto bus_drivers_fail;
     }

     klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
     klist_init(&priv->klist_drivers, NULL, NULL);
[/cc]
这段代码会在bus所在的目录下建立两个目录，分别为devices和drivers，并初始化挂载设备和驱动的链表。
[cc lang="c"] 
     retval = add_probe_files(bus);
     if (retval)
         goto bus_probe_files_fail;

     retval = bus_add_attrs(bus);
     if (retval)
         goto bus_attrs_fail;

     pr_debug("bus: '%s': registered\n", bus->name);
     return 0;
[/cc]
在这里,会为bus_attr_drivers_probe, bus_attr_drivers_autoprobe.注册bus_type中的属性建立文件
[cc lang="c"] 
bus_attrs_fail:
     remove_probe_files(bus);
bus_probe_files_fail:
     kset_unregister(bus->p->drivers_kset);
bus_drivers_fail:
     kset_unregister(bus->p->devices_kset);
bus_devices_fail:
     bus_remove_file(bus, &bus_attr_uevent);
bus_uevent_fail:
     kset_unregister(&bus->p->subsys);
     kfree(bus->p);
out:
     return retval;
}
[/cc]
这段代码为出错处理

这段代码中比较繁锁的就是bus_type对应目录下的属性文件建立,为了直观的说明,将属性文件的建立统一放到一起分析。从上面的代码中可以看,创建属性文件对应的属性分别为:bus_attr_uevent、bus_attr_drivers_probe、bus_attr_drivers_autoprobe。
分别定义如下:
[cc lang="c"]
static BUS_ATTR(uevent, S_IWUSR, NULL, bus_uevent_store);
static BUS_ATTR(drivers_probe, S_IWUSR, NULL, store_drivers_probe);
static BUS_ATTR(drivers_autoprobe, S_IWUSR | S_IRUGO,
         show_drivers_autoprobe, store_drivers_autoprobe);
[/cc]
BUS_ATTR定义如下:
[cc lang="c"]
#define BUS_ATTR(_name, _mode, _show, _store)  \
struct bus_attribute bus_attr_##_name = __ATTR(_name, _mode, _show, _store)
#define __ATTR(_name,_mode,_show,_store) { \
     .attr = {.name = __stringify(_name), .mode = _mode },   \
     .show    = _show,                    \
     .store   = _store,                   \
}
[/cc]
由此可见.上面这三个属性对应的名称为别为uevent、drivers_probe、drivers_autoprobe。也就是说，会在bus_types目录下生成三个文件，分别为uevent、probe、autoprobe。
根据之前的分析,我们知道在sysfs文件系统中,对普通属性文件的读写都会回溯到kobject->ktype->sysfs_ops中.在这里,注意到有:
[cc lang="c"]
     priv->subsys.kobj.kset = bus_kset;
     priv->subsys.kobj.ktype = &bus_ktype;
[/cc]
显然,读写操作就回溯到了bus_ktype中.定义如下:
[cc lang="c"]
static struct kobj_type bus_ktype = {
     .sysfs_ops    = &bus_sysfs_ops,
};
static struct sysfs_ops bus_sysfs_ops = {
     .show    = bus_attr_show,
     .store   = bus_attr_store,
};
[/cc]
show和store函数对应的代码为:
[cc lang="c"]
static ssize_t bus_attr_show(struct kobject *kobj, struct attribute *attr,
                   char *buf)
{
     struct bus_attribute *bus_attr = to_bus_attr(attr);
     struct bus_type_private *bus_priv = to_bus(kobj);
     ssize_t ret = 0;

     if (bus_attr->show)
         ret = bus_attr->show(bus_priv->bus, buf);
     return ret;
}

static ssize_t bus_attr_store(struct kobject *kobj, struct attribute *attr,
                    const char *buf, size_t count)
{
     struct bus_attribute *bus_attr = to_bus_attr(attr);
     struct bus_type_private *bus_priv = to_bus(kobj);
     ssize_t ret = 0;

     if (bus_attr->store)
         ret = bus_attr->store(bus_priv->bus, buf, count);
     return ret;
}
[/cc]
从代码可以看出.读写操作又会回溯到bus_attribute中的show和store中.在自定义结构里嵌入struct attribute,.然后再操作回溯到自定义结构中,这是一种比较高明的架构设计手法.

闲言少叙.我们对应看一下上面三个文件对应的最终操作:
uevent对应的读写操作为：NULL、bus_uevent_store。对于这个文件没有读操作，只有写操作，用cat 命令去查看这个文件的时候,可能会返回“设备不存在”的错误。
bus_uevent_store()代码如下:
[cc lang="c"]
static ssize_t bus_uevent_store(struct bus_type *bus,
                   const char *buf, size_t count)
{
     enum kobject_action action;

     if (kobject_action_type(buf, count, &action) == 0)
         kobject_uevent(&bus->p->subsys.kobj, action);
     return count;
}
[/cc]
从这里可以看到,可以在用户空间控制事件的发生,如echo add > event就会产生一个add的事件。

probe文件对应的读写操作为：NULL、store_drivers_probe。 store_drivers_probe()这个函数的代码涉及到struct device，等分析完struct device可以自行回过来看下这个函数的实现。实际上,这个函数是将用户输入的设备名称对应的设备与驱动匹配一次。

autoprobe文件对应的读写操作为show_drivers_autoprobe, store_drivers_autoprobe.对应读的代码为:
[cc lang="c"]
static ssize_t show_drivers_autoprobe(struct bus_type *bus, char *buf)
{
     return sprintf(buf, "%d\n", bus->p->drivers_autoprobe);
}
[/cc]
它将总线对应的drivers_autoprobe的值输出到用户空间，这个值为1时，自动将驱动与设备进行匹配，否则，反之。

写操作的代码如下:
[cc lang="c"]
static ssize_t store_drivers_autoprobe(struct bus_type *bus,
                          const char *buf, size_t count)
{
     if (buf[0] == '0')
         bus->p->drivers_autoprobe = 0;
     else
         bus->p->drivers_autoprobe = 1;
     return count;
}
[/cc]
写操作就会改变bus->p->drivers_autoprobe的值，就这样，通过sysfs就可以控制总线是否要进行自动匹配了。
从这里也可以看出，内核开发者的思维是何等的灵活。我们从sysfs中找个例子来印证一下：
[cc lang="bash"]cd  / sys/bus/usb[/cc]
用ls命令查看：
[cc lang="bash"]devices  drivers  drivers_autoprobe  drivers_probe  uevent[/cc]
与上面分析的相吻合

<a name=4.2>

### 设备注册
</a>
设备的注册接口为: device_register().
[cc lang="c"]
int device_register(struct device *dev)
{
     device_initialize(dev);
     return device_add(dev);
}
[/cc]
device_initialize()中有几个很重要的操作,如下:
[cc lang="c"]
void device_initialize(struct device *dev)
{
     dev->kobj.kset = devices_kset;
     kobject_init(&dev->kobj, &device_ktype);
     klist_init(&dev->klist_children, klist_children_get,
            klist_children_put);
     INIT_LIST_HEAD(&dev->dma_pools);
     INIT_LIST_HEAD(&dev->node);
     init_MUTEX(&dev->sem);
     spin_lock_init(&dev->devres_lock);
     INIT_LIST_HEAD(&dev->devres_head);
     device_init_wakeup(dev, 0);
     set_dev_node(dev, -1);
}
[/cc]
在这里,它为device的内嵌kobject指定了ktype和kset。device_kset的值如下:
[cc lang="c"]
devices_kset = kset_create_and_add("devices", &device_uevent_ops, NULL);
[/cc]
即对应sysfs中的/sys/devices。
device_ktype中对属性的读写操作同bus中的类似,被回溯到了struct device_attribute中的show和store。

接着往下看device_add()的实现.这个函数比较长,分段分析如下:
[cc lang="c"]
int device_add(struct device *dev)
{
     struct device *parent = NULL;
     struct class_interface *class_intf;
     int error;

     dev = get_device(dev);
     if (!dev || !strlen(dev->bus_id)) {
         error = -EINVAL;
         goto Done;
     }

     pr_debug("device: '%s': %s\n", dev->bus_id, __FUNCTION__);

     parent = get_device(dev->parent);
     setup_parent(dev, parent);

     /* first, register with generic layer. */
     error = kobject_add(&dev->kobj, dev->kobj.parent, "%s", dev->bus_id);
     if (error)
         goto Error;
[/cc]
     如果注册device的时候,没有指定父结点,在kobject_add将会在/sys/device/下建立相同名称的目录
[cc lang="c"]
     /* notify platform of device entry */
     if (platform_notify)
         platform_notify(dev);

     /* notify clients of device entry (new way) */
     if (dev->bus)
         blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
                            BUS_NOTIFY_ADD_DEVICE, dev);
[/cc]
忽略notify部份,这部份不会影响本函数的流程
[cc lang="c"] 
     error = device_create_file(dev, &uevent_attr);
     if (error)
         goto attrError;

     if (MAJOR(dev->devt)) {
         error = device_create_file(dev, &devt_attr);
         if (error)
              goto ueventattrError;
     }
[/cc]
建立属性为uevent_attr的属性文件,如果device中指定了设备号,则建立属性为devt_attr的属性文件
[cc lang="c"] 
     error = device_add_class_symlinks(dev);
     if (error)
         goto SymlinkError;
     error = device_add_attrs(dev);
     if (error)
         goto AttrsError;
     error = dpm_sysfs_add(dev);
     if (error)
         goto PMError;
     device_pm_add(dev);
[/cc]
在这里,不打算讨论class的部份,dpm、pm是选择编译部份,不讨论. device_add_attrs中涉及到了group的部分,暂不讨论
[cc lang="c"]
     error = bus_add_device(dev);
     if (error)
         goto BusError;
     kobject_uevent(&dev->kobj, KOBJ_ADD);
     bus_attach_device(dev);
     if (parent)
         klist_add_tail(&dev->knode_parent, &parent->klist_children);

     if (dev->class) {
         down(&dev->class->sem);
         /* tie the class to the device */
         list_add_tail(&dev->node, &dev->class->devices);

         /* notify any interfaces that the device is here */
         list_for_each_entry(class_intf, &dev->class->interfaces, node)
              if (class_intf->add_dev)
                   class_intf->add_dev(dev, class_intf);
         up(&dev->class->sem);
     }
[/cc]
bus_add_device()在对应总线代表目录的device目录下创建几个到device的链接，然后调用kobject_uevent()产生一个add事件，再调用bus_attach_device()去匹配已经注册到总线的驱动程序。全部做完之后，将设备挂到父结点的子链表。
[cc lang="c"]
 Done:
     put_device(dev);
     return error;
 BusError:
     device_pm_remove(dev);
 PMError:
     if (dev->bus)
         blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
                            BUS_NOTIFY_DEL_DEVICE, dev);
     device_remove_attrs(dev);
 AttrsError:
     device_remove_class_symlinks(dev);
 SymlinkError:
     if (MAJOR(dev->devt))
         device_remove_file(dev, &devt_attr);
 ueventattrError:
     device_remove_file(dev, &uevent_attr);
 attrError:
     kobject_uevent(&dev->kobj, KOBJ_REMOVE);
     kobject_del(&dev->kobj);
 Error:
     cleanup_device_parent(dev);
     if (parent)
         put_device(parent);
     goto Done;
}
[/cc]
出错处理部份.

bus_attach_device()是一个很重要的函数。它将设备自动与挂在总线上面的驱动进行匹配。代码如下：
[cc lang="c"]
void bus_attach_device(struct device *dev)
{
     struct bus_type *bus = dev->bus;
     int ret = 0;

     if (bus) {
         dev->is_registered = 1;
         if (bus->p->drivers_autoprobe)
              ret = device_attach(dev);
         WARN_ON(ret < 0);
         if (ret >= 0)
              klist_add_tail(&dev->knode_bus, &bus->p->klist_devices);
         else
              dev->is_registered = 0;
     }
}
[/cc]
从上面的代码我们可以看出。只有在bus->p->drivers_autoprobe为1的情况下，才会去自己匹配。这也就是bus目录下的drivers_probe 文件的作用.然后，将设备挂到总线的设备链表。
device_attach()代码如下：
[cc lang="c"]
int device_attach(struct device *dev)
{
     int ret = 0;

     down(&dev->sem);
     if (dev->driver) {
         ret = device_bind_driver(dev);
         if (ret == 0)
              ret = 1;
         else {
              dev->driver = NULL;
              ret = 0;
         }
     } else {
         ret = bus_for_each_drv(dev->bus, NULL, dev, __device_attach);
     }
     up(&dev->sem);
     return ret;
}
[/cc]
对于设备自己已经指定驱动的情况，只需要将其直接和驱动绑定即可。如果没有指定驱动,就匹配总线之上的驱动，这是在
[cc lang="c"]bus_for_each_drv(dev->bus, NULL, dev, __device_attach);[/cc]
完成的。代码如下：
[cc lang="c"]
int bus_for_each_drv(struct bus_type *bus, struct device_driver *start,
              void *data, int (*fn)(struct device_driver *, void *))
{
     struct klist_iter i;
     struct device_driver *drv;
     int error = 0;

     if (!bus)
         return -EINVAL;

     klist_iter_init_node(&bus->p->klist_drivers, &i,
                   start ? &start->p->knode_bus : NULL);
     while ((drv = next_driver(&i)) && !error)
         error = fn(drv, data);
     klist_iter_exit(&i);
     return error;
}
[/cc]
很明显，这个函数就是遍历总线之上的驱动。每遍历一个驱动就调用一次回调函数进行判断，如果回调函数返回不为0，就说明匹配已经成功了，不需要再匹配剩余的，退出。在这里调用的回调函数是__device_attach()，在这里，完成了设备与驱动匹配的最核心的动作。代码如下：
[cc lang="c"]
static int __device_attach(struct device_driver *drv, void *data)
{
     struct device *dev = data;
     return driver_probe_device(drv, dev);
}
[/cc]
转到driver_probe_device():
[cc lang="c"]
int driver_probe_device(struct device_driver *drv, struct device *dev)
{
     int ret = 0;

     if (!device_is_registered(dev))
         return -ENODEV;
     if (drv->bus->match && !drv->bus->match(dev, drv))
         goto done;

     pr_debug("bus: '%s': %s: matched device %s with driver %s\n",
          drv->bus->name, __FUNCTION__, dev->bus_id, drv->name);

     ret = really_probe(dev, drv);

done:
     return ret;
}
[/cc]
如果设备没有注册到总线之上，即dev->is_registered不为1， 就直接返回。然后，再调用总线的match()函数进行匹配。如果match()函数返回0，说明匹配失败，那退出此函数。如果match函数返回1，说明初步的检查已经通过了，可以进入really_probe()再进行细致的检查。如果匹配成功，这个函数会返回1。此函数比较长而且比较重要，分段列出代码：
[cc lang="c"]
static int really_probe(struct device *dev, struct device_driver *drv)
{
     int ret = 0;

     atomic_inc(&probe_count);
     pr_debug("bus: '%s': %s: probing driver %s with device %s\n",
          drv->bus->name, __FUNCTION__, drv->name, dev->bus_id);
     WARN_ON(!list_empty(&dev->devres_head));

     dev->driver = drv;
     if (driver_sysfs_add(dev)) {
         printk(KERN_ERR "%s: driver_sysfs_add(%s) failed\n",
              __FUNCTION__, dev->bus_id);
         goto probe_failed;
     }
[/cc]
先假设驱动和设备是匹配的，为设备结构设置驱动成员，使其指向匹配的驱动，然后再调用driver_sysfs_add()建立几个符号链接。这几个链接分别为：
1、在驱动目录下建立一个到设备的同名链接；
2、在设备目录下建立一个名为driver到驱动的链接。
[cc lang="c"] 
     if (dev->bus->probe) {
         ret = dev->bus->probe(dev);
         if (ret)
              goto probe_failed;
     } else if (drv->probe) {
         ret = drv->probe(dev);
         if (ret)
              goto probe_failed;
     }
[/cc]
然后，再调用总线的probe函数，如果总线的此函数不存在，就会调用驱动的probe函数。如果匹配成功，返回0；如果不成功，就会跳转到probe_failed。
[cc lang="c"] 
     driver_bound(dev);
     ret = 1;
     pr_debug("bus: '%s': %s: bound device %s to driver %s\n",
          drv->bus->name, __FUNCTION__, dev->bus_id, drv->name);
     goto done;
[/cc]
到这里，设备和驱动已经匹配成功，调用driver_bound()将其关联起来，在这个函数里会将设备加至驱动的设备链表，这在我们之前分析bus、device、driver中分析到的。相关的代码如下示：
[cc lang="c"]
klist_add_tail(&dev->knode_driver, &dev->driver->p->klist_devices);
[/cc]
至此，这个匹配过程已经圆满结束了，返回1
[cc lang="c"] 
probe_failed:
     devres_release_all(dev);
     driver_sysfs_remove(dev);
     dev->driver = NULL;

     if (ret != -ENODEV && ret != -ENXIO) {
         /* driver matched but the probe failed */
         printk(KERN_WARNING
                "%s: probe of %s failed with error %d\n",
                drv->name, dev->bus_id, ret);
     }
     /*
      * Ignore errors returned by ->probe so that the next driver can try
      * its luck.
      */
     ret = 0;
[/cc]
这里是匹配不成功的处理，在这里，删除之前建立的几个链接文件，然后将设备的driver域置空。
[cc lang="c"]
done:
     atomic_dec(&probe_count);
     wake_up(&probe_waitqueue);
     return ret;
}
[/cc]
从上面的分析可以看到，对应创建的属性文件分别为：uevent_attr devt_attr。它们的定义如下：
[cc lang="c"]
static struct device_attribute uevent_attr =
     __ATTR(uevent, S_IRUGO | S_IWUSR, show_uevent, store_uevent);
static struct device_attribute devt_attr =
     __ATTR(dev, S_IRUGO, show_dev, NULL);
[/cc]
uevent_attr对应的读写函数分别为：show_uevent、store_uevent。先分析读操作。它的代码如下：
[cc lang="c"]
static ssize_t show_uevent(struct device *dev, struct device_attribute *attr,
                 char *buf)
{
     struct kobject *top_kobj;
     struct kset *kset;
     struct kobj_uevent_env *env = NULL;
     int i;
     size_t count = 0;
     int retval;

     /* search the kset, the device belongs to */
     top_kobj = &dev->kobj;
     while (!top_kobj->kset && top_kobj->parent)
         top_kobj = top_kobj->parent;
     if (!top_kobj->kset)
         goto out;

     kset = top_kobj->kset;
     if (!kset->uevent_ops || !kset->uevent_ops->uevent)
         goto out;

     /* respect filter */
     if (kset->uevent_ops && kset->uevent_ops->filter)
         if (!kset->uevent_ops->filter(kset, &dev->kobj))
              goto out;

     env = kzalloc(sizeof(struct kobj_uevent_env), GFP_KERNEL);
     if (!env)
         return -ENOMEM;

     /* let the kset specific function add its keys */
     retval = kset->uevent_ops->uevent(kset, &dev->kobj, env);
     if (retval)
         goto out;

     /* copy keys to file */
     for (i = 0; i < env->envp_idx; i++)
         count += sprintf(&buf[count], "%s\n", env->envp[i]);
out:
     kfree(env);
     return count;
}
[/cc]
从代码可以看出，这里会显示出由设备对应的kset，也就是由devices_kset所产生的环境变量。例如，在shell中输入如下指令：
[cc lang="bash"]cat /sys/devices/LNXSYSTM:00/uevent[/cc]
输出结果如下：
[cc lang="bash"]
PHYSDEVBUS=acpi
MODALIAS=acpi:LNXSYSTM:
[/cc]
这就是由devices_kset所添加的环境变量

写操作对应的代码如下：
[cc lang="c"]
static ssize_t store_uevent(struct device *dev, struct device_attribute *attr,
                  const char *buf, size_t count)
{
     enum kobject_action action;

     if (kobject_action_type(buf, count, &action) == 0) {
         kobject_uevent(&dev->kobj, action);
         goto out;
     }

     dev_err(dev, "uevent: unsupported action-string; this will "
              "be ignored in a future kernel version\n");
     kobject_uevent(&dev->kobj, KOBJ_ADD);
out:
     return count;
}
[/cc]
从上面的代码可以看出，这个文件的作用是输入一个字符字串，如果字符不合法，就会默认产生一个add事件。

devt_attr对应的读写函数为show_dev、NULL。写函数为空，也就是说这个属性文件不允许写，只允许读。读操作的代码如下示：
[cc lang="c"]
static ssize_t show_dev(struct device *dev, struct device_attribute *attr,
              char *buf)
{
     return print_dev_t(buf, dev->devt);
}
[/cc]
也就是说，会将设备号显示出来.

<a name=4.3>

### 驱动注册
</a>
分析完了bus、device，再接着分析driver。这里我们要分析的最后一个元素了，耐着性子往下看，快要完了^_^
驱动注册的接口为：driver_register()。代码如下：
[cc lang="c"]
int driver_register(struct device_driver *drv)
{
     int ret;

     if ((drv->bus->probe && drv->probe) ||
         (drv->bus->remove && drv->remove) ||
         (drv->bus->shutdown && drv->shutdown))
         printk(KERN_WARNING "Driver '%s' needs updating - please use "
              "bus_type methods\n", drv->name);
     ret = bus_add_driver(drv);
     if (ret)
         return ret;
     ret = driver_add_groups(drv, drv->groups);
     if (ret)
         bus_remove_driver(drv);
     return ret;
}
[/cc]
如果设备与总线定义了相同的成员的函数，内核是优先使用bus中定义的，这一点我们在分析device注册的时候已经分析过。所以，这里打印出警告信息，用来提醒代码编写者。在这里，忽略有关group的东西，剩余的便只剩下bus_add_driver()。代码如下：
[cc lang="c"]
int bus_add_driver(struct device_driver *drv)
{
     struct bus_type *bus;
     struct driver_private *priv;
     int error = 0;

     bus = bus_get(drv->bus);
     if (!bus)
         return -EINVAL;

     pr_debug("bus: '%s': add driver %s\n", bus->name, drv->name);

     priv = kzalloc(sizeof(*priv), GFP_KERNEL);
     if (!priv) {
         error = -ENOMEM;
         goto out_put_bus;
     }
     klist_init(&priv->klist_devices, NULL, NULL);
     priv->driver = drv;
     drv->p = priv;
     priv->kobj.kset = bus->p->drivers_kset;
     error = kobject_init_and_add(&priv->kobj, &driver_ktype, NULL,
                        "%s", drv->name);
[/cc]
初始化驱动的driver_private域，使其内嵌的kobject的kset指bus中的drivers_kset，这样，这个内嵌的kobject所生成的目录就会存在于bus对应目录的driver目录之下。这里还要注意的是,为内嵌kobject指定的ktype是driver_ktype，属性文件的读写操作都回回溯到struct driver_attribute中，这在之后再分析。
[cc lang="c"] 
     if (error)
         goto out_unregister;

     if (drv->bus->p->drivers_autoprobe) {
         error = driver_attach(drv);
         if (error)
              goto out_unregister;
     }
     klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);

     module_add_driver(drv->owner, drv);
[/cc]
如果总线允许自动进行匹配，就会调用driver_attach()进行这个自己匹配过程。这个函数跟我们在上面分析的device自动匹配过程是一样的，请自行分析。最后，将驱动挂到bus对应的驱动链表。
[cc lang="c"] 
     error = driver_create_file(drv, &driver_attr_uevent);
     if (error) {
         printk(KERN_ERR "%s: uevent attr (%s) failed\n",
              __FUNCTION__, drv->name);
     }
[/cc]
生成一个属性为driver_attr_uevent的属性文件
[cc lang="c"]
     error = driver_add_attrs(bus, drv);
     if (error) {
         /* How the hell do we get out of this pickle? Give up */
         printk(KERN_ERR "%s: driver_add_attrs(%s) failed\n",
              __FUNCTION__, drv->name);
     }
[/cc]
为bus中的driver属性生成属性文件
[cc lang="c"]
     error = add_bind_files(drv);
     if (error) {
         /* Ditto */
         printk(KERN_ERR "%s: add_bind_files(%s) failed\n",
              __FUNCTION__, drv->name);
     }
[/cc]
生成属性为driver_attr_unbind和driver_attr_bind的属性文件
[cc lang="c"] 
     kobject_uevent(&priv->kobj, KOBJ_ADD);
[/cc]
生成一个add事件
[cc lang="c"]
     return error;
out_unregister:
     kobject_put(&priv->kobj);
out_put_bus:
     bus_put(bus);
     return error;
}
[/cc]
总的来说，这个函数比较简单，其中涉及到的子函数大部份都在之前分析过。我们接下来分析一下，它所创建的几个属性文件的含义。
如上所述，在这里会创建三个属性文件，对应属性分别为：driver_attr_uevent，driver_attr_unbind，driver_attr_bind。这几个属性的定义如下：
[cc lang="c"]
static DRIVER_ATTR(uevent, S_IWUSR, NULL, driver_uevent_store);
static DRIVER_ATTR(unbind, S_IWUSR, NULL, driver_unbind);
static DRIVER_ATTR(bind, S_IWUSR, NULL, driver_bind);
[/cc]
DRIVER_ATTR宏的定义如下：
[cc lang="c"]
#define DRIVER_ATTR(_name, _mode, _show, _store)   \
struct driver_attribute driver_attr_##_name =      \
     __ATTR(_name, _mode, _show, _store)
[/cc] 
对于driver_attr_uevent，它的读写函数分别为：NULL，driver_uevent_store。也就是说这个文件只允许写，不允许读操作。写操作的代码如下示：
[cc lang="c"]
static ssize_t driver_uevent_store(struct device_driver *drv,
                      const char *buf, size_t count)
{
     enum kobject_action action;

     if (kobject_action_type(buf, count, &action) == 0)
         kobject_uevent(&drv->p->kobj, action);
     return count;
}
[/cc]
很明显，这是一个手动产生事件的过程。用户可间可以写事件到这个文件来产生事件。
对于driver_unbind，它的读写函数分别为：NULL，driver_unbind。这个文件也是不允许读的，写操作代码如下：
[cc lang="c"]
static ssize_t driver_unbind(struct device_driver *drv,
                   const char *buf, size_t count)
{
     struct bus_type *bus = bus_get(drv->bus);
     struct device *dev;
     int err = -ENODEV;

     dev = bus_find_device_by_name(bus, NULL, buf);
     if (dev && dev->driver == drv) {
         if (dev->parent)   /* Needed for USB */
              down(&dev->parent->sem);
         device_release_driver(dev);
         if (dev->parent)
              up(&dev->parent->sem);
         err = count;
     }
     put_device(dev);
     bus_put(bus);
     return err;
}
[/cc]
从上面的代码可以看出，写入文件的是一个设备名称，这个函数对应操作是将这个设备与驱动的绑定分离开来。

driver_attr_bind属性对应的读写函数分别为：NULL，driver_attr_bind。 即也是不允许写的。从字面意思和上面分析的driver_attr_unbind操作代码来看，这个属性对应的写函数应该是将写入的设备文件与此驱动绑定起来。我们来看下代码，以证实我们的猜测。代码如下：
[cc lang="c"]
static ssize_t driver_bind(struct device_driver *drv,
                 const char *buf, size_t count)
{
     struct bus_type *bus = bus_get(drv->bus);
     struct device *dev;
     int err = -ENODEV;

     dev = bus_find_device_by_name(bus, NULL, buf);
     if (dev && dev->driver == NULL) {
         if (dev->parent)   /* Needed for USB */
              down(&dev->parent->sem);
         down(&dev->sem);
         err = driver_probe_device(drv, dev);
         up(&dev->sem);
         if (dev->parent)
              up(&dev->parent->sem);

         if (err > 0) {
              /* success */
              err = count;
         } else if (err == 0) {
              /* driver didn't accept device */
              err = -ENODEV;
         }
     }
     put_device(dev);
     bus_put(bus);
     return err;
}
[/cc]
果然，和我们猜测的是一样的。

<a name=5>

## 五：小结
</a>
在这一节里，分析了设备模型中的最底层的元素和他们之间的关系，也分析了它们建立的几个属性文件的含义。到这里，我们已经可以自己写驱动架构代码了 ^_^
