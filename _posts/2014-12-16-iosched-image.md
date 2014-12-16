    Copyright 2014 Google Inc. All rights reserved.

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.


# IOSched中的图片尺寸改变

最小化数据传输对一个移动app来说是一个重要的要求，尤其是在一个网络请求受限制的环境，如一个安排满满的大会。但是，另一方面，丰富的 session 和 speaker 图片可以使如IOSched这样的 app 更易操作也更加美观，甚至可以帮助参会者识别speakers。

为了达到一个平衡，我们引入了一个根据不同的尺寸(维持宽高比，根据宽度决定)去获取图片的机制。一个服务器端的脚本压缩，优化，根据指定的要求重新生成新的尺寸图片，保存到根据原始URL生成的可变的变量中。在客户端，一个图片 loader 识别图片URL中的特殊规则，根据图片容器的宽度请求最合适的图片。


## 去耦合

为了避免图片尺寸重新生成机制和Android app的高耦合，图片URL的格式中包含了可取的内容信息，允许服务器动态改变图片内容而不需要改变Android客户端。同样的，如果不理解这种特定的URL格式，比如旧版本的app，URL没有任何特殊则处理指向原始尺寸的图片。


## 自适应的图片URL格式

任何符合下面(简化)正则表达式的图片URL都视为一个自适应的图片URL：`.*__w(-\d+)+__.*`

例如：

<pre>
myserver.com/images/<b>__w-200-400-600-800-1000__</b>/session1.jpg
</pre>

正如前面所说，调用这个URL获取的是全尺寸的图片。要获取限制宽度为200px的图片，替换`__w-...__`为"**w200**"：

<pre>
myserver.com/images/<b>w200</b>/session1.jpg
</pre>

400px宽度：

<pre>
myserver.com/images/<b>w400</b>/session1.jpg
</pre>

你应该已经知道了，可取的图片尺寸就是限定在`__w-`和`__`之间的数字。URL

<pre>
myserver.com/images/<b>__w-200-400-600-800-1000__</b>/session1.jpg
</pre>

意味着该图片可以通过以下任意一种URL获取：

URL | Image size
--- | ----------
myserver.com/images/**\_\_w-200-400-600-800-1000\_\_**/session1.jpg | original
myserver.com/images/w200/session1.jpg | 200px wide
myserver.com/images/w400/session1.jpg | 400px wide
myserver.com/images/w600/session1.jpg | 600px wide
myserver.com/images/w800/session1.jpg | 800px wide
myserver.com/images/w1000/session1.jpg | 1000px wide


## 服务器端

我们决定线下处理图片然后以静态方式提供，目的是增进可扩展性和可靠性。动态服务亦可。

每一个新的或改变的session的图片都被转化为JPG格式，以0.8压缩比压缩，重新生成合适的尺寸图片然后存储到云服务器。这些都是通过一个简单的作为定时计划任务的Bash脚本实现的。其结果是一个目录结构如下：

    /__w-200-400-600__/session1.jpg
    /__w-200-400-600__/session2.jpg
    /__w-200-400-600__/session3.jpg
    /w200/session1.jpg
    /w200/session2.jpg
    /w200/session3.jpg
    /w400/session1.jpg
    /w400/session2.jpg
    /w400/session3.jpg
    /w600/session1.jpg
    /w600/session2.jpg
    /w600/session3.jpg

我们保持这些目录结构到Google Cloud Storage，每当一个session或speaker更新了它的图片都会更新它。

# Android app端

IOSched中的每个图片都是通过一个库 [Glide](https://github.com/bumptech/glide) 加载的，它可以处理异步加载和缓存。我们使用一个自定义的 [ImageLoader](https://github.com/google/iosched/blob/master/android/src/main/java/com/google/samples/apps/iosched/util/ImageLoader.java) 扩展了Glide，可以适配我们的URL格式。作为详细说明，内部类 VariableWidthImageLoader 包含了下面的代码段：

```Java
private static final Pattern PATTERN =
      Pattern.compile("__w-((?:-?\\d+)+)__");

@Override
protected String getUrl(String model, int width, int height) {
    Matcher m = PATTERN.matcher(model);
    int bestBucket = 0;
    if (m.find()) {
        String[] found = m.group(1).split("-");
        for (String bucketStr : found) {
            bestBucket = Integer.parseInt(bucketStr);
            if (bestBucket >= width) {
                // the best bucket is the first immediately
                // bigger than the requested width
                break;
            }
        }
        if (bestBucket > 0) {
            model = m.replaceFirst("w"+bestBucket);
        }
    }
    return model;
}
```

逻辑很简单：每当一个图片要加载时，这段代码检查图片URL是否符合特殊正则表达式。如果是，它会被分解成一块块来遍历最合适的可用图片宽度，也就是可选的图片中第一个大于或等于图片容器宽度的图片。URL会被改变为 w<bucket> 版本，异步加载继续。

