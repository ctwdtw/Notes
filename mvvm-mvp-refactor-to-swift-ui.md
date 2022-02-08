## 使用 MVP UI Design Pattern 將基於 UIKit 的 Toy project 重構到 SwiftUI

### 摘要：
本篇筆記實驗如何使用 MVP Pattern 將基於 UIKit 和 MVC Pattern 的 legacy code 遷移到 iOS 平台新的 UI framweork - SwiftUI. 筆記一開始點出了使用 MVC 實作一個 toy project 帶來的問題，與 migrate 到 swiftUI 的動機，接著介紹了 MVVM, MVP Pattern 在 Clean Architecture 的分層結構中所處的位置，與加入 Presentation Layer 所帶來的好處，並為使用 MVP 或 MVVM pattern 進行 SwiftUI 的 migration 做了一個簡單的比較。筆記最後展示了使用 MVP 對一個 toy project 進行 migration 的結果。

### 狀況：
Apple 在 2019 年的 WWDC 推出了新的 UI framework - SwiftUI。隨著 iOS 15 的問世，SwiftUI 也越來越成熟。SwiftUI 提供宣告式 (declarative) 的 api, 讓開法者可以更有效率的進行 UI 開發，是時候開始將老舊的 codebase migrate 到 swiftUI 了。

