## 使用 Polymorphism 簡化 TableView DataSource 中的條件敘述

### 摘要：
本篇筆記介紹了 Replace Conditional with Polymorphism (以多型代替條件) 的重構技巧，並運用此技巧，將 `UITableViewDataSource` 中的 `switch statement` 移除掉，得到一個較為乾淨的 code.

### 狀況：
我們常在 code base 中看到如下的 tableView data source:
```Swift
enum Section: Int {
        case input = 0, todos, max
}
```
```Swift
override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    guard let section = Section(rawValue: section) else {
        fatalError()
    }
    switch section {
        case .input: return 1
        case .todos: return todos.count
        case .max: fatalError()
    }
}

override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    guard let section = Section(rawValue: indexPath.section) else {
        fatalError()
    }
    
    switch section {
    case .input:
        let cell = tableView.dequeueReusableCell(withIdentifier: inputCellReuseId, for: indexPath) as! TableViewInputCell
        cell.delegate = self
        return cell
    case .todos:
        let cell = tableView.dequeueReusableCell(withIdentifier: todoCellResueId, for: indexPath)
        cell.textLabel?.text = todos[indexPath.row]
        return cell
    default:
        fatalError()
    }
}

```
以上的 code 至少存在兩個問題：1. Fatal Error 的路徑永遠不會被執行到，徒增 code 閱讀上的困難。2. 每次增加新的 section, 就要把 tableView dataSource 打開來維護，隨著時間增加，dataSource 可能會越來越肥大，導致難以維護。

在這樣的災難發生之前，我們必須採取積極的行動，免得問題變的一發不可收拾。

### 行動：
寫出好的物件導向 (Object-oriented programming) 程式碼的關鍵之一在於找出物件的介面 [1, 2]，並對物件的介面傳送訊息；一但我們有了介面，就可以對介面發送同樣的訊息，而不需關心底層不同型別的實作，大幅降低 client 端 code 的複雜度。

觀察我們的 data source code, `.input` 和 `.todos` 就是不同型別的物件，data source 的 switch statement 判斷了這兩個型別後，再把用不同的實作
```Swift
let cell = tableView.dequeueReusableCell(withIdentifier: inputCellReuseId, for: indexPath) as! TableViewInputCell
cell.delegate = self
```
和
```Swift
let cell = tableView.dequeueReusableCell(withIdentifier: todoCellResueId, for: indexPath)
cell.textLabel?.text = todos[indexPath.row]
```
去回應 TableView 傳送給 data source 的 訊息：
```Swift
func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int

func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell
```

有了以上的分析，我們就可以建立 `ToDoSection` 和 `InputSection` 這兩個物件去回應 tableView dataSource 和 tableView delegate 的訊息：

```Swift
class ToDoSection: NSObject, UITableViewDataSource, UITableViewDelegate {
    private let presenter: TablePresenter
    
    init(presenter: TablePresenter) {
        self.presenter = presenter
    }
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        presenter.numberOfToDos
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: todoCellResueId, for: indexPath)
        cell.textLabel?.text = presenter.toDoViewData(at: indexPath.row).title
        return cell
    }
    
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        presenter.removeToDo(at: indexPath.row)
    }
}
```
```Swift
class InputSection: NSObject, UITableViewDataSource {
    private let presenter: TablePresenter
    init(presenter: TablePresenter) {
        self.presenter = presenter
    }
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        1
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: inputCellReuseId, for: indexPath) as! TableViewInputCell
        cell.presenter = presenter
        presenter.inputView = cell
        return cell
    }
}
```

接著，我們只要將 `InputSection` 和 `ToDoSection` 覆蓋上一層 `TableViewDataSourceDelegate` 的 `Protocol`, 再注入我們的 client - `TableViewController`, 就可以簡化我們的 client `TableViewController` 中的 data source 邏輯了：

```Swift
protocol TableViewDataSourceDelegate: UITableViewDataSource, UITableViewDelegate {}

extension ToDoSection: TableViewDataSourceDelegate {}
extension InputSection: TableViewDataSourceDelegate {}
```

```Swift
class ToDoUIComposer {
    static func compose(getToDo: @escaping (@escaping (GetToDoResult) -> Void) -> Void) -> (TableViewController, TablePresenter) {
        let navc = UIStoryboard(name: "Main", bundle: nil).instantiateInitialViewController() as! UINavigationController
        let sut = navc.children[0] as! TableViewController
        let presenter = TablePresenter(getTodo: getToDo)
        sut.presenter = presenter
        
        /// 注入型別為 `TableViewDataSourceDelegate` 的
        /// `InputSection` 和 `ToDoSection` 物件
        sut.sections = [
            InputSection(presenter: presenter),
            ToDoSection(presenter: presenter)
        ]
        
        presenter.titleView = WeakRefVirtualProxy(sut)
        presenter.addActionView = WeakRefVirtualProxy(sut)
        presenter.tableView = WeakRefVirtualProxy(sut)
        
        return (sut, presenter)
    }
}

```

```Swift
extension TableViewController {
    override func numberOfSections(in tableView: UITableView) -> Int {
        return sections.count
    }
    
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        sections[section].tableView(tableView, numberOfRowsInSection: section)
    }
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        sections[indexPath.section].tableView(tableView, cellForRowAt: indexPath)
    }
    
    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        sections[indexPath.section].tableView?(tableView, didSelectRowAt: indexPath)
    }
}
```

甚至，我們可以任意調整 factory method 中任意調整 `InputSection` 和 `ToDoSection` 的順序，而不需要改動到 `TableViewController`, 讓 `TableViewController` 對 TableView Section 的順序修改，滿足SOLID 中的開放封閉原則 (Open/Closed principle). 

### 結果：
透過尋找物件的介面，我們在把原本的充滿條件敘述的 code (也稱為 procedure code) 轉變成易讀，且容易修改的物件導向 code. 我們花的代價不高，只是多開幾個 class 而已，這樣的修改保護了我們的 code base 在未來調整 TableView 的 Section 的時候更有彈性，是一個值得且應該做的投資。

完整的程式碼可以在 [GitHub](https://github.com/ctwdtw/ToDoDemo/tree/illustrate-polymorphism) 上被找到。

### 參考資料：
1. Ruby物件導向設計實踐：敏捷入門 (Practical Object-Oriented Design in Ruby: An Agile Primer), CH4 - 博碩文博碩文化, Sandi Metz 
2. 設計模式的解析與活用（Design Patterns Explained: A New Perspective on Object-Oriented Design, 2nd Edition）, CH1 - 博碩文化, Alan Shalloway, James R. Trott
3. iOS Lead Essential Program - https://www.essentialdeveloper.com/

### 版本資訊:
- 0.1.0 2022/02/08