# 手动编写异步代码
这一章，我们将讨论如何在没有C#5.0以及async的帮助下异步代码是怎么写的。在某种程度上，这是一种你永远都不会用到的技术，但它的重要性在于可以帮助你理解幕后的到底发生了什么。也因此，我将快速地回复这些例子，只指出那些有助于理解的要点。

### .Net中使用的一些异步模式
正如我前面提到的，Silverlight只提供类似web请求的异步API。这有一个展示了如何下载网页并呈现的例子：
```C#
private void DumpWebPage(Uri uri)
{
    WebClient webClient = new WebClient();
    webClient.DownloadStringCompleted += OnDownloadStringCompleted;
    webClient.DownloadStringAsync(uri);
}
private void OnDownloadStringCompleted(object sender,
    DownloadStringCompletedEventArgs eventArgs)
{
    m_TextBlock.Text = eventArgs.Result;
}
```
这种API是基于事件的异步模式（EAP）。它的想法不是去使用一个同步方法下载页面，而是用一个方法和一个事件，在页面下载完成之前这个方法都会出于阻塞状态。这个方法看起来就像同步的，除了它的返回类型是void的。事件有一个特别的EventArgs类型，包含检索值。

在调用这个方法之前我们会立即去注册这个事件。这个方法会理解返回，当然，因为它是异步代码。然后在将来的某个时刻，我们可以处理将触发的事件。

这个模式显然是很混乱的，尤其是你必须要将一个简单的指令序列分成两个方法。重要的是，你已经注册了一个事件，这增加了问题的复杂度。如果另一个请求你继续使用这个WebClient实例，你也许并不希望原来的事件仍然附加在后面并且处理程序也任然可以运行。

另一种.NET中的异步模式涉及到IAsyncResult接口。一个例子是DNS查找主机IP地址的方法BeginGetHostAddresses。这个设计需要两个方法，一个叫BeginMethodName用来启动操作，另一个叫EndMethodName在回调方法中使用。
```C#
private void LookupHostName()
{
    object unrelatedObject = "hello";
    Dns.BeginGetHostAddresses("oreilly.com", OnHostNameResolved, unrelatedObject);
}
private void OnHostNameResolved(IAsyncResult ar)
{
    object unrelatedObject = ar.AsyncState;
    IPAddress[] addresses = Dns.EndGetHostAddresses(ar);
    // Do something with addresses
    ...
}  
```

至少这个设计不会受到遗留事件处理程序的问题的影响。然而，它仍然给API增加了额外的复杂度，有两种方法而不是一种，我觉得它不自然。

这两种模式都需要你将你的过程分成两个方法。IAsyncResult模式要求你将第一个方法的结果传递到第二个方法中，就好像我传递了一个string类型的hello参数一样。但这种方式比较复杂，即使你不需要传参，它也要求你必须传参，而且强制你给出一个Object对象的返回值。基于事件的异步模式也支持这种复杂的传参方式。

在方法之间传递上下文是异步模式的一个普遍问题。在下一节中，我们将会发现Lambda是一个可以在任何情况下使用的解决方案。

### 最简单的异步模式
具有异步行为的最简单代码，如果没有使用async，就需要将一个回调作为参数传递给该方法：
```C#
void GetHostAddress(string hostName, Action<IPAddress> callback)
```
我发现这比其他模式要好用一些。
```C#
private void LookupHostName()
{
    GetHostAddress("oreilly.com", OnHostNameResolved);
}
private void OnHostNameResolved(IPAddress address)
{
    // Do something with address
    ...
}
```
正如我前面提到的，你可以使用一个匿名方法或者Lambda表达式进行回调，而不是使用两个方法。这样做有一个重要的好处就是它允许你访问方法第一部分的变量。
```C#
private void LookupHostName()
{
    int aUsefulVariable = 3;
    GetHostAddress("oreilly.com", address =>
    {
        // Do something with address and aUsefulVariable
        ...
    });
}   
```
尽管Lambda表达式有点儿难以阅读，但如果你使用多个异步API的话，你将要使用多个彼此嵌套的Lambda表达式。你的代码将会快速缩小且难以处理。

这种简单方法的缺点是任何异常都不会抛出给调用者。之前.NET中使用的模式，无论是调用EndMethodName方法，还是获取Result属性都会重新抛出异常，所以我们可以处理异常。相反，这种简单方法可能在某个不恰当的地方结束，且没有做任何异常处理。

### Task介绍
任务并行是在.Net framework4.0版本引入的。其最重要的地方就是Task类，代表了一个正在执行的操作。当操作完成时返回一个```Task<T>```类型的值。

