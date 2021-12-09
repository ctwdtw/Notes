## 使用 Composite & Decorator Pattern 實作資料的讀取和緩存

### 摘要：
本篇筆記介紹如何使用 `Decorator Pattern` 和 `Composite Pattern` 來重構 ViewController 中複雜的條件敘述。筆記一開始粗略地描述了用程序式寫法 (`Procedure code`) 實現需求的問題，接著說明使用物件導向的設計原則和前設計模式可以得到一個更好的解決更好的設計解決方案。文末


### 狀況：
在 app 開發的場景中，可能會遇到 PM 開出類似以下的需求：
1. app 可以連接網路，抓取 server 端最新的資料，顯示給 user。
2. app 可以將最新抓取的資料，保存到本地端。
3. app 可以檢查本地端保存的資料是否在期限內。
4. app 若本地端的資料在期限內，app 會優先顯示本地端的資料。
5. app 若本地端的資料過期，app 會抓取 server 端最新的資料。
6. app 若本地端的資料過期，抓取 server 端資料也失敗，會顯示目前的本地端資料。
5. app 中有預設的資料，若 app 首次啟動，且無網路，無本地端的資料時，app 會顯示此份資料。

- 定義問題：
為了完成需求，我們可能會憑直覺地在用條件敘述 (`if...else`) 在 `ViewController` 中寫下類似以下的 code：

```Swift
class ViewController {
    ...
    ...
    ...
    func loadData() {
        if networkingManager.hasConnectiviy {
            AF.request("https://data-url.com").responseData { [weak self] response in
                ...
                ...
                self?.cacheData(data)
            }

        } else {
            let data = fileManager.contents(atPath: cachePath)
            ...
            ...
        }
    }

    func cacheData(_ data: Data) {
        try? data.write(to: fileURL)
    }

}
```
然而隨著條件分支不斷地增加，以上的 code 會逐漸變得難以理解，且每當讀取資料相關的需求有更動時，我們都必須把 `ViewController` ( 簡稱 VC ) 打開來修改。換句話說把讀取資料的邏輯寫在 VC 中，讓我們的 code 對讀取資料的需求變更，是不滿足開放封閉原則 (`Open/Closed Principle`) 的。

此外，VC 本身也會攜帶「動畫轉場」、「響應 View 的事件」等與 View 相關的職責，在 VC 中加入資料讀取相關的邏輯，也違反了單一職責原則 (`Single Responsibility Principle`)。

### 行動：
遵循物件導向設計原則的建議，我們應該可以找到更好的設計，讓我們整體的 code 有更好的可讀性，並且可以應付可能的需求變化。

- 尋找介面：
觀察需求，我們不難發現 app 在不同的「條件」底下 (ex. 有無網路, 資料是否過期 ... etc.)，需要有類似的`行為` - 即「讀取資料」。這些類似的行為，可以用同一介面讓 VC 去依賴，VC本身只關心有一個介面讓它傳送`讀取資料`這個訊息VC 完全信賴介面背後的實作會好好地處理讀取資料的細節，它只告訴介面去執行讀取資料的任務，而不問資料處理的細節中，所遇到的問題 (ex. 有沒有網路，本地端資料有沒有過期 ... etc.)，此即物件導向中的 `Tell don't ask principle`：

```Swift
protocol DataLoader {
    typealias LoadDataComplete = (Result<Data, Error>) -> Void
    func loadData(completion: @escaping LoadDataComplete)
}
```
- 實作介面：
我們可以用 `DefaultLoader`, `LocalLoader`, `RemoteLoader` 這三個 class 分別去實作需求中「讀取預設資料」、「讀取網路端資料」和「讀取本地端資料」:

```Swift
class DefaultLoader: DataLoader {
    func loadData(completion: @escaping LoadDataComplete) {
         guard let url = Bundle(for: DefaultLoader.self)
            .url(forResource: "default_data",
                 withExtension: "json") else {
                completion(.failure(Error.noSuchFile))
                return
            }
        
        let data = try? Data(contentsOf: url)
        ...
        ...
        ...
    }
}
```
```Swift
class LocalLoader: DataLoader {
    private let cachePolicy: CachePolicy

    init(cachePolicy: CachePolicy) {
        self.cachePolicy = cachePolicy
    }

    func loadData(completion: @escaping LoadDataComplete) {
        guard let cachedData = fileManager.contents(atPath: cacheURL.path) else {
            complete(.failure(Error.EmptyCache))
            return
        }
        
        do {
            try cachePolicy.validate(cachedData)
            complete(.success(cachedData))

        } catch {
            complete(.failure(Error.expiredCache))
        }
    }
}
```
```Swift
class RemoteLoader: DataLoader {
    func loadData(completion: @escaping LoadDataComplete) {
        session.request("https://data-end-point").responseData { response in
            ...
            ...
            ...
        }
    }
}

```

我們可以在`RemoteLoader`加入繼續加入「將最新抓取的資料，保存到本地端」的需求。然而，這樣做會將`RemoteLoader`的職責變得不再單純 - 混入了「讀取資料」和「緩存資料」的邏輯。在`RemoteLoader`中混入這兩種邏輯並ㄧ定是不好的做法，依據我們對需求變化的了解程度與對程式碼可讀性的容忍度，我們可以選擇是否要將這兩個職責分離開來。例如，如果我們知道 pm 有計劃在產品推廣期結束後，將「離線閱讀資料」這個功能分離出來給「付費使用者」(premium user)的話，我們就有必要尋找比直接混入職責到 `RemoteLoader` 中更好的設計。

- 利用 Decorator Pattern 添加行為
我們想要增加「保存資料到本地端」的行為給 `RemoteLoader` 但又不希望對 `RemoteLoader` 進行直接修改。《設計模式的解析與活用》 一書中介紹並描述了 Decorator Pattern 的意圖：
> 動態地給一個物件增加一些額外的職責。就增加功能來說，Decorator Pattern 比生成子類別更為靈活。

這恰巧是我們需要的。利用 Decorator Pattern, 我們可以造出另一種遵循 `DataLoader` protocol 的物件，此物件同時具有 `RemoteLoader`原本的行為，和我們想要添加的特性：

```Swift
class DataLoaderCacheDecorator: DataLoader {
    ...
    ...
    ...

    private let decoratee: DataLoader
    
    init(decoratee: DataLoader) {
        self.decoratee = decoratee
    }
    
    func loadData(completion: @escaping LoadDataComplete) {
        decoratee.loadData { [weak self] result in
            guard let self = self else { return }
            
            do {
                let data = try result.get()
                try self.cacheData(data)
                
            } catch {
                ...
                ...
                ...
            }
        }
    }
    
    func cacheData(_ data: Data) throws {
        try data.write(to: fileURL)
    }
}

```
- 利用 Composite Pattern 組合條件
到目前為止，我們已經將需求中的職責分散到一個個的需求中了，現在我們需要將這些物件依照需求中安排的「條件」組合起來。仔細觀察需求，我們會發現需求中不斷地一再重複 `先走 xxx, 若 xxx 失敗，就改走 yyy` 的模式；換句話說，我們有個一再重複的 `primary/secondary` 結構將 `DataLoader` 的實作組織起來，每次的 secondary 都可以帶出下一組的 primary/secondary 結構，如同樹枝向上不斷地長出樹枝和樹葉一般。這種樹狀結構，在 Raywenderlich 的 《Design Patterns By Tutorial》一書中，被稱為 Composite Pattern：
> The composite pattern is a structural pattern that groups a set of objects into a tree structure so they may be manipulated as though they were one object 

如共該書中所介紹 composite 的使用情境，我們的 VC 並不知道 composite 的細節內容，只知道他是一個遵循 `DataLoader` protocol 的物件。

Composite Pattern 的實作，在我們的需求中非常容易：
```Swift
class DataLoaderWithFallbackComposite: DataLoader {
    private let primary: DataLoader
    private let fallback: DataLoader
    
    init(primary: DataLoader, fallback: DataLoader) {
        self.primary = primary
        self.fallback = fallback
    }
    
    func loadData(completion: @escaping LoadDataComplete) {
        primary.loadData { [weak self] result in
            guard let self = self else { return }
            
            do {
                let course = try result.get()
                completion(.success(course))
                
            } catch {
                self.fallback.loadData(completion: completion)
                
            }
        }
    }
}
```
我們只要接著用 `DataLoaderWithFallbackComposite` 把各種 `DataLoader` 組合起來，再注入給 ViewController, 就完成我們的需求了：

```Swift
let dataLoader = DataLoaderWithFallbackComposite(
    primary: DataLoaderWithFallbackComposite(
        primary: DataLoaderWithFallbackComposite(
            primary: LocalLoader(cachePolicy: TwentyFourHoursPolicy()),
            fallback: DataLoaderCacheDecorator(decoratee: RemoteLoader())
        ),
        fallback: LocalLoader(cachePolicy: NullPolicy())
    ),
    fallback: DefaultLoader()
)

let viewController = ViewController(dataLoader: dataLoader)

```
- 流式接口 (Fluent Interface)：
Fluent Interface 是由 Martin Fowler 和 Eric Evans 在 2005 提出的一種物件導向 API 的設計方法，目的是為了提高 code 的可讀性。實作 Fluent Interface API 的關鍵在於讓物件接收訊息之後將自己回傳出去，作為下一次接收訊息的物件：
```Swift
extension DataLoader {
    func fallback(to fallback: DataLoader) -> DataLoader {
        DataLoaderWithFallbackComposite(primary: self, fallback: fallback)
    }
}
```

```Swift
extension DataLoader {
    func cache() -> DataLoader {
        DataLoaderCacheDecorator(decoratee: self)
    }
}
```
藉由 Fluent Interface，我們可以將各種 `DataLoader` 的組合簡化如下：
```Swift
let loader = LocalLoader(cachePolicy: TwentyFourHoursPolicy())
            .fallback(to: RemoteLoader().cache())
            .fallback(to: LocalLoader(cachePolicy: NullPolicy()))
            .fallback(to: DefaultLoader())

let viewController = ViewController(dataLoader: dataLoader)
```
幾乎可以說是一目瞭然了。

### 結果：
遵從 `Single Responsibility`, `Tell Don't Ask` 等設計原則，我們移除了 VC 中複雜的條件敘述，且讓讀取資料相關的邏輯需要修改或新增時，VC 仍然是保持封閉的，我們需要修改的 code 只有新的 `DataLoader` 的實作 (ex. `InMemoryDataLoader`) 和 `DataLoader` 的組合。藉由 `Fluent Interface` 風格設計的 API, 後者可以很容易地被完成。 

### 參考資料：
1. iOS Lead Essential Program - https://www.essentialdeveloper.com/
2. 設計模式的解析與活用（Design Patterns Explained: A New Perspective on Object-Oriented Design, 2nd Edition）- 博碩文化, Alan Shalloway, James R. Trott  
3. Design Patterns by Tutorials - Raywenderlich Tutorial Team, Joshua Greene, et al. 
4. Fluent interface - WikiPedia

### 版本資訊:
0.1.0 2021/12/09 