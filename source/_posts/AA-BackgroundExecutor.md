title:	Somethings about AA's BackgroundExecutor
date:	2015-05-07 22:51:51
categories: 程序媛
tags:
	- Android
	- AndroidAnnotations
---

记录一下公司工作里使用AndroidAnnotations出现的一个问题的解决过程。
<!--more-->
由AA的实现机制可知，一个被标记为`@Background`的方法，例如：
```java
	@EActivity(R.layout.activity_main)
	public class SampleActivity extends Activity {
		...		
		@Background
		protected void method() {
			...// do anything you like
		}
		...
	}
```

AA实际生成的代码是：
```java
	public class SampleActivity_ extends SampleActivity implements HasViews, OnViewChangedListener {
		...
		@Override
		public void method() {
			BackgroundExecutor.execute(new BackgroundExecutor.Task("", 0, "") {
				@Override
				public void execute() {
					try {
						SampleActivity_.super.method();
					} catch (Throwable e) {
						Thread.getDefaultUncaughtExceptionHandler().uncaughtException(Thread.currentThread(), e);
					}
				}
			});
		}
		...
	}
```

因此，我们在方法`method()`中写的代码，即使是进了方法后的第一行代码，事实上也必须在`BackgroundExecutor`从线程池中获取到可用的后台线程后，才会被执行，如果线程池当前已满，就必须等到现有的线程有彻底执行完毕的，才会将执行`method()`中的代码排上日程，于是在低配手机上，频繁测试后很容易出现按钮不响应的现象——线程池满啦！
这显然是我们不愿意见到的。
于是我查找了项目中使用`@Background`的所有地方，将进入`@Background`的方法后才做的出现loading框和禁用相关联控件的代码都提出到调用这个方法的语句之前。
之前之所以放那里是因为我在库项目里封装了`showProgress()`系列方法，能够方便地用一行代码实现将出现loading框和禁用相关联控件的操作post回主线程执行，放在此处可以避免多次调用`@Background`方法漏掉这个语句，但现在看来这个小心思显然有点儿蠢。
顺便这时候我还去更新了库项目，对`showProgress()`系列方法增加了判断当前线程是不是UI线程的代码，以期达到更好的运行效率，在已经是身处UI线程的情况下，省掉了new一个`Runnable`对象所带来的资源分配开销。

顺便也是基于同样的理由，建议后续使用AA的项目中，在用到`@UiThread`时，如果此方法既可能被Background线程调用又可能被UI线程调用，建议使用`(propagation = Propagation.REUSE)`标示来节省资源。但若你很清楚此方法仅仅会在Background线程中调用的话，建议还是不要加标示了，毕竟判断当前线程是否是UI线程的代码执行时候也是要消耗时间的。
并且，如果一个方法仅仅会在UI线程中被执行，调用它的方法也都很明确是运行在UI线程中的方法（包括已经被标示为`@UiThread`的方法和各种控件的点击事件、触摸事件等），则不建议将这个方法再标示为`@UiThread`的方法，即，不要滥用AA标记！

但是事情到这里还没有结束，虽然我很想到此为止了，但是这样实际上是治标不治本的，只是防止了多次点击，并没有解决操作缓慢的问题。
于是再次求助谷歌——“万能的谷哥哟，为什么BackgroundExecutor会执行缓慢呢？”
在数不清的搜索结果中，这样一个条目引起了我的注意：[New BackgroundExecutor is failing in my app](https://github.com/excilys/androidannotations/issues/625)。这是一位名为RomainPiel的朋友在AA的开源项目中提交的一条issue，点进去详细看一下，几乎与我们碰到的现象一模一样！
“别高兴的太早，继续看下去，问题解决了吗？”带着疑问滚动着页面，终于发现了这么一条回复：

> rom1v commented on 11 Jun 2013
>
> @RomainPiel The executor has a fixed number of threads.
> If you submit more tasks than this threshold, then your tasks will be blocked until previous ones have completed execution.
>
> Before pull request #569, Executor implementation was Executors.newCachedThreadPool(), so your threads were not blocked above a threshold.
>
> But you should not rely on this unspecified behaviour: if you need to start a lot of long-running threads, then you must set your own Executor to ensure the behaviour:
>
> `BackgroundExecutor.setExecutor(Executors.newCachedThreadPool());`

顺着这条回复提供的线索，我找到pull request #569当时对`BackgroundExecutor`所做的更改，在这次提交中，`BackgroundExecutor`的`DEFAULT_EXECUTOR`被改成了
`Executors.newScheduledThreadPool(2 * Runtime.getRuntime().availableProcessors());`。
个人揣测这么做的原因是想要节约系统资源，避免滥用多线程拖慢系统的情况发生。
在当前四核八核机当道的Android机市场上，这样的代码也许问题并不大，线程池中允许同时运行十几个线程对大多数的应用来收都足够了，在我们的应用里，每个界面中的`@Background`其实鲜有并行执行的。但是在早期的单核Android手机上，这样的线程池规模显然就捉襟见肘了。
找到了问题所在，下一步当然就是fix it了！
但是我必须坦承，现在的解决方案并不算好，甚至可以说是很简陋，我所做的事情就是在应用的`MainApplication`的`onCreate()`方法中增加了这么一句：
`BackgroundExecutor.setExecutor(Executors.newCachedThreadPool());`
相当简陋，用回`Executors.newCachedThreadPool()`也就意味着我们放弃了AA节约系统资源的这次努力，又开始铺张浪费了XD，不是好习惯XD。
所以，后续在这方面可能还需要再进行一些研究，现在考虑可能有这么几个方向：

* 用回`Executors.newScheduledThreadPool(n)`，但是要如何确定一个合适的`n`值？
* 是否还有其他`Executor`能够实现线程池的动态扩展，按需扩展？
* ……
* AA现在提供了为`@Background`方法指定ID来实现cancel，在何时的时候取消后台线程的执行，从代码编写上更严密地避免无用后台线程占用资源。
* 是否自己实现一个新的`Executor`来满足以上所有需求？

以上，就是这两天解决问题的所思所想。对公司其他使用了AA的项目，建议了先采用当前的解决方案，在应用的`MainApplication`的`onCreate()`方法中增加`BackgroundExecutor.setExecutor(Executors.newCachedThreadPool());`来应对低配手机上的不响应问题。
求大神们提出更好的解决资源浪费问题的解决方案～～

#### END