C#5.0的async特性广泛地使用了Task，我们将在后面继续讨论。然而，即使没有async，你也可以使用Task，尤其是```Task<T>```来编写异步程序。你需要启动一个操作，这个操作将返回一个```Task<T>```对象，然后使用ContinueWith方法去注册你的回调。
```C#
private void LookupHostName()
{
    Task<IPAddress[]> ipAddressesPromise = Dns.GetHostAddressesAsync("oreilly.com");
    ipAddressesPromise.ContinueWith(_ =>
    {
        IPAddress[] ipAddresses = ipAddressesPromise.Result;
        // Do something with address
        ...
    });
}
```
Task的优势在于，DNS里只需要一个方法，使API更简洁了。与调用异步行为相关的所有逻辑都可以在Task类中进行，而无需在每个异步方法中进行复制。这个逻辑可以做一些很重要的事情，比如异常处理和上下文同步，这些对于在一个特殊的线程上（例如UI线程）运行回调函数是非常有用的，我们将在第八章详细讨论。

最重要的是，Task让我们能够以一种抽象的方式处理异步操作。我们可以利用这种可组合性来编写一些实用工具，可以和Task配合提供一些在很多情况下都非常有用的功能。我们将在第七章看到更多这样的实用工具。

### 手动编写异步代码的问题
正如我们所看到的，有许多方式可以实现异步编程。有些方式比较简洁。但希望你已经发现，他们有一个缺点。实现的过程必须被分成两部分：方法和回调。使用一个匿名方法或者Lambda作为回调在某种程度上可以缓解这个问题，但你的代码会被过度简化，难以阅读和维护。

还有一个问题，我们已经讨论了实现一个异步回调的方法，但如果需要多个回调会发生什么呢？更糟的是，如果你要在一个循环中调用异步方法又会发生什么呢？你唯一的选择是比普通循环更加难以阅读的递归方法。
```C#
private void LookupHostNames(string[] hostNames)
{
    LookUpHostNamesHelper(hostNames, 0);
}
private static void LookUpHostNamesHelper(string[] hostNames, int i)
{
    Task<IPAddress[]> ipAddressesPromise = Dns.GetHostAddressesAsync(hostNames[i]);
    ipAddressesPromise.ContinueWith(_ =>
    {
        IPAddress[] ipAddresses = ipAddressesPromise.Result;   
        // Do something with address
        ...
        if (i + 1 < hostNames.Length)
        {
            LookUpHostNamesHelper(hostNames, i + 1);
        }
    });
}
```
天哪！
采用这些方式手动编写异步代码还会引发另一个问题，就是消耗你所编写的异步代码。如果你写了一些异步代码，想要在你程序的其他地方使用，你必须提供一个异步API的使用说明。使用一个异步API看上去已经很困难很复杂了，再提供一个异步API的使用说明会双倍的困难和复杂。异步代码的复杂度是会蔓延的，所以您不仅要处理异步api，而且您的调用者和他们的调用者也要处理，直到整个程序一团糟。
### 使用手动编写的异步代码
再看第二章的最后一个示例，我们讨论了一个WPF UI的应用程序，在下载网站的图标时阻塞了UI线程导致程序无响应。现在我们看看如何使用本章手动的技术来让它变成异步的方式。

首先就是要找到一个异步API，我使用的是WebClient.DownloadData。正如我们已经看到的，WebClient使用的是基于事件的异步模式（EAP），所以，我们可以注册一个事件句柄作为回调，然后开始下载。
```C#
private void AddAFavicon(string domain)
{
    WebClient webClient = new WebClient();
    webClient.DownloadDataCompleted += OnWebClientOnDownloadDataCompleted;
    webClient.DownloadDataAsync(new Uri("http://" + domain + "/favicon.ico"));
}
private void OnWebClientOnDownloadDataCompleted(object sender,
    DownloadDataCompletedEventArgs args)
{
    Image imageControl = MakeImageControl(args.Result);
    m_WrapPanel.Children.Add(imageControl);
}
```

当然，我们的整体逻辑需要被分成两种方法。我不喜欢用Lambda代替EAP，因为Lambda会出现在调用下载之前，这是我觉得难以理解的地方。

这个例子的[源码](https://bitbucket.org/alexdavies74/faviconbrowser)，在manual分支。如果你运行它会发现，UI可响应了，图标也是一个一个出现的。也因此，我们引入了一个Bug。现在，所有的下载操作都是同时开始的，图标的排列顺序是根据下载完成的先后顺序去显示的，而不是我们的请求顺序去显示的。如果你想检验一下自己是否真的理解了手动编写异步代码是如何工作的，我建议你修复这个bug。orderedManual分支下有一个可用的解决方案，包括把循环转换为递归的方法。可能也有其他的解决方案。