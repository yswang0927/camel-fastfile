我在现场遇到的最常见的用例之一是摄取基于文件的数据（批处理或其他方式），然后对其进行验证，转换，处理，存储……。幸运的是，Apache Camel的 文件和FTP组件使此操作非常容易。如此之多，以至于它几乎不需要任何思考就可以启动并运行。而且，如果您使用少量的较大批处理文件，则默认值可能足够好，并且不需要在其中添加太多内容。但是，如果您正在消耗大量较小的文件，并且想要获得最高的性能，则可能需要考虑一些配置。

编写文件轮询器时，最常被忽略的要求之一是您需要某种机制来确定编写器何时完成编写。如果您在文件完成之前抓取文件，则最终将其截断并得到垃圾数据。不幸的是，您如何在一个文件系统上进行确定并不一定适用于所有文件系统。此外，不同的策略将具有不同的性能特征，并且通常最终会成为您最大的瓶颈（实际数据处理之外）。幸运的是，骆驼为您提供了多种现成的策略，即使它们都不满足您的需求，您甚至可以创建自己的策略。那么如何选择……

首先，让我们介绍绝对最快，最通用的解决方案。如果您控制写入文件的过程以及读取文件的过程，则可以使用“临时文件”策略。也就是说，我可以让我的编写者编写一个具有某种临时名称的文件（即，附加一个“ .inprogress”扩展名），然后在写入完成时将其原子地移动到其最终名称（即，删除“。”）。进行中”扩展）。然后，我可以轻松地让使用者将临时文件过滤掉，而只使用完整的文件。骆驼可以为您完成所有这些工作。因此，无需担心编写大量额外的代码。只需在生产者（即tempFileName）和消费者（即exclude或antExclude）上设置适当的选项，然后将其命名为一天。:)

另一个类似的解决方案是使用“完成文件”策略。在这种策略下，您将写入文件，完成后，您将写出另一个具有相同名称和某些“标记”扩展名（即“ .done”）的文件。然后，您将指示使用者仅在找到其对应的“完成”文件后才选择文件。同样，骆驼使这成为简单的配置问题。只需doneFileName在生产者和doneFileName消费者上设置选项。对我来说，这似乎比以前的解决方案笨拙，但最终结果是相同的。

先前的两种策略都非常快，几乎可以在任何文件系统上运行。但是，如前所述，它们要求您控制生产者和消费者方面。那么，如果您仅控制消费者呢？好吧……您可以使用其中一个readLock选项。不幸的是，大多数可用readLock选项都比确保编写者完成写入更关心确保其他使用者没有拾取相同的文件。并且由于我们还有其他方法来确保其他消费者不会踩我们的脚趾，因此我们将专注于可以帮助我们解决后一个问题的选项。

现成可用的最强大的选项是“已更改”策略。基本上，它的工作方式是使用者将获取文件并检查其最后修改的时间戳。然后它将继续检查上次修改的时间戳记（在配置的readLockCheckInterval），直到确定文件已停止更改（即，上次轮询的上次修改的时间戳与当前文件匹配）。一旦确定文件不再更改，它将使用该文件并将其传递到路由的其余部分进行处理。此策略是一个很好的选择，因为它几乎可以在任何地方工作（例如，本地文件系统，FTP，SFTP等），并且可以进行配置，足以处理“慢速写程序”（通过调整/增加readLockCheckInterval选项）。而且，如果您要获取少量较大文件，则可能足够快。但是，如果您尝试使用大量较小的文件，则会很快看到瓶颈……当前的实现将遍历每个文件，并（针对每个文件）检查时间戳。它将继续循环并检查该文件上的时间戳，直到检测到它已停止更改或达到超时（通过readLockTimeout选项配置）为止。在满足这些条件之一之前，它不会继续前进到下一个文件。这意味着，如果我有很多生产者在编写文件，则由于单个生产者速度较慢，这些文件可能全部卡住，等待被消耗。在实践中，我实际上已经看到这种情况的发生，这会导致非常糟糕的情况，即轮询本身开始花费的时间太长（在文件系统级别，并且在Camel的控制范围之外），因为目录中的文件数量很多。一旦开始发生，它实际上只会产生滚雪球效应。因此很难从中恢复，通常需要手动干预。那么我们该怎么办？

好吧……幸运的是，Camel非常出色，它允许我们在即装即用的选项无法满足我们的需求时对其进行扩展。参加那场比赛！:)具体来说，在前面的场景中，我们实际上通过创建我们自己的“已更改”策略的自定义版本来解决了该问题。只是，在我们的版本中，我们没有暂停并重复检查单个文件。取而代之的是，我们遍历文件并（针对每个文件）检查其统计信息。然后，我们将这些统计信息添加到缓存中并移至下一个文件。在随后的每次轮询中，我们将对照缓存的文件检查其统计信息，以确定其是否已停止更改（至少对于readLockCheckInterval多少时间）。这使我们能够继续处理所有准备就绪的文件，而不必等待一个尚未完成的文件。实际上，我们仅使用一台服务器就可以使用这种策略来消耗大量文件。如果您想尝试一下，请看一下示例源代码：https : //github.com/joshdreagan/camel-fastfile。

#Camel Fast File Consumer

This project contains example `org.apache.camel.component.file.GenericFileExclusiveReadLockStrategy` implementations that speed up file consumption when the only options for determining a read lock are the 'changed' strategy.

##Requirements

- [Apache Maven 3.x](http://maven.apache.org)

##Building Example

Run the Maven build

```
~$ cd $PROJECT_ROOT
~$ mvn clean install
```

##Running Camel

Run the Maven Camel Plugin

```
~$ cd $PROJECT_ROOT
~$ mvn camel:run
```

You can also pass in extra JVM args to bump logging and point to an external config file

```
~$ cd $PROJECT_ROOT
~$ mvn camel:run -Dorg.slf4j.simpleLogger.log.org.apache.camel.component.file=debug -DconfigFile=$HOME/myConfig.properties
```


##Testing

Simply write a file into the configured "input" directory. To really see things work, you'll want to simulate a slow writer. To do so, you can use the following bash command in a separate terminal window:

```
~$ cd $HOME/input #Replace this with your configured "input" directory.
~$ while [ 0 -eq 0 ]; do echo 'foo' >> file01.txt; sleep 0.5; done;
```

Create as many of these "slow writers" as you'd like (using separate terminal windows and unique file names for each) to simulate multiple files being written simultaneously. The files should not be picked up by the Camel routes until you kill the writer script. Once you do, the Camel route will detect that the file has stopped changing and will pick it up.
