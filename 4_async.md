# 编写Async方法
我们现在已经知道异步代码有多棒了，那它到底有多难写呢？是时候看看C#5.0的async特性了。正如我们在第一章的Async做了什么一小节中所看到的，一个方法被标记了async，就允许包含await关键字。
```C#
private async void DumpWebPageAsync(string uri)
{
    WebClient webClient = new WebClient();
    string page = await webClient.DownloadStringTaskAsync(uri);
    Console.WriteLine(page);
}
```
这个方法执行的时候当遇到await就会停下来，等下载完成之后才会继续执行。这个转换使方法异步。这一章我们将探索如何像这样编写异步方法。

### 将示例转换为async的
还是第二章最后一小节的那个示例。我们现在将使用async修改之前的图标浏览示例。如果可以的话， 打开示例最原始的代码，也就是default分支的代码，在继续阅读更多的内容之前，尝试通过添加async和await关键字将其转换为异步方法。

最重要的方法就是下载图标的AddAFavicon方法。我们想让这个方法是异步的，以便让UI线程可以在下载的过程中继续响应用户的请求。第一步就是添加async关键字到方法上。和static关键字在方法上的标记方式一样。

然后我们需要使用await关键字去等待下载。在C#语法中，await是一个一元操作符，就和非操作符和强类型转换一样。它放在表达式的左边，意思是异步地等待表达式。

最后调用DownloadData要改成调用异步版本的DownloadDataTaskAsync。

> 一个async方法并不是自动就异步的，async方法只是让消费其他异步方法变得简单。它们在遇到一个异步方法并且要等待这个异步方法之前是同步运行的。当它们这样做时它们就变成异步的了。有些时候，一个async方法不等待任何事情，那么它就是同步运行的。

```C#
private async void AddAFavicon(string domain)
{
    WebClient webClient = new WebClient();
    byte[] bytes = await webClient.DownloadDataTaskAsync("http://" + domain + "/
        favicon.ico");
    Image imageControl = MakeImageControl(bytes);
    m_WrapPanel.Children.Add(imageControl);
}
```
比较这两个我们看到的不同版本的代码，这个看上去更像原始的同步版本的代码。没有额外的方法，只是在相同结构上做了一点小小的改动。但它和第三章中我们手动编写的异步代码表现一致。

### Task和await
让我们分解一下我们所写的await表达式。这是WebClient.DownloadStringTaskAsync方法：
```
Task<string> DownloadStringTaskAsync(string address)
```
它的返回类型是Task<string>，正如第三章《Task介绍》那一小节所提到的，Task代表一个进行中的操作，它的子类Task<T>表示操作将返回一个类型为T的结果。你可以认为Task<T>会在耗时操作完成时承诺返回一个T类型的值。

Task和Task<T>都可以代表异步操作，并且都可以在操作完成的时候进行回调。在手动编写异步代码的方式中如果要实现这种能力，需要使用ContinueWith方法传递一个委托，当耗时操作完成时继续执行下一步。await使用了相同的方式去执行你异步方法中剩余的代码。

如果你await一个Task<T>，它就变成了一个await expression，而且整个表达式返回一个T类型。正如我们在示例中所看到的，这意味着你可以将等待的结果赋值给一个变量，并且在后续的方法中使用它。然而，当你await一个Task，它就变成了一个await statement，并且不能被指定为任何类型，就好像调用了一个void方法。也就是说，作为一个Task，没有返回结果，只代表操作本身。
```
await smtpClient.SendMailAsync(mailMessage);
```
没什么可以阻止我们通过拆分await表达式去直接访问Task，或者在等待它之前去做其他事情。
```
Task<string> myTask = webClient.DownloadStringTaskAsync(uri);
// Do something here
string page = await myTask;
```
充分理解它的含义是非常重要的。DownloadStringTaskAsync方法在第一行执行。在当前线程它是同步执行，一旦开始下载，它返回一个Task<string>，依然在当前线程中。只有在我们等待那个任务的时候，编译器才会做一些特别的事情。就算await写在同一行当中也是一样的原理。（await不会另启线程的）

一旦调用DownloadStringTaskAsync耗时操作就会开始，这给了我们一个非常简单的方式可以同时执行多个异步操作。我们可以启动多个操作，保持所有的Task，然后等待它们。
```C#
Task<string> firstTask = webClient1.DownloadStringTaskAsync("http://oreilly.com");
Task<string> secondTask = webClient2.DownloadStringTaskAsync("http://simpletalk.
com");
string firstPage = await firstTask;
string secondPage = await secondTask;
```
> 如果有异常的话，这种等待多个Task的方式是危险的。如果两个操作都抛出异常，第一个await会传递它的异常，这就意味着第二个Task根本不会被等待。它的异常也不会被观察到，并且还依赖于.NET的版本和设置，也许会丢失，甚至是在一个不可预期的线程中被抛出，终止程序。我们将会在第七章看到更好的处理方式。

