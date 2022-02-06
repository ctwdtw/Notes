## 使用 Decorator Pattern 管理橫切關注點 (cross-cutting-concern) 的程式碼 - 以 threading 為例。

### 摘要：
本篇筆記介紹應用 `Decorator Pattern` 來處理 threading 的相關問題。筆記一開始說明了散落在 UI 層或 Service 層的 threading code 造成的問題，接著用示範了如何使用 Decorator Pattern 來管理 threading 相關的 code。文末最後還簡介了 `Decorator` 可能的其他應用場景，例如 logging (日誌) 和 authentication (認證) 等 cross-cutting concern (橫切關注點) 的相關問題。

### 狀況：
作為一個 iOS dev. ，和 threading 打交道是我們常會遇到的問題。我們習慣將 App 至少分兩層 - UI Layer 和 Model Layer (還可以再細分，為了簡化，筆記中把整個 non-ui 的部分稱之為 Model Layer, 並將 Model Layer 中讀取資料的物件稱為 Service Object)。因為 UIKit 限制我們在 main thread or main queue 去使用 UIKit 大部分的 class [1], 我們可能會寫下這樣的 code:

```Swift
class ProfileViewController {
    private let profileService: ProfileService
    private let profileView: ProfileView
    private let errorView: ErrorView
    ...
    ...
    ...
    override func viewDidLoad() {
        profileService.getProfile() { result in
            DispatchQueue.main.async {
                do {
                    let profile = try result.get()
                    profileView.username = profile.username
                    profileView.address = profile.address

                } catch {
                    errorView.show(msg: error.localizedDescription)
    
                }
            }
        }
    }
}
```

這樣的 code 看起來不太複雜，還算可以。然而，隨著需求增加，把 threading 的邏輯放在 ViewController 裡面，可能變成一個不恰當的做法。例如我們可能有對 `ProfileService` 的不同實作，這實作本身可以有不同的 threading 細節，例如從 backend 或本地端撈取資料的 `remote profile service`, `local profile service` 跑在 background thread 較為合適，作為例外狀況替代路徑的 `in-memory cache service` 和作為測試替身的 `spy profile service` 則可以跑在 main thread 中。

因此，每次都在 UI 層加入 `DispatchQueue.main.async { ... }` 有可能造成不必要的浪費 (Service Object 可能原本就跑在 main thread 上)，或是當 Model 層的合作物件增加，情況變複雜後，programmer 開始在 UI 層任意做 Dispatch 的動作，造成 code 的執行在時間順序上變得難以理解。

一個可能的解法是把 Threading 的邏輯放到 model 層，事實上，這是一些知名開源套件 - 例如 `Alamofire` 的做法，預設 response 的 callback 在 main thread 上，並讓開發者可以自行決定：
```Swift
@discardableResult
    public func responseData(queue: DispatchQueue = .main,
                             dataPreprocessor: DataPreprocessor = DataResponseSerializer.defaultDataPreprocessor,
                             emptyResponseCodes: Set<Int> = DataResponseSerializer.defaultEmptyResponseCodes,
                             emptyRequestMethods: Set<HTTPMethod> = DataResponseSerializer.defaultEmptyRequestMethods,
                             completionHandler: @escaping (AFDataResponse<Data>) -> Void) -> Self {
        response(queue: queue,
                 responseSerializer: DataResponseSerializer(dataPreprocessor: dataPreprocessor,
                                                            emptyResponseCodes: emptyResponseCodes,
                                                            emptyRequestMethods: emptyRequestMethods),
                 completionHandler: completionHandler)
    }
```

如果我們有好的套件，我們當然可以選擇站在巨人的肩膀上；如果我們承接的 code base 中沒有像 Alamofire 這樣良好的設計，我們就必須打開既有的 model 層物件作修改，或是在無法修改這些物件的時候，另覓出路。

### 行動：
為了降低 model 層物件和 UI 的複雜度，我們需要有一個單一的物件來管理 threading 的邏輯 (遵從`單一職責原理`)，並且這些邏輯可以被添加在 model layer 的物件上。根據這個思路，打開我們的設計模式工具箱，`Decorator Pattern` 會是一個可能的選擇 [2]：
> 《DESIGN PATTERNS》一書中對 Decorator 模式的意圖是如此敘述的：動態地給一個物件增加一些額外的職責。就增加功能來說，Decorator 模式必生成子類別更為靈活。 - 《設計模式的活用與解析》 pp. 250

我們可以實作以下的 `MainQueueDecorator`

```Swift
class MainQueueDecorator<T> {
    private let decoratee: T
    init(decoratee: T) {
        self.decoratee = decoratee
    }
    
    func dispatch(action: @escaping () -> Void) {
        guard Thread.isMainThread else {
            DispatchQueue.main.async {
                action()
            }
            
            return
        }
        
        action()
    }
}

extension MainQueueDecorator: ProfileService where T == ProfileService {
    func getProfile(complete: @escaping (Result<Profile, Error>) -> Void) {
        decoratee.getProfile { [weak self] result in
            self?.dispatch(action: {
                complete(result)
            })
        }
    }
}

```
如此一來，只要我們在建構 `ProfileViewController` 的時候，替它注入經過 `MainQueueDecorator` 裝飾過的 `ProfileService`, 就能夠從`ProfileViewController` 中移除 threading 的職責。當我們需要不同的 threading 策略的時候，也只需要修改 `MainQueueDecorator` 裡面的 code, 讓 View Controller 和 Service Object 對 threading 的邏輯變更都不受影響。

```Swift
let service: ProfileService = RemoteProfileService()
let mainQueueService = MainQueueDecorator(decoratee: service)
let vc = ProfileViewController(service: service)

class ProfileViewController {
    private let profileService: ProfileService
    private let profileView: ProfileView
    private let errorView: ErrorView
    ...
    ...
    ...
    override func viewDidLoad() {
        profileService.getProfile() { result in
            do {
                let profile = try result.get()
                profileView.username = profile.username
                profileView.address = profile.address
            } catch {
                errorView.show(msg: error.localizedDescription)
            }
        }
    }
}
```

### 結果：
我們使用了 `DecoratorPattern` 去管理了原本被放置在 `ProfileViewController` 裡的程式碼，事實上，這樣的程式碼除了被放置在 `ProfileViewController` 中，還可能散落在其他的 View Controller 中，這種無法依照分層結構 (例如 Clean Architecture 的分層 [4] ) 被妥善管理的程式碼，我們稱之為 cross-cutting concern (橫切關注點) [5] - 這種程式碼橫

根據維基百科, cross-cutting-concern 相關的問題還有 1. logging, 2.authentication

//4. threading 是不能分層的, 他可能在某個垂直的 feature 的某層中有, 也可能在某個垂直的 feature 的某層中沒有, 他橫切整個分層, 因此被稱為是 cross-cutting concern.

### 參考資料：
1. https://developer.apple.com/documentation/uikit 
2. 設計模式的解析與活用（Design Patterns Explained: A New Perspective on Object-Oriented Design, 2nd Edition）- 博碩文化, Alan Shalloway, James R. Trott
3. iOS Lead Essential Program - https://www.essentialdeveloper.com/
4. https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
5. https://en.wikipedia.org/wiki/Cross-cutting_concern
### 版本資訊:
- 0.1.0 2022/02/07