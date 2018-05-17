> 原文地址：[Angular’s $digest is reborn in the newer version of Angular][1]

我已经从事Angular.js相关方面的工作好几年了，尽管这是一个饱受批评的框架，但我依旧认为它是极其出色的。一开始从[《构建你自己的Angular.js》][2]这本书入门，几年间我也阅读过框架的大部分源码，因此我坚信对于Angular.js内部的工作原理我有扎实的基础，也很好的领会了框架的思想。现在，对于新版Angular我试图达到与Angular.js相同的理解水平并对比版本间实现思想的差异。我发现，与网上声称的相反，Angular从前任借鉴的很多思想。

一个臭名昭著的思想是`digest`循环：

> 这会造成可怕的性能开销。应用中的任何改变都会导致成百上千的函数寻求变化。这是Angular（Angular.js）的基础组成部分，为了保持高性能，这将限制住构建应用时的UI数量。

尽管对angular（Angular.js）中的`digest`的实现有比较好的理解，但设计出高性能的应用依旧存在可能。比如说，选择性的使用`$scope.$digest`取代`$scope.$apply`以及拥抱不可变对象（译者注1）。事实上，理解框架内部实现才是构建高性能应用的必经之路，但这确实是大部分人的拦路虎。

这也难怪大部分教程都声称Angular摒弃了`$digest`循环。这个观点很大程度上取决于你对`digest`的定义，但我认为，鉴于其目的，这是一个误导性的说法。`digest`依旧存在，是的，我们不再明确的使用scopes和watchers，也不再调用`$scope.$digest`，但是遍历组件树、隐式调用watcher以及对DOM的更新，这些变化检测的机制给了Angular第二次生命。完全的重写，更强大的性能。

本文探讨了`digest`在Angular.js和Angular中实现的不同，无论对Angular的开发者还是那些想把项目从Anguar.js迁移到Angular上的人来说都是很有帮助的。

digest的必要性
----------

在开始之前，让我们先回忆一下为什么digest会首先出现在angular.js中，很多框架都解决了数据模型（JavaScript对象）和UI（浏览器DOM）之间的同步问题，其中最大的挑战是对数据变化知悉的实现。我们把验证变化的过程叫做变化检测，它的实现是现今主流框架的最大区别，我计划写一篇关于变化检测在各个已存在框架中实现的对比，如果你对此感兴趣，请关注我。

有两种变化检测的方法 — 用户通知框架或者通过比较自动检测变化。假设我们有如下对象：

    let person = {name: 'Angular'};

并且已经更新了name属性，那么框架是如何知道它已经被更新了呢？一种方法是要求用户去通知框架：

    constructor() {
        let person = {name: 'Angular'};
        this.state = person;
    }
    ...
    //明确的对变化做出通知
    this.setState({name: 'Changed'});

或者强迫对属性使用包装器，这样框架就可以对其添加setter：

    let app = new Vue({
        data: {
            name: 'Hello Vue!'
        }
    });
    // 这个setter被触发，这样Vue就知道什么改变了
    app.name = 'Changed';

另一种方法是将当前值与前值对比：

    if (previousValue !== person.name) // 变化检测，更新DOM

每次运行代码都会伴随着验证的执行，那么什么时候应该完成比较呢？我们知道异步事件触发了上述代码的运行—称之为虚拟机轮询（VM turn/tick），我们可以在每次循环结束后开始校验，Angular.js中的digest也是这么做的，说到这里，我们可以给digest下个定义：

> 是一种变化检测的机制，通过遍历组件树，校验每个组件的的变化，并且在组件属性变化时更新DOM。

如果我们对digest如此定义，我敢断定在新版的Angular中这种主要机制没有改变，改变的仅仅是`digest`的具体实现。

Angular.js
----------

Angular.js中使用了观察者（`watcher`）和监听器（`listener`）的概念。观察者函数会返回一个被观察的值，通常情况下这会是数据模型的属性，但这是不一定的-我们可以跟踪作用域上组件的状态、计算值、第三方组件等等。如果（watcher）返回的值与前值不同，那么angular就会调用监听器，监听器通常用于更新UI。

这些都反应在`$watch`这个函数参数中：

    $watch(watcher, listener);

因此，如果我们在html（如：`<span>{{name}}</span>`）中使用了person对象的name属性，那么我们可以按照如下来代码追踪属性、更新DOM:

    $watch(() => {
        return person.name
    }, (value) => {
        span.textContent = value
    });

这本质上就是angular.js中的插值表达式和指令（如：`ng-bind`）的实现。angular.js利用指令把数据映射到DOM中。新版Angular已经不这么做了， 它使用属性映射来连接数据模型和DOM。前面的例子现在是这样实现的：

    <span [textContent]="person.name"></span>

因为我们有很多组件，每个组件拥有不一样的数据模型，因此我们拥有一个与组件树非常类似的watcher的层级结构。顺便说一下，watcher是使用$scope分组访问的。

Angular.js中digest 遍历watcher树并更新DOM，通常情况下，如果你使用现有的机制如`$timeout`,`$http,$scope.$apply`,`$scope.$digest`，那么每一次的异步事件会都会触发`digest` 循环。