### Async方法的返回类型
一个被标记为async的方法可能有三种返回类型：
+ void
+ Task
+ Task<T>

没有其他返回类型，因为异步方法在返回的时候通常都没有执行结束。通常，异步方法将等待一个耗时操作，这意味着等待的那个方法会立即返回，但却在后续某个时刻实现。也就是说当方法返回的时候没有可用的结果。
> 我将展示方法返回类型的区别，例如Task<string>程序实际打算返回给调用者的类型在这个情况下就是string。在通常的非异步方法中，返回类型和结果类型总是相同的，这个不同点对于异步方法是非常重要的。

很明显，void是异步情况下一个合理的返回类型的选择。一个void类型的异步方法是一个“触发和忘记”的异步操作。调用者永远不会等到任何结果，也不会知道操作什么时候结束，有没有成功。当调用者不需要知道操作何时结束，有没有执行成功时，你应该使用void。在实际应用中，void用的非常少。大多数常见的使用void的异步方法是在异步代码和其他代码之间的边界上，例如，UI事件处理程序必须返回void。

返回Task类型的异步方法允许调用者等待操作完成，并且传递异步操作期间的发生的任何异常。当不需要返回值时，一个返回类型为Task的异步方法比一个返回类型为void的异步方法好，因为它也允许调用者用await去等待且异常变得更容易处理。

最后，返回Task<T>类型的异步方法，比如Task<string>，是当异步方法有返回类型时使用。

### 异步，方法签名和接口
async关键字出现在方法的声明处，就和public或者static关键字一样。尽管如此，async不能用于方法的签名、方法的重写、接口的声明，或者回调。

async关键字的唯一作用是对其应用的方法进行编译，这与应用于方法的其他关键字不同，这改变了它与外部世界的交互方式。也因此，围绕方法重写和接口定义的规则都忽略了async关键字。
```C#
class BaseClass
{
	public virtual async Task<int> AlexsMethod()
	{
		...
	}
}
class SubClass : BaseClass
{
	// This overrides AlexsMethod above
	public override Task<int> AlexsMethod()
	{
		...
	}
}
```
接口在声明时不能使用async，因为没有必要。如果一个接口需要一个方法返回一个Task，在实现时也许会选择使用async，但不管用不用这都是实现方法自己的选择。接口不需要指定要不要用async。

### async方法的返回语句
一个async方法的返回语句有不同的行为。记住，在一个非异步的方法中，返回语句依赖于方法的返回类型：
+ 返回类型为void的方法
　　返回语句必须是return;或者不写。
+ 返回类型为T的方法
　　返回语句必须是一个类型为T的表达式，比如return 5+X;而且必须写在方法的结尾处。

在标记为async的方法中，不同的情况也有不同的规则：
+ 返回类型为void的方法和返回类型为Task的方法
　　返回语句必须是return;或者不写。
+ 返回类型为Task<T>的方法
　　返回语句必须是一个类型为T的表达式，而且必须写在方法的结尾处。

在async方法中，方法的返回类型和返回语句中表达式的类型不同。在返回给调用者之前，编译器被认为可以将结果包裹起来并转换为Task<T>类型。当然，事实上Task<T>是被立即创建的，一旦耗时操作结束，就会被你的结果所填充。

### 异步方法的传递性
正如我们所看到的，消耗一个异步API返回的Task的最好方式就是在一个async方法里await它。当你这样做时，你的方法也会返回Task。为了享受异步风格的优势，调用你方法的代码必须不能是阻塞的等待你的Task完成，而且你的调用者很可能也在await你。

这有一个我写的帮助方法示例，用来获取Web页面的字符数并异步的返回。
```C#
private async Task<int> GetPageSizeAsync(string url)
{
    WebClient webClient = new WebClient();
    string page = await webClient.DownloadStringTaskAsync(url);
    return page.Length;
}   
```
为了使用它，我需要写另一个异步方法来异步地返回结果：
```C#
private async Task<string> FindLargestWebPage(string[] urls)
{
    string largest = null;
    int largestSize = 0;
    foreach (string url in urls)
    {
        int size = await GetPageSizeAsync(url);
        if (size > largestSize)
        {
            size = largestSize;
            largest = url;
        }
    }
    return largest;
}
```
通过这种方式，我们最终编写了异步方法的链，每一个都在等待下一个。异步是一种具有传递性的编程模型，它可以很容易地渗透到整个代码库中。但我想这事因为async方法非常容易编写，而这并不是什么问题。

### 异步匿名委托和Lambda
普通的方法可以是异步，有两种表达方式的匿名方法也可以是异步的。语法和一般的方法很像。如何编写一个异步的匿名委托：
```C#
Func<Task<int>> getNumberAsync = async delegate { return 3; };
```
异步的Lambda：
```
Func<Task<string>> getWordAsync = async () => "hello";
```
所有的规则都适用，就和普通的异步方法一样。你可以使用它们来保持代码简洁，捕捉闭合，就和非异步代码中一样。