我們選用 onevcat 王巍大大的 toy project [ToDoDemo](https://github.com/onevcat/ToDoDemo/blob/basic/ToDoDemo/TableViewController.swift#L6) 作為實驗的對象，onevcat 在他的 blog [[8](https://onevcat.com/2017/07/state-based-viewcontroller/)] 中詳細介紹了在僅僅 100 多行的 MVC 實作中隱藏了哪些暗潮洶湧的 code smell, 並將之重構到單向數據流的函數式 View Controller. 本篇筆記不再詳細覆述這些 code smell, 僅指出散落在 line [24](https://github.com/onevcat/ToDoDemo/blob/dd66572689b1de5e10c99a5fcecd1816d99c1cb8/ToDoDemo/TableViewController.swift#L24), [29](https://github.com/onevcat/ToDoDemo/blob/dd66572689b1de5e10c99a5fcecd1816d99c1cb8/ToDoDemo/TableViewController.swift#L29), [74](https://github.com/onevcat/ToDoDemo/blob/dd66572689b1de5e10c99a5fcecd1816d99c1cb8/ToDoDemo/TableViewController.swift#L74), [86](https://github.com/onevcat/ToDoDemo/blob/dd66572689b1de5e10c99a5fcecd1816d99c1cb8/ToDoDemo/TableViewController.swift#L86) 的 presentation logic 即使抽取成 `TableViewController` 中的一個函數，仍然會和 UIKit 耦合在一起。我們需要一個新的物件去讓這些被抽取出來的函數能夠安身立命，而且這個物件是不能 import UIKit 的，如果能做到這點，我們就能把 UIKit 相關的 code 都隔離開來，並且替換成 SwiftUI。

### 行動：
參考 Uncle Bob 的 Clean Architecture 分層結構 [[1](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)]，我們發現 Interface Adapter (綠色) 的 Presenter 就是我們需要的物件 (注意圖中洋蔥圈的依賴方向，外層的洋蔥圈依賴裡面一層的洋蔥圈，而且不可跨層)：

![Diagram for Clean Architecture](https://blog.cleancoder.com/uncle-bob/images/2012-08-13-the-clean-architecture/CleanArchitecture.jpg)

事實上，這樣的 UI Pattern 在 mobile app 的生態圈已經被提出來過，分別就是我們常聽到的 MVVM [[2](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel)] 和 MVP [[3](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter)] 架構。事實上，MVVM 和 MVP 嚴格說起來不能算是mobile app 的整個架構 (architecture), 只能算是處理 mobile app 中 UI 部分的邏輯的 Design Pattern, 整個 mobile app 還會包含業務邏輯 (business logic, use cases), 導航 (Navigation, Routing) 等邏輯，當這些邏輯出現的時候，我們會再需要其他的物件去分拆這些職責。[[4](https://chiahsien.github.io/post/common-ios-architecture-from-mvc-to-viper-with-redux/)]

把 MVVM 和 MVP 的 class diagram 畫出來，並在不同 layer 上的物件塗上對應 Bob Clean Architecture 分層結構的顏色，就能夠一目瞭然了：

![Class Diagram for MVVM & MVP](https://raw.githubusercontent.com/ctwdtw/Notes/develop/mvvm-mvp.png)

不難發現，MVVM 和 MVP 在結構上是非常類似的，都有一個物件 (ViewMode/Presenter) 來把 model 層的資料介接給 UI 層來顯示。

差別只在於 MVVM 通常會搭配特定的 binding framework, 因此 ViewModel 會和 3rd. party framework (ex. RxSwift＆RxRelay/Combine) 有另外的耦合，UI 層的物件也需要依賴 3rd. party framwork 的 extension (ex. RxCocoa/CombineCocoa) 來讓 binding 的工作簡化：

> Declarative data and command-binding are implicit in the MVVM pattern. In the Microsoft solution stack, the binder is a markup language called XAML.[9] The binder frees the developer from being obliged to write boiler-plate logic to synchronize the view model and view. When implemented outside of the Microsoft stack, the presence of a declarative data binding technology is what makes this pattern possible,[5][10] and <B>without a binder, one would typically use MVP or MVC instead and have to write more boilerplate (or generate it with some other tool).</B> - Model–view–viewmodel, Wikipedia.

而 MVP 則是切出了一個 `AbstractView` 的介面 (interface, 即 Swift 中的 `Protocol`), 讓 UI Layer 的物件去實作行為。 Presenter 則創建了 UI 層物件可以 access 的 `ViewData` object (pure data, no behavior), 並透過這個 `AbstractView` interface 傳遞訊息給 UI Layer 的物件。不同的 UI framwork 都可以去實作這個 `AbstractView` 的介面，讓 Presenter 和 model 可以被重用： 

> MVP is a user interface architectural pattern engineered to facilitate automated unit testing and improve the separation of concerns in presentation logic:
a. The model is an interface defining the data to be displayed or otherwise acted upon in the user interface.
b. <B>The view is a passive interface that displays data (the model) and routes user commands (events) to the presenter to act upon that data.</B>
c. The presenter acts upon the model and the view. It retrieves data from repositories (the model), and formats it for display in the view. Model-view-presenter, Wikipedia.

Wikipedia 中 Ｃ# 的 MVP 範例：
```csharp
public class DomainView : IDomainView
{
    private IDomainPresenter _domainPresenter = null;

    /// <summary>Constructor.</summary>
    public DomainView()
    {
        _domainPresenter = new ConcreteDomainPresenter(this);
    }
}
```

如同 Wikipedia 的詞條所描述，MVVM 的 binding framwork (ex. RxSwift/Combine) 讓開發者用統一的宣告式 (declarative) API 來定義資料流 (stream)，並提供了一系列的 operator 對這些 stream 進行操作，讓我們的腦袋可以專注在這些 stream 的互動所定義的業務邏輯，而不需要花心思在個別 stream 的不同實作細節 (ex. IBAction, KVO, closure callback) 上。

André Staltz 的 `The introduction to Reactive Programming you've been missing` [[5](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)] 為這些 binding framwork 背後的 motivation 做了詳細的解釋：

> Reactive programming is programming with asynchronous data streams.
... You are able to create data streams of anything, not just from click and hover events. Streams are cheap and ubiquitous, anything can be a stream: variables, user inputs, properties, caches, data structures, etc. For example, imagine your Twitter feed would be a data stream in the same fashion that click events are. You can listen to that stream and react accordingly. On top of that, <B> you are given an amazing toolbox of functions to combine, create and filter any of those streams </B> - The introduction to Reactive Programming you've been missing

> Why should I consider adopting RP?
Reactive Programming raises the level of abstraction of your code so <B> you can focus on the interdependence of events that define the business logic </B>, rather than having to constantly fiddle with a large amount of implementation details. Code in RP will likely be more concise. - The introduction to Reactive Programming you've been missing

> The Combine framework provides a declarative Swift API for processing values over time. These values can represent many kinds of asynchronous events. - The Combine framework documentation

因此，若我們的 app 中有很複雜的 async stream 在交互影響 (ex WWDC 2019, session 722 中的 Login Screen), 選擇使用 MVVM pattern 會較為合理，但由於 ViewModel, ui component 甚至 model layer 都會和特定的 framework 耦合在一起, 將來要替換 framework 的時候，成本也可能跟著升高 (ex. 使用 RxCombine migrate from RxSwift to Combine [[6](https://github.com/CombineCommunity/RxCombine)] )。

我們選用 MVP 作為我們 migration to swfitUI 實驗的 pattern, 讓 presenter 最多只依賴於 Foundation 中的資料結構。

Migration 的起手式，就是為 VC 加入 [unit testing](https://github.com/ctwdtw/ToDoDemo/blob/swiftUI/ToDoDemoTests/ToDoDemoTests.swift), 這裡只貼 testing case 的 interface, 完整的實作放在 [link](https://github.com/ctwdtw/ToDoDemo/blob/swiftUI/ToDoDemoTests/ToDoDemoTests.swift) 裡面：
```Swift
internal class ToDoDemoTests : XCTestCase {

    internal func test_viewDidLoad_requestToDos()

    internal func test_renderTwoSections()

    internal func test_renderOneTableViewInputCellOnInputSection()

    internal func test_renderEmptyToDos_onEmptyToDo()

    internal func test_renderOneToDo_OnOneToDo()

    internal func test_renderManyToDo_onManyToDo()

    internal func test_renderToDoCounts()

    internal func test_renderAddAction()

    internal func test_renderAddedToDo_onAdd()

    internal func test_disableAddAction_afterAdd()

    internal func test_removeToDo_onRemove()

    internal func test_shouldHaveNoMemoryLeakAndNoCrash_onGetToDoServiceEmitValueAfterSutDeallocated()
}
```

接著讓 SwiftUI View 去實作各個 Abstract View Protocol (注意 SwiftUI 提供的各種 property wrapper 都被集中在 view layer, `Presenter` 是不需要 import SwiftUI 的 )：

```Swift
struct ToDoListView: View {
    private class ToDoListViewModel: ObservableObject {
        @Published
        var title = ""
        
        @Published
        var todos: [ToDoViewData] = []
        
        @Published
        var inputText = ""
        
        @Published
        var addActionEnabled = false
    }
    
    @ObservedObject private var viewModel: ToDoListViewModel = ToDoListViewModel()
    
    var body: some View {
        NavigationView {
            List {
                SwiftUI.Section("input") {
                    TextField(viewModel.inputText, text: $viewModel.inputText)
                        .onChange(of: viewModel.inputText) { newValue in
                            presenter.inputText(newValue)
                        }
                }
                
                SwiftUI.Section("todos") {
                    ForEach(Array(zip(viewModel.todos.indices, viewModel.todos)), id: \.1.id) { idx, todo in
                        Button(action: {
                            presenter.removeToDo(at: idx)
                            
                        }, label: {
                            Text(todo.title)
                            
                        })
                    }
                }
                
            }.navigationBarItems(
                trailing: Button(action: {
                    presenter.insertToDo(viewModel.inputText)
                    
                }) {
                    Image(systemName: "plus")
                }.disabled(!viewModel.addActionEnabled)
                
            ).navigationBarTitle(
                Text(viewModel.title),
                displayMode: .inline
            )
        }.onAppear {
            presenter.getInitialViewData()
            presenter.fetchToDo()
        }
        
    }
    
    let presenter: TablePresenter!
    
    init(presenter: TablePresenter) {
        self.presenter = presenter
    }
    
}

extension ToDoListView: TitleView {
    func didUpDateTitle(_ title: String) {
        viewModel.title = title
    }
}

extension ToDoListView: TableView {
    func didUpdateTable() {
        viewModel.todos = presenter.toDoViewDatas()
    }
}

extension ToDoListView: AddActionView {
    func didUpdateAddActionView(isEnabled: Bool) {
        viewModel.addActionEnabled = isEnabled
    }
}

extension ToDoListView: InputView {
    func didUpdateInputText(_ text: String) {
        viewModel.inputText = text
    }
}

//MARK: - TablePresenter + SwiftUI
extension TablePresenter {
    func toDoViewDatas() -> [ToDoViewData] {
        (0..<numberOfToDos).map { toDoViewData(at:$0) }
    }
}

```

我們就可以在 `AppDelegate` 中決定要使用 SwiftUI 或是 UIKit 作為我們 GUI 介面的 Framework 了：

```Swift
import UIKit
import SwiftUI

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    let uikit = false
    
    var window: UIWindow?
    
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        if uikit {
            let (vc, _) = ToDoUIComposer.compose(getToDo: ToDoStore.shared.getToDoItems(completionHandler:))
            window?.rootViewController = UINavigationController(rootViewController: vc)
            
        } else {
            let (listView, _) = ToDoSwiftUIComposer.compose(getToDo: ToDoStore.shared.getToDoItems(completionHandler:))
            window?.rootViewController = UIHostingController(rootView: listView)
        }
        
        return true
    }
}

```

### 結果：
使用 MVP pattern, 我們將基於 UIKit 的 MVC 的 toy project 成功地 migrate 到 swiftUI。完整的 code 可以在 [GitHub](https://github.com/ctwdtw/ToDoDemo/tree/swiftUI) 上照到。 MVP 的 migration 策略適用於本身沒有引入 (RxSwift, RxCocoa)/(Combine, CombineCocoa) 的專案，如果專案本身為了簡化 async 邏輯的交互影響 (interdependency) 已經引入了 reactive programming 的套件，則需要再考慮其他的 migration 策略 [[7](https://github.com/CombineCommunity/RxCombine/issues/4)]。

### 參考資料：
[1] Clean Architecture - https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
[2] MVVM - https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel
[3] MVP - https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter
[4] 漫談 iOS 架構：從 MVC 到 VIPER，以及 Redux - https://chiahsien.github.io/post/common-ios-architecture-from-mvc-to-viper-with-redux/
[5] The introduction to Reactive Programming you've been missing - https://gist.github.com/staltz/868e7e9bc2a7b8c1f754
[6] RxCombine - https://github.com/CombineCommunity/RxCombine
[7] Bridging RxSwift to SwiftUI - https://github.com/CombineCommunity/RxCombine/issues/4
[8] 單向數據流的函數式 View Controller - https://onevcat.com/2017/07/state-based-viewcontroller/
[9] iOS Lead Essential Program - https://www.essentialdeveloper.com/

### 版本資訊:
- 0.1.0 2022/02/08