观察者（`watchers`）按照严格的顺序触发—先是父级组件随后才是子组件。这有一定的道理，但某些情况下也会造成不好的影响。一个watcher的监听器（`listener`）存在各种各样的副作用，其中就包括更新父级组件的属性。如果父组件的监听器已经执行，但是子组件又更新了它的属性，那么这个变化将不被检测到。这就是为什么，`digest` 循环不得不运行多次才能稳定（确保没有更多变化）。循环次数被限制在10次。这个设计是有缺陷的，Angular已不再采用。

Angular
-------

Angular 没有类似于Angular.js中的`watcher`的概念，但是模型属性的追踪还是存在的。这些更新的方法在框架编译时产生并且无法访问。它们也与底层的DOM有着强连接。这些方法被存在View的一个属性名叫`updateRender`的方法中。

这些方法是非常明确的，它们只追踪模型的变化而不是像Angular.js那样追踪所有。每一个组件有且仅有一个观察者（watcher），用于跟踪在模板中使用的所有组件属性。Angualr使用`checkAndUpdateTextInline`这个方法来追踪属性而不是返回一个值。这个方法对比当前值和前值而后更新DOM。

举个例子，AppComponent中存在如下模板：

    <h1>Hello {{model.name}}</h1>

这将被编译成如下代码：

    function View_AppComponent_0(l) {
        // jit_viewDef2 is `viewDef` constructor
        return jit_viewDef2(0,
    
    
            // array of nodes generated from the template
            // first node for `h1` element
            // second node is textNode for `Hello {{model.name}}`
            [
                jit_elementDef3(...),
                jit_textDef4(...)
            ],
            ...
    
    
            // updateRenderer function similar to a watcher
            function (ck, v) {
                var co = v.component;
    
    
                // gets current value for the component `name` property
                var currVal_0 = co.model.name;
    
    
                // calls CheckAndUpdateNode function passing
                // currentView and node index (1) which uses
                // interpolated `currVal_0` value
                ck(v, 1, 0, currVal_0);
            });
    }

因此，即使`watcher`的实现方式不同，但是`digest` 循环依旧存在。只是换了个名称而已。

在开发者模式中，`tick()`也会执行第二次以确保没有检测到其他改变。

我前面提到在angular.js中，`digest`是通过遍历`watcher`树并更新DOM的。在Anuglar中同样的事情也在发生。Angular通过遍历组件树并调用渲染更新函数来实现变化检测。这作为检测和更新视图过程的一部分，我已经在[“你所要知道的所有关于Angular变化检测”][3]中说的很详细。

正如Angular.js，在新版Angular中变化检测也是由异步事件触发。不同的是Angular使用`zone`接管了几乎所有异步事件，对于大部分异步事件而言无需手动触发变化检测。zone订阅了`onMicrotaskEmpty`事件，在每个异步事件完成后将获得通知。如果在当前VM轮询中没有`microtasks`需要执行（译者注2），那么这个事件就会被触发。当然，变化检测也可以通过`view.detectChanges`或者`ApplicationRef.tick`手动被执行。

Angular强迫使用自上而下的单项数据流（译者注3）。如果父组件的更改处理完毕了，那么子组件更新父组件的属性是不被允许的。 如果你在组件的`DoCheck`钩子中执行父组件属性的更新，这是可以的，因为这个生命周期的钩子在属性变化检测之前调用。但是这个操作在其他步骤执行，比如说，在`AfterViewChecked`钩子中，在开发者模式下就会有如下错误：`Expression has changed after it was checked。`想了解关于此类错误的更多信息，你可以阅读：[你所需要知道的 ExpressionChangedAfterItHasBeenCheckedError错误][4]

在生产环境中这不会报错，但Anuglar不会检测这些变化直到下一轮脏值检测。

使用生命周期的钩子来追踪变化
--------------

在angular.js中每个组件都定义了一系列的`watchers`来追踪以下内容：

	* 父级组件的绑定

	* 自身组件属性

	* 计算值

	* 第三方插件


以下是这些功能在Angular中实现。为了跟踪父组件属性，我们现在可以使用`OnChanges`。

我们可以使用`DoCheck`钩子来跟踪组件本身属性以及计算值属性。由于此钩子在当前组件上的Angular进程属性发生更改之前触发，因此我们可以根据需要执行任何操作，以便在UI中正确反映更改。

我们可以使用`OnInit`钩子来监听Angular生态系统之外的第三方插件，并手动运行变更检测。

例如，我们有一个显示当前时间的组件。时间由Time服务提供。下面是它将如何在Angular.js中实现的：

    function link(scope, element) {
        scope.$watch(() => {
            return Time.getCurrentTime();
        }, (value) => {
            $scope.time = value;
        })
    }

以下是在Angular中的实现：

    class TimeComponent {
        ngDoCheck()
        {
            this.time = Time.getCurrentTime();
        }
    }


另一个例子是，如果我们有第三方`slider`组件未集成到`Angular`生态系统中，但我们需要显示当前幻灯片，我们只需将此组件包装到角度组件中，changed 手动跟踪滑块的事件并手动触发摘要以反映UI中的更改：

    function link(scope, element) {
        slider.on('changed', (slide) => {
            scope.slide = slide;
    
            // detect changes on the current component
            $scope.$digest();
    
            // or run change detection for the all app
            $rootScope.$digest();
        })
    }

同样的思路也适用于Angular:

    class SliderComponent {
        ngOnInit() {
            slider.on('changed', (slide) => {
                this.slide = slide
    
                // detect changes on the current component
                // this.cd is an injected ChangeDetector instance
                this.cd.detectChanges();
    
                // or run change detection for the all app
                // this.appRef is an ApplicationRef instance
                this.appRef.tick();
            })
        }
    }

译者总结：
----
本文作者主要阐述了一个事实：digest依旧存在于Angular中，只是内部实现的方式有所不同。区别可以概括为一下几点：
* 由于采用单项数据流，使得Angular可以自上而下的树型检测而不是像Angular.js那样需要循环检测（如下图所示），提升了执行效率（Angular.js需要循环检测10次，Angular只要一次），更多内容可以参考[Victor Savkin在ng-conf的演讲][7]，需翻墙
![image](https://segmentfault.com/img/bV4O60)

* Angular使用zone.js接管了所有异步事件，使得脏值检测在异步事件中也无需手动触发。


译者注：
----

1. [不可变对象][5]
2. 关于microtask，可以点击[event loop][6]查看，在Angular源码中，触发变化检测的基本流程是这样的：
   `ngZone`监听`onMicrotaskEmpty`事件，如果事件触发则执行`tick()`，摘录源码如下：
    
    ```
    this._zone.onMicrotaskEmpty.subscribe(
        {next: () => { this._zone.run(() => { this.tick(); }); }});
    ```
    之后`tick()`在循环所有视图，并依此调用`detectChanges`
    
    ```
    tick(): void {

        if (this._runningTick) {
          throw new Error('ApplicationRef.tick is called recursively');
        }
    
        const scope = ApplicationRef._tickScope();
        try {
          this._runningTick = true;
          this._views.forEach((view) => view.detectChanges());
          if (this._enforceNoNewChanges) {
            this._views.forEach((view) => view.checkNoChanges());
          }
        } catch (e) {
          // Attention: Don't rethrow as it could cancel subscriptions to Observables!
          this._zone.runOutsideAngular(() => this._exceptionHandler.handleError(e));
        } finally {
          this._runningTick = false;
          wtfLeave(scope);
        }
      }
    ```


3. 单项数据流。

   什么是单项数据流？顾名思义数据的流向是单一方向的，即数据是按照`Model->Component->View`的顺序流动的，这么做能够有效的保证数据的统一，Angular正是采用了这个原则，才能减少变化检测的次数。
   
   那么是不是意味着在Angular中就无法通过组件改变数据结构呢？显然不是这样的，只要在Angular开始渲染视图之前，改变都是允许的。
   
   举个例子：
   
   假设我们有一个组件cd，定义如下：
   
    ```
    import { 
        Component,
        OnChanges,
        OnInit,
        DoCheck,
        AfterContentInit,
        AfterContentChecked,
        AfterViewInit,
        AfterViewChecked
    } from '@angular/core';
    
    
    @Component({
        selector: 'cd',
        template: `
            <span>计数器：{{count}}</span>
            <ng-content></ng-content>
        `
    })
    export class CdComponent implements OnChanges,OnInit,DoCheck,AfterContentInit,AfterContentChecked,AfterViewInit,AfterViewChecked{
    
        //计数器
        count: number = 0;
    
        constructor(){}
    
        ngOnChanges(){ }
        ngOnInit(){ }
        ngDoCheck(){ }
        ngAfterContentInit(){ }
        ngAfterContentChecked(){ }
        ngAfterViewInit(){ }
        ngAfterViewChecked(){ }
    }
    ```
    假设现在要改变计数器的数值，实现的方式有很多种，在这里为了得到想要的结果，对比以下两种方式：
    
    在生命周期DoCheck这个钩子中实现：
    
    ```
    ngDoCheck(){ ++this.count; }
    ```
    
    在生命周期AfterViewInit这个钩子中实现：
    
    ```
    ngAfterViewInit(){ ++this.count; }
    ```
    
    两种方式都实现了数据的修改，但是第二种方式Angular会抛出错误提示，因为此时视图已经初始化完成，Angular不允许再修改数据。



  [1]: https://blog.angularindepth.com/angulars-digest-is-reborn-in-the-newer-version-of-angular-718a961ebd3e
  [2]: https://teropa.info/build-your-own-angular/
  [3]: https://segmentfault.com/a/1190000013502022
  [4]: https://blog.angularindepth.com/everything-you-need-to-know-about-the-expressionchangedafterithasbeencheckederror-error-e3fd9ce7dbb4
  [5]: https://juejin.im/post/58d0ff6f1b69e6006b8fd4e9
  [6]: https://segmentfault.com/a/1190000013233792
  [7]: https://www.youtube.com/watch?v=jvKGQSFQf10&feature=youtu.be
  [8]: /img/bV4O60