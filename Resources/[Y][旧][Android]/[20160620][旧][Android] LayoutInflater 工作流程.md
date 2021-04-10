## 备注
**原发表于2016.06.20，资料已过时，仅作备份，谨慎参考**

## 前言
感觉很长时间没写文章了，这个星期因为回家和处理项目问题，还是花了很多时间的。虽然知道很多东西如果只是看一下用一次，很快就会遗忘，但认认真真地做输出还是需要一定恒心的。

这次写 LayoutInflater 的工作流程，是由于小组一位成员在调用inflate 方法时，没有传入 parent 参数导致生成的布局宽高失效的问题。

这里先说原因，是因为如果 inflate 的 View，没有包含在某个 Viewgroup 下，也没有传入 parent 参数，那么他的 layout_width 等属性就会失效，这些属性是需要 View 处在某个布局下才能生效。

## 使用
### 获取 LayoutInflater
通过 LayoutInflater layoutInflater = LayoutInflater.from(context) 就可以获取到 LayoutInflater。

如果调用 View.inflate 也是先会获取 LayoutInflater，再使用 LayoutInflater 去加载布局。获取 LayoutInflater 的源码如下：

    public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }

可以看到，我们其实也可以自己去调用 getSystemService 来获取 LayoutInflater。
### 加载布局
接下来，使用 layoutInflater.inflate 方法即可加载相应布局，有四个重载方法如下图：
![](http://upload-images.jianshu.io/upload_images/1903766-b7b53228436e011e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中 parser 参数是一个描述 View 层次的 XML 文件，在 Android 中我们就是用 XML 来描述布局的，所以直接传入布局的 int 值就可以了。

另外两个需要注意的参数是 root 和 attachToRoot，root 是指定了本次加载布局的父容器，attachToRoot 则表示是否将本次加载的内容添加到父容器中。

我们常常会碰到在 adapter 中，设置 attachToRoot 为 true 时会报错，是因为 adapter 会自己将加载的布局添加到父容器里，如果自己设置的话，就会导致重复添加了。

## 工作流程
所有的 inflate 到最后都会返回到下面这个方法中来进行布局加载：

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
                ...

                final String name = parser.getName();
                

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

                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                   

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);

                    

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

代码较多，我去掉了有关 DEBUG 的部分，同时最后异常处理的部分也可以略过不看，那么逻辑就比较清晰了。LayoutInflater 使用 XmlPullParser 来解析布局文件，首先根据我们传入的布局参数创建根布局：

    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

    ViewGroup.LayoutParams params = null;

这个 params 参数是用来为 temp 即加载的根布局进行属性参数设置的，他有以下三种情况：

1. ViewGroup root 参数为 null，则返回加载的布局不做其他设置
2. root != null && attachToRoot == null，则使用父布局的参数对 temp 进行设置

        params = root.generateLayoutParams(attrs);
        temp.setLayoutParams(params);
3. root != null && attachToRoot != null，则将加载的布局添加到父布局，再返回父布局：

        root.addView(temp, params);

接下来还会遍历加载根布局的子元素：
    
    rInflateChildren(parser, temp, attrs, true);

这样就完成了一次布局的加载，具体是如何加载的，就得去学习 View 的工作原理了。

## 参考资料
这里也提示一下，你看到的资料不一定都是最正确的，尽量接受多方的信息，比如一篇博客看完后，最好是能把评论也翻看一遍。

[Android LayoutInflater原理分析，带你一步步深入了解View(一)](http://blog.csdn.net/guolin_blog/article/details/12921889)