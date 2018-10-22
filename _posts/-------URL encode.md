最近因项目需要，需重写网络组件。在跟朋友闲聊，说到一些iOS开发者编写URL编码方法很不严谨，某些写法其实是有问题时，几个朋友都表示非常诧异并几度辩驳。本人表示有点小心惊，在网上搜索时还真的很少有另外的写法。在此以自己的一些理解和经验，做一下URL编码的普及，希望对大家有所帮助，有问题也请不吝赐教。 （参考RFC1738，3986，6874，7320）

　　**一、了解URL编码以及编码时机、运算过程**

　　URI包括URL和URN，常用说法URL encode实际是遵循URI的相关文件。在URI的最初设计时，希望能通过书面转录，比如写在餐巾纸上告诉另外一人，因此URI的构成字符必须是可写的ASCII字符。在这些可书写的字符里，由于一些字符在不同操作系统的编码有不同的解析，被包含在Unsafe characters（图2）之中，要格外注意。最后，在URI的构成字符中，最安全的方案是正确使用Reserved Characters （图1）和 Unreserved Characters （图1）的并集。

　　在对非法字符编码到合法URI时，规定使用percent encode编码，对非法字符的编码结果为三个字节（%+16进制字符*2）。然而如何生成percent编码，没有明确的指导规定，这也是大部分开发者拷贝旧代码却不知其所以然的原因。

　　percent encode从字面上语义明确的指出其使用%做编码标识，URL encode的实质就是正确的使用percent encode.

　　正确完成URL encode的关键问题在于：什么时候，对哪些内容，采用何种过滤原则，以及如何生成percent编码？

　　在WWW最初时，做法是将字符流转换成字节流，按照ASCII字符与字节一一对应可相互转换，使用对应ASCII字符的整型值作为%的后两个16进制字符，构成percent编码。后来出现了多种percent编码生成方法，导致了URI的难以识别。

　　现下比较通用的做法，包括iOS使用percent相关的函数时，指定或系统默认的使用UTF8转成字节流，每个字节编成一个percent编码，例如中文“网易”的URL编码为%e7%bd%91%e6%98%93，而其UTF8字节流为e7 bd 91 e6 98 93，可以看出其一一对应关系。

　　那么percent编码是在对非法字符采用某种编码（约定为UTF8）转成字节流后，逐字节加上%构成percent编码。

　　由于不同scheme或协议对URI格式有不同的要求，RFC关于对哪些内容编码，采用何种过滤原则不做硬性规定。而将决定权延后到执行时由开发者根据需要决定。通常遵循以下原则：

　　**1.不要对Unreserved Characters做percent encode编码。**

　　2.除了保留字符和非保留字符外的所有字符，必须使用percent encode进行编码。

　　3.保留字符不用于URI分隔符，而是用于其它位置，比如query部分的value时，要对这时用到的保留字符做percent encode编码。 

　　4.当两个URI的字符几乎对等，区别只在于一个对某些字符用的原有字符，另一个URI对这些字符做了percent encode时。绝大部分情况下，这两个URI应当被认为是不同的两个URI。因此，**不应当对保留字在作为保留字的使用场景时使用percent encode编码**。

![img](http://images2015.cnblogs.com/blog/586675/201605/586675-20160518011119669-1200281466.png)

图1. 保留字和非保留字

![img](http://images2015.cnblogs.com/blog/586675/201605/586675-20160517050921435-1088234871.png)

图2 不安全字符

　　**二、iOS开发中URL encode的方法编写**

　　有三个函数或系统方法用于URL encode，CFURLCreateStringByAddingPercentEscapes(9.0废弃)，stringByAddingPercentEscapesUsingencode（9.0废弃），

stringByAddingPercentencodeWithAllowedCharacters（系统推荐）。该系统推荐方法默认使用了UTF8编码然后再根据我们指定允许的字符集完成percent编码。通常我们在拼接GET请求URL时使用URL encode。

　　应用场景一般是传入URLString和一个参数NSDictionary, 这时需要**传入方保证URLString是已正确编码的**，然后遍历NSDictionary的key和value, 按需指定允许的字符集对key值和value做编码。结果输出为

　　URLString[& | ?] URLencode(key1)=URLencode(value1)&URLencode(key2)=URLencode(value2)….

 　　笔者使用的允许字符集为Unreserved Characters, 包括Reserved Characters以及中文等非法字符均会被percent编码，示例方法如下

```
+ (NSURL *) createGETURLFromString:(NSString *)urlString params:(NSDictionary *)params {
    NSURL *parsedURL = [NSURL URLWithString:urlString];
    
    NSString* queryPrefix = parsedURL.query ? @"&" : @"?";
    NSMutableArray* pairs = [NSMutableArray array];
    for (NSString* key in [params keyEnumerator]) {
        if (![[params objectForKey:key] isKindOfClass:[NSString class]]) {
            continue;
        }
        NSString *value = (NSString *)[params objectForKey:key];
        
        NSCharacterSet *allowedCharacterSet = [NSCharacterSet characterSetWithCharactersInString:@"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_.~"];
        
        NSString *urlEncodingKey = [key stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
        NSString *urlEncodingValue = [value stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
        [pairs addObject:[NSString stringWithFormat:@"%@=%@", urlEncodingKey, urlEncodingValue]];
    }
    NSString* query = [pairs componentsJoinedByString:@"&"];
    return [NSURL URLWithString:[NSString stringWithFormat:@"%@%@%@", urlString, queryPrefix, query]];
}
```

　　**三、URL encode中的 + 与 空格**

　　在使用过base64编码的童鞋多半会知道，基础base64衍生了web safe base64，更改了编码字符集，其中基本表中会出现+和/字符，这个一般会被浏览器理解成空格和路径分割符。所以为了让其工作正常，需要把索引表的最后两个字符+和/分别替换成点 **.** 和下划线 **_ **。+号经过percent编码后为%20，而20正是空格的ASCII码，这大概是浏览器的设计者将+理解成空格的原因。

　　那么在URL encode中是不是需要针对性的指定许可字符集，对+号和空格做处理防止混淆呢。实际上我们在之前使用Unreserved Characters对内容做编码时，并没有允许+和/符号出现在内容项的编码结果中。

　　因此，正确的使用percent编码，当传入的参数字典中包含有+号和/字符也是可以放心的。在传入参数是base64的结果时，并不需要特别的将base64换成web safe base64。

　　四**、不正确的URL encode可能导致的问题**

 　　有的iOS开发者拿到  GET请求URLString和参数字典后，先拼接参数，然后再对整个字符串做URL encode，造成不能区分某些字符是处于分割组件的作用，或者是作为组件的content。这是千万不可取的，可能发生如下问题：

**　　**1.如果没有正确过滤，比如http://www.baidu.com做编码后变成了http%3a%2f%2fwww.baidu.com%2findex.htm，将不能正常访问。 也就是说，为了支持对拼接后的字符串作URL encode, 必须对整个拼接后的字符串**禁止对所有URI的保留字作编码**，比如&字符，这就造成了问题2和3。

　　2. 当构建参数传入{“name” : ”namepart1&namepart2”，“id” : "kk"}。此时拼接字符串编成了http://www.baidu.com?name=namepart1&namepart2&id=kk，那么如何解析得到"name"字段“namepart1&namepart2”的实际value值？

　　3. 当构建参数传入{“name” : ”Mitty&isLogin=true”}。此时拼接字符串编成了http://www.baidu.com?name=Mitty&isLogin=true，如果isLogin真的是有意义的queryKey时，直接造成服务器接收了额外的参数。当然关于URL的攻击有很多，比如semantic attack, 这里不做讨论。

　　另外，有童鞋担心非法字符，先对中文做base64再放到get参数，从担心非法字符来讲不是必要的，经过URL编码后，与base64的结果同样是ASCII字符集，在网络上是可以正常不丢失信息的传输的。服务器接到请求时，以PHP为例，应针对每个$_GET[“key”]做URL做解码。