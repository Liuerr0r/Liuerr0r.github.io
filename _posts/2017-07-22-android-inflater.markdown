---
layout:     post
title:      "Layout.inflat()中的那些坑"
subtitle:   "学习总结"
date:       2017-07-22 12:00:00
author:     "溜大虾"
header-img: "img/bg-inflater.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Android
    

---

LayoutInflater.inflate()这个方法，大家一定很熟悉——在给fragment添加布局文件，或者在RecyclerView的Adapter中为item添加布局时，都会用到。inflate()这个方法需要最多三个参数：resource，root，以及attachToRoot。参考源码，就知道了这里的resource是你具体要添加的那个布局文件，root是布局的根参数，那么attachToRoot是什么意思呢？什么时候为true，什么时候为false？

## 三个参数的关系

参见官方文档，对这三个参数的介绍是：被填充的层是否应该附在root参数内部？如果是false，root参数只适用于为xml根元素View创建正确的LayoutParams的子类。

什么意思呢？就是说，如果attachToRoot为true，那么resource指定的布局文件就会依附于root指定的ViewGroup，然后这个方法就会返回root，否则，只会将resource指定的布局文件填充并将其返回，具体可以参考源码：

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {

        synchronized (mConstructorArgs) {

            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;

            final AttributeSet attrs = Xml.asAttributeSet(parser);

            Context lastContext = (Context) mConstructorArgs[0];

            mConstructorArgs[0] = inflaterContext;

            View result = root;

            try {

                // Look for the root node.

                int type;

                while ((type = parser.next()) != XmlPullParser.START_TAG &&

                        type != XmlPullParser.END_DOCUMENT) {

                    // Empty
                if (type != XmlPullParser.START_TAG) {

                    throw new InflateException(parser.getPositionDescription()

                            + ": No start tag found!");

                }

                final String name = parser.getName();

      

                if (DEBUG) {

                    System.out.println("**************************");

                    System.out.println("Creating root view: "

                            + name);

                    System.out.println("**************************");

                }



                if (TAG_MERGE.equals(name)) {

                    if (root == null || !attachToRoot) {

                        throw new InflateException("<merge /> can be used only with a valid "

                                + "ViewGroup root and attachToRoot=true");

                    }



                    rInflate(parser, root, inflaterContext, attrs, false);

                } else {

                    // Temp is the root view that was found in the xml

                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);



                    ViewGroup.LayoutParams params = null;



                    if (root != null) {

                        if (DEBUG) {

                            System.out.println("Creating params from root: " +

                                    root);

                        }

                        // Create layout params that match root, if supplied

                        params = root.generateLayoutParams(attrs);

                        if (!attachToRoot) {

                            // Set the layout params for temp if we are not

                            // attaching. (If we are, we use addView, below)

                            temp.setLayoutParams(params);

                        }

                    }



                    if (DEBUG) {

                        System.out.println("-----> start inflating children");

                    }



                    // Inflate all children under temp against its context.

                    rInflateChildren(parser, temp, attrs, true);



                    if (DEBUG) {

                        System.out.println("-----> done inflating children");

                    }



                    // We are supposed to attach all the views we found (int temp)

                    // to root. Do that now.

                    if (root != null && attachToRoot) {

                        root.addView(temp, params);

                    }



                    // Decide whether to return the root that was passed in or the

                    // top view found in xml.

                    if (root == null || !attachToRoot) {

                        result = temp;

                    }

                }



            } catch (XmlPullParserException e) {

                InflateException ex = new InflateException(e.getMessage());

                ex.initCause(e);

                throw ex;

            } catch (Exception e) {

                InflateException ex = new InflateException(

                        parser.getPositionDescription()

                                + ": " + e.getMessage());

                ex.initCause(e);

                throw ex;

            } finally {

                // Don't retain static reference on context.

                mConstructorArgs[0] = lastContext;

                mConstructorArgs[1] = null;

            }



            Trace.traceEnd(Trace.TRACE_TAG_VIEW);



            return result;

        }

    }



```

总结一下，就是：

- 若attachToRoot为true且root不为null，则调用root.addView()方法
- 若root为null，或者attachToRoot为false，则直接将temp赋于result(temp是通过root构造的，result就是root)

## 何时为true，何时为false?

就拿我们的Adapter来说吧，在创建item布局时，有下列几种情况：

- inflate([R.layout.xxx](http://r.layout.xxx/),null);
- inflate([R.layout.xxx](http://r.layout.xxx/),parent,false);
- inflate([R.layout.xxx](http://r.layout.xxx/),parent,true);

那么就讲一下这三种情况把。

首先，inflate([R.layout.xxx](http://r.layout.xxx/),null) 。这是最简单的写法，这样生成的布局就是根据R.layout.xxx返回的View。要知道，这个布局文件中的宽高属性都是相当于父布局而言的。由于没有指定parent，所以他的宽高属性就失效了，因此不管你怎么改宽高属性，都无法按你想象的那样显示。

然后，inflate([R.layout.xxx](http://r.layout.xxx/),parent,false)。相较于前者，这里加了父布局，不管后面是true还是false，由于有了parent，布局文件的宽高属性是有依靠了，这时候显示的宽高样式就是布局文件中的那样了。

最后，inflate([R.layout.xxx](http://r.layout.xxx/),parent,true)。这样……等等，报错了？？？哦，不要惊奇，分析一下原因：首先，有了parent，所以可以正确处理布局文件的宽高属性。然后，既然attachToRoot为true，那么根据上面的源码就会知道，这里会调用root的addView方法。而如果root是listView等，由于他们是继承自AdapterView的，看看AdapterView的addView方法： 

```
@Override

    public void addView(View child) {

        throw new UnsupportedOperationException("addView(View) is not supported in AdapterView");

    }


```

不资磁啊，那好吧，如果换成RecyclerView呢？还是报错了，看看源码：

```
if (child.getParent() != null) {

            throw new IllegalStateException("The specified child already has a parent. " +

                    "You must call removeView() on the child's parent first.");

        }


```

现在知道了吧，adpater里面不要用true。那么什么时候用true呢？答案是fragment。在为fragment创建布局时，如果为true，那么这个布局文件就会被添加到父activity中盛放fragment的布局中。