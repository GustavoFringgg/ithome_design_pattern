# 柴咖啡系統 — 程式碼演進追蹤表

這份文件不是鐵人賽文章，是給作者自己用的檢查表

目的：每一天的故事都會讓「柴咖啡系統」裡的某些類別/介面長出新的樣子，或是被重新定義。這份表只記錄**當天故事結尾、阿柴改完之後的最終版本**，不記錄過程中的壞版本／半成品，方便之後寫新的一天時，回頭查「這個名字之前是不是已經用過、長什麼樣子」，避免像 `IDrink`／`Latte` 那樣同一個名字在不同天被定義成完全不同的東西

每加一天，就在下面續寫一個章節，並視情況更新最上面的「累積總覽表」

---

## 累積總覽表（目前寫到 Day 23）

| 名稱 | 種類 | 首次出現 | 目前定案（最後一次被改的那天） | 目前定案的簽章／欄位 |
|---|---|---|---|---|
| `Order` | class | Day 3（隱含） | 🔴 Day 16 | **確認衝突**：Day 3-4 隱含的 `Order` 只提過 `Items: List<OrderItem>`、`Date`；Day 16 給出完整定義卻是 `OrderId`、`Status: OrderStatus`、`TotalPrice: int`、`Items: List<OrderItem>`——**沒有 `Date` 欄位**，且新增了 Day 3-8 從沒出現過的 `OrderId`／`Status`／`TotalPrice` |
| `OrderItem` | class | Day 3（隱含） | 🔴 Day 16 | **確認衝突**：Day 4 隱含的 `OrderItem` 是 `Price`、`Quantity`、`DrinkType`（enum）、`OatMilkAddOn`（bool）；Day 16 給出完整定義卻變成 `DrinkName: string`、`Quantity: int`——**沒有 `Price`、`DrinkType`、`OatMilkAddOn`**，用字串代表飲料而不是 Day 4 的 `DrinkType` enum，也是柴咖啡系統第三種「怎麼代表一杯飲料」的方式（enum／`IDrink`多型類別／純字串） |
| `OrderCalculator` | class | Day 3 | Day 4（Day 17 有引用） | `Calculate(Order order): decimal`（Day 4 起改成建構子注入 `IEnumerable<IDrinkPricing>`，內部邏輯已跟 Day 3 版本不同，但方法簽章沒變）——⚠️ Day 17 的 `CheckoutFacade` 直接呼叫 `_calculator.Calculate(order)`，但 Day 4 版本的計算邏輯要靠 `item.Price`／`item.DrinkType` 去找對應的 `IDrinkPricing`；Day 16 重新定義的 `OrderItem` 已經沒有 `Price`／`DrinkType` 這兩個欄位了，兩邊接不起來，這條依賴鏈實際上兜不攏 |
| `ReceiptPrinter` | class | Day 3 | Day 3 | `Print(Order order, decimal total): void` |
| `LineNotifier` | class | Day 3 | Day 7（Day 17 有引用） | ⚠️ `INotifier.Notify(decimal total): void`——Day 3 原本簽章是 `async Task Notify(decimal total)`，Day 7 改實作 `INotifier` 之後變成 `void Notify(decimal total)`、內部呼叫 `PostAsync` 但沒有 `await`，兩天的簽章不一致；Day 17 的 `CheckoutFacade` 又直接依賴**具體類別** `LineNotifier`（不是 `INotifier` 介面，等於繞過 Day 7 學到的 DIP），而且寫 `await _notifier.Notify(total)`，把它當成回傳 `Task` 在用——這跟 Day 7 訂的 `void` 版本又對不上，三天各自認定的簽章都不一樣 |
| `OrderRepository` | class | Day 3 | Day 7 | 改實作 `IOrderRepository`，`Save(Order order): void` 簽章不變 |
| `OrderService` | class | Day 3 | Day 3 | 組合 `OrderCalculator`、`ReceiptPrinter`、`LineNotifier`、`OrderRepository`，`ProcessOrder(Order order): Task`（Day 7 之後 `LineNotifier`／`OrderRepository` 改實作介面，這裡沒有同步更新建構子型別，回頭潤稿時要注意） |
| `DrinkType` | enum（隱含，未完整定義） | Day 4 | Day 4 | 至少包含 `Latte`、`Americano`、`MatchaLatte` |
| `IDrinkPricing` | interface | Day 4 | Day 4 | `CanHandle(DrinkType type): bool`、`Calculate(OrderItem item): decimal` |
| `LattePricing` / `AmericanoPricing` / `MatchaLattePricing` | class | Day 4 | Day 4 | 皆實作 `IDrinkPricing` |
| `Discount` | 抽象類別 | Day 5 | ⚠️ Day 5 內就被棄用 | `Apply(decimal price): decimal`——被證明設計錯誤（`BuyOneGetOneDiscount` 硬要繼承會被迫丟例外），**不要在之後的天數延用這個名字** |
| `IPromotion` | interface | Day 6 | ⚠️ Day 6 內就被棄用 | `Apply(Order order, decimal currentTotal): decimal`、`RedeemPoints(int points): decimal`——示範「介面塞太多職責」的錯誤設計，**不要在之後的天數延用這個名字** |
| `IDiscount` | interface | Day 5 | Day 6 | `Apply(Order order, decimal currentTotal): decimal`——簽章從 Day 5 到 Day 6 沒變，Day 6 只是把它從 `IPromotion` 的岔路繞回來，重新確認這是「結帳流程」專用的介面 |
| `IPointRedeemable` | interface | Day 6 | Day 6 | `RedeemPoints(int points): decimal`——「點數兌換頁面」專用，跟 `IDiscount` 分開 |
| `PercentageDiscount` / `FixedAmountDiscount` / `BuyOneGetOneDiscount` | class | Day 5 | Day 6 | 皆實作 `IDiscount`（Day 6 一度改實作 `IPromotion`，最後改回 `IDiscount`） |
| `PointsRedemption` | class | Day 6 | Day 6 | 實作 `IPointRedeemable` |
| `IOrderRepository` | interface | Day 7 | Day 7 | `Save(Order order): void` |
| `INotifier` | interface | Day 7 | Day 7 | `Notify(decimal total): void` |
| `CheckoutService` | class | Day 5 | Day 7 | 建構子：`(OrderCalculator calculator, IOrderRepository repository, INotifier notifier)`；`Checkout(Order order, IDiscount discount): decimal`——Day 5 版本只依賴 `OrderCalculator`，Day 7 起新增 `IOrderRepository`／`INotifier` 兩個依賴，且 `Checkout()` 內部多做了 `_repository.Save()`、`_notifier.Notify()` 兩件事 |
| `FakeOrderRepository` / `FakeNotifier` | class（測試替身，Day 7 示範用） | Day 7 | Day 7 | 分別實作 `IOrderRepository`／`INotifier`，只在記憶體裡記錄，不碰真的資料庫/LINE API |
| `IDrink` | interface | Day 9 | Day 9 | ⚠️ `Name { get; }`、`Price { get; }`——這是柴咖啡系統第二次替「飲料」建模，跟 Day 4 的 `DrinkType` enum + `IDrinkPricing` 是完全不同的機制，兩套之間目前沒有任何銜接說明（CLAUDE.md 已知待辦，Day 4-8 暫緩回頭改） |
| `Latte` / `Americano` | class | Day 9 | Day 9 | 皆實作 `IDrink`，`Latte`：`Name => "拿鐵"`、`Price => 90`；`Americano`：`Name => "美式"`、`Price => 70`——⚠️ 已知 Day 11（Builder）會把 `Latte` 重新定義成完全不同的欄位（`Size`/`IceLevel`/`SugarLevel`/`MilkType`，不再實作 `IDrink`），寫 Day 11 段落時記得處理這個撞名 |
| `DrinkFactory` | class | Day 9 | ⚠️ Day 9 內就被棄用 | `CreateDrink(string type): IDrink`——示範用的 Simple Factory 版本，被同一天的 Factory Method 取代，**不要在之後的天數延用這個名字** |
| `DrinkStore` | 抽象類別 | Day 9 | Day 9 | `abstract IDrink CreateDrink()`、`OrderDrink(): IDrink` |
| `LatteStore` / `AmericanoStore` / `OatMilkLatteStore` | class | Day 9 | Day 9 | 皆繼承 `DrinkStore` |
| `OatMilkLatte` | class | Day 9 | Day 9 | 實作 `IDrink`，`Name => "燕麥拿鐵"`、`Price => 110` |
| `Cashier` | class | Day 9 | Day 9 | `ProcessCustomerOrder(DrinkStore store): void` |
| `ISideItem` | interface | Day 10 | Day 10 | `Name { get; }` |
| `Bagel` / `Cake` | class | Day 10 | Day 10 | 皆實作 `ISideItem` |
| `ComboService` | class | Day 10 | Day 10 | `ServeCombo(IComboFactory factory): void`——同一天內從「壞版本」（`GetDrink(string)`／`GetSide(string)` 各自判斷字串）改版成這個最終版，壞版本不用記錄 |
| `IComboFactory` | interface | Day 10 | Day 10 | `CreateDrink(): IDrink`、`CreateSide(): ISideItem` |
| `BreakfastComboFactory` / `AfternoonTeaComboFactory` | class | Day 10 | Day 10 | 皆實作 `IComboFactory`，內部呼叫 `new Americano()`/`new Bagel()`、`new Latte()`/`new Cake()`，沿用 Day 9 的 `Latte`/`Americano`/`IDrink` |
| `Latte` | class | Day 9 | 🔴 Day 11 | **確認衝突**：Day 9/10 的 `Latte : IDrink`（`Name`、`Price`）在 Day 11 被整個重新定義成完全不同的形狀——不再實作 `IDrink`，改成 `Size: CupSize`、`IceLevel: IceLevel`、`SugarLevel: SugarLevel`、`MilkType: MilkType` 四個客製化欄位，建構子也從無參數屬性變成 `Latte(CupSize, IceLevel, SugarLevel, MilkType)`。同一個類別名稱，Day 9-10 用來代表「訂單裡的一個飲料品項」，Day 11 用來代表「一杯飲料的客製化規格」，兩者無法共存 |
| `CupSize` / `IceLevel` / `SugarLevel` / `MilkType` | enum | Day 11 | Day 11 | `CupSize{Medium,Large}`、`IceLevel{Normal,Less,None}`、`SugarLevel{Full,Half,None}`、`MilkType{Regular,Oat}`——⚠️ `IceLevel` 這個 enum 型別名稱跟 `Latte.IceLevel` 這個屬性名稱撞名（C# 語法上合法，但容易讓人一時看錯是型別還是屬性） |
| `LatteBuilder` | class | Day 11 | Day 11 | Fluent Builder：`SetSize`/`SetIce`/`SetSugar`/`SetMilk`（皆回傳 `this`）、`Build(): Latte` |
| `SimpleLatte` | class（Day 11「取捨」段落示範用，非主線） | Day 11 | Day 11 | `SimpleLatte(string size = "中杯", string iceLevel = "正常冰")`，用來示範「選項不多時具名引數就夠用，不必上 Builder」 |
| `MachineDriver` | class | Day 12 | Day 12 | `sealed class`，`static MachineDriver Instance`（`Lazy<T>` 包裝）、私有建構子、`Brew(string drinkName): void` |
| `OrderCounter` | class | Day 12 | Day 12 | ⚠️ `TakeOrder(string drinkName): void`——這是第 5 個「前台接單」類的類別了（`OrderService`／`CheckoutService`／`Cashier`／`ComboService`／`OrderCounter`），彼此之間目前沒有交代關係，回頭潤稿時可以考慮這幾個是不是該收斂 |
| `ChaiLatte` | class | Day 13 | Day 13 | 「招牌柴拿鐵」原型物件，跟 Day 11 的 `Latte` 是不同概念（不衝突）：`BeanOrigin`/`RoastLevel`/`BrewTemp`/`MilkRatio`/`FoamDensity`（固定基底參數，皆非 enum）、`Sugar`/`Ice`/`MilkType`（客製化選項，皆是 `string`，不是 enum）、`Toppings: List<string>`、`Clone(): ChaiLatte`（深拷貝 `Toppings`） |
| `PaymentResult` | class | Day 15 | Day 15 | `Success: bool`、`TransactionId: string`、`InvoiceNumber: string`（開發票後才有值） |
| `IPaymentProcessor` | interface | Day 15 | Day 15 | `Charge(string orderId, int amount): PaymentResult` |
| `PaymentService` | class | Day 15 | Day 15 | 實作 `IPaymentProcessor` |
| `PaymentDecorator` | 抽象類別 | Day 15 | Day 15 | 實作 `IPaymentProcessor`，內部持有一個 `IPaymentProcessor`，預設轉呼叫 |
| `InvoiceDecorator` / `SalesReportDecorator` | class | Day 15 | Day 15 | 皆繼承 `PaymentDecorator` |
| `CheckoutController` | class | Day 15 | 🔴 Day 18（共 3 種定義） | **確認衝突**：Day 15 依賴 `IPaymentProcessor`、`Checkout(string orderId, int amount): void`；Day 17 依賴 `CheckoutFacade`、`Checkout(Order order): Task<PaymentResult>`；Day 18 依賴 `IMemberPointsService`、`ApplyMemberDiscount(string phoneNumber): void`。三天各自獨立定義，沒有互相銜接，也沒有交代這三個版本是「同一個類別逐步演進」還是「三個不同職責硬用同一個名字」 |
| `OrderStatus` | enum | Day 16 | Day 16 | `Pending`、`Preparing`、`Completed` |
| `KitchenService` | class | Day 16 | Day 16 | `ReceiveOrder(Order order): void` |
| `OrderController` | class | Day 16 | Day 16 | `HandleIncomingOrder(string platform, object payload): void`——外送平台 webhook 接單用，跟前面幾個「前台接單」類別（`OrderService`/`CheckoutService`/`Cashier`/`ComboService`/`OrderCounter`）職責不同（這個是收外部平台訂單，不是門市點餐），但名字風格很像，建議之後統一命名慣例 |
| `IOrderAdapter` | interface | Day 16 | Day 16 | `ToOrder(): Order` |
| `QberEatsOrderAdapter` / `FoodDogOrderAdapter` | class | Day 16 | Day 16 | 皆實作 `IOrderAdapter` |
| `QberEatsOrderPayload` / `QberEatsProduct` / `FoodDogOrderPayload` / `FoodDogItem` | class | Day 16 | Day 16 | 外部平台的原始資料格式，不是柴咖啡內部模型 |
| `CheckoutFacade` | class | Day 17 | Day 17 | 建構子 `(OrderCalculator calculator, IPaymentProcessor paymentProcessor, LineNotifier notifier)`（⚠️ `LineNotifier` 是具體類別，不是 `INotifier`）；`Checkout(Order order): Task<PaymentResult>`，內部呼叫 `_calculator.Calculate(order)`、`_paymentProcessor.Charge(order.OrderId, (int)total)`、`await _notifier.Notify(total)` |
| `IMemberPointsService` | interface | Day 18 | Day 18 | `GetPoints(string phoneNumber): int` |
| `QberEatsMemberInfo` | class | Day 18 | Day 18 | `Name: string`、`Points: int`——模擬 Qber Eats 對外 API 回傳格式，跟 Day 16 的 `QberEatsOrderPayload` 系列是不同用途、不衝突 |
| `QberEatsMemberService` | class | Day 18 | Day 18 | 實作 `IMemberPointsService` |
| `IBindingVerifier` | interface | Day 18 | Day 18 | `IsVerified(string phoneNumber): bool` |
| `MemberPointsProxy` | class | Day 18 | Day 18 | 實作 `IMemberPointsService`，建構子 `(IMemberPointsService realService, IBindingVerifier bindingVerifier)`，內部做保護代理（驗證）+ 快取代理 |
| `StoreBindingVerifier` | class（文中只提到名字，沒有show完整程式碼） | Day 18 | Day 18 | 文中說是「查店內綁定資料表的實作」，`IBindingVerifier` 的其中一個實作，但沒有給出完整類別定義 |
| `IMenuComponent` | interface | Day 19 | Day 19 | `Display(int depth): void` |
| `MenuItem` | class | Day 19 | Day 19 | 這是柴咖啡系統**第一次**出現叫 `MenuItem` 的類別，不跟前面任何一天衝突；同一天內從壞版本（`Name`/`Price`/`Category: string` 扁平屬性）演進到最終版本（實作 `IMenuComponent`，改成私有欄位 `_name`/`_price`，透過建構子注入，`Category` 欄位整個拿掉） |
| `MenuCategory` | class | Day 19 | Day 19 | 實作 `IMenuComponent`，`_children: List<IMenuComponent>`，`Add(IMenuComponent child)`、`Display(int depth)` |
| `MenuPrinter` | class | Day 19 | ⚠️ Day 19 內就被棄用 | `PrintMenu(List<MenuItem> items): void`——示範用的字串切割版本，被同一天的 Composite 版本取代，**不要在之後的天數延用這個名字** |
| `Report` | 抽象類別 | Day 20 | Day 20 | 同一天內從壞版本（無參數 `Generate(): void`，繼承階層依「報表種類 × 匯出格式」寫死組合）演進到最終版本（`protected IExportFormat _exportFormat`，建構子注入，`Generate(): void`）——這是柴咖啡系統第一次用 `Report` 這個名字，不跟前面任何一天衝突 |
| `SalesReport` / `InventoryReport` / `MemberPointsReport` | class | Day 20 | Day 20 | 皆繼承最終版 `Report`，建構子吃 `IExportFormat`，`Generate()` 內部呼叫 `_exportFormat.Export(...)` |
| `IExportFormat` | interface | Day 20 | Day 20 | `Export(string content): void` |
| `PdfExport` / `ExcelExport` / `CsvExport` | class | Day 20 | Day 20 | 皆實作 `IExportFormat` |
| `DrinkSpec` | class | Day 21 | Day 21 | Flyweight（ConcreteFlyweight，簡化版沒拉獨立介面）：`Name`／`BasePrice`／`IconUrl`，只放不會因訂單而改變的內在狀態，建構子注入後三個屬性皆唯讀 |
| `DrinkSpecFactory` | class | Day 21 | Day 21 | FlyweightFactory：內部 `Dictionary<string, DrinkSpec> _pool`，`GetSpec(string name, int basePrice, string iconUrl): DrinkSpec`——依 `name` 查快取，查無才 `new` 並存入 `_pool` |
| `OrderLine` | class | Day 21 | Day 21 | 這是柴咖啡系統**第一次**出現叫 `OrderLine` 的類別，跟 Day 16 定案的 `OrderItem`（`DrinkName`/`Quantity`）是刻意取的不同名字、不衝突；同一天內從壞版本（`DrinkName`/`BasePrice`/`IconUrl`/`Sugar`/`Ice`/`Quantity` 六個欄位各自持有一份完整資料）演進到最終版本（`Spec: DrinkSpec`＋`Sugar`/`Ice`/`Quantity` 三個外在狀態，飲品資料改成共享的 `DrinkSpec` 參考） |
| `IPromotionStrategy` | interface | Day 22 | Day 22 | `Apply(Order order): int`——直接沿用 Day 16 定案的 `Order.TotalPrice`／`Order.Items[].Quantity`，不依賴 `OrderItem.Price`（目前沒有這個欄位）。刻意跟 Day 5/6（LSP/ISP）的 `IDiscount` 錯開命名，兩組介面彼此無關、不算衝突 |
| `PercentageOffStrategy` / `BuyOneGetOneStrategy` / `PointsRedemptionStrategy` | class | Day 22 | Day 22 | 皆實作 `IPromotionStrategy` |
| `PromotionContext` | class | Day 22 | Day 22 | 建構子注入 `IPromotionStrategy`，`SetStrategy(IPromotionStrategy strategy): void`（執行期可替換）、`CalculateFinalPrice(Order order): int` |
| `StoreCheckoutService` | class | Day 22 | ⚠️ Day 22 內就被棄用 | 同一天內從壞版本（`CalculateFinalPrice(Order order, string storeCode): int`，只有台北/台中兩個分支）演進到再壞版本（新增高雄店分支後簽章變成 `CalculateFinalPrice(Order order, string storeCode, int usedPoints): int`，`usedPoints` 只有高雄店用得到），被同一天的 Strategy 版本取代，**不要在之後的天數延用這個名字**；這是柴咖啡系統第 6 個「結帳/前台」類的類別（`OrderService`/`CheckoutService`/`Cashier`/`ComboService`/`OrderCounter`/`StoreCheckoutService`），但這次是總部端管理跨分店促銷計算，跟前面幾個門市端類別職責不同 |
| `IStockObserver` | interface | Day 23 | Day 23 | `OnStockLow(string itemName, int quantity): void` |
| `ProcurementNotifier` / `StoreManagerNotifier` / `HeadquartersDashboardNotifier` / `SmsProcurementNotifier` | class | Day 23 | Day 23 | 皆實作 `IStockObserver`；`SmsProcurementNotifier` 是高雄店專用、跟 `ProcurementNotifier` 做同一件事但改用簡訊 |
| `InventoryService` | class | Day 23 | Day 23 | 同一天內從壞版本（`CheckStock(itemName, quantity)` 內部寫死依序呼叫 `NotifyProcurement`/`NotifyStoreManager`/`NotifySyncToHeadquartersDashboard` 三個 private 方法）演進到最終版本（`_observers: List<IStockObserver>`，`Subscribe`/`Unsubscribe`，`CheckStock()` 只負責偵測庫存＋通知已訂閱的觀察者） |
| `IOrderObserver` | interface | Day 23 | Day 23 | `OnOrderCompleted(Order order): void`——直接沿用 Day 16 定案的 `Order`，補第二個 Subject 範例用，文中沒有寫出對應的 `OrderService` 訂閱端完整程式碼 |
| `CustomerNotifier` / `DeliveryPlatformSyncer` | class | Day 23 | Day 23 | 皆實作 `IOrderObserver` |

**非柴咖啡系統類別**（獨立示範用，跟柴咖啡系統無關，不算進上面的累積表）：
- Day 5（LSP）：`Vehicle`、`Toyota`、`Honda`、`Tesla`、`IRefuelable`、`IChargeable`
- Day 6（ISP）：`IMultiFunctionDevice`、`SimplePrinter`、`MultiFunctionPrinter`、`IPrinter`、`IScanner`、`IFax`
- Day 7（DIP）：`Bulb`、`Fan`、`Switch`、`ISwitchable`
- Day 11（Builder）：`Burger`、`BurgerBuilder`
- Day 12（Singleton）：`OfficePrinter`
- Day 13（Prototype）：`Resume`
- Day 20（Bridge）：`Vehicle`、`FuelCar`/`FuelMotorcycle`/`FuelTruck`、`ElectricCar`/`ElectricMotorcycle`/`ElectricTruck`（壞版本，示範繼承爆炸）→ 最終版本重新定義 `Vehicle`（Abstraction）、`Car`/`Motorcycle`（RefinedAbstraction）、`IPowerSource`（Implementor）、`FuelEngine`/`ElectricMotor`（ConcreteImplementor）——⚠️ 這裡的 `Vehicle` 跟 Day 5（LSP）示範用的 `Vehicle` 是同一個名字，但兩天都是「非柴咖啡系統類別」的獨立示範，各自獨立、不影響柴咖啡主線，不算衝突，只是提醒一下這兩天剛好都借了同一個生活化情境的名字
- Day 21（Flyweight）：`TreeType`（Flyweight/ConcreteFlyweight 簡化版，`Name`/`MeshData`/`TextureData`＋`Draw(int x, int y)`）、`TreeTypeFactory`（FlyweightFactory，`Dictionary<string, TreeType>` 快取池）、`Tree`（Context，持有外在狀態 `_x`/`_y` 與共享的 `TreeType` 參考）——遊戲森林場景類比，跟柴咖啡系統無關，也不跟前面任何一天的類別撞名
- Day 22（Strategy）：`IRouteStrategy`、`DrivingStrategy`/`TransitStrategy`/`WalkingStrategy`（ConcreteStrategy）、`NavigatorContext`（Context，`SetStrategy` 可執行期替換）——導航 App 選路線類比，跟柴咖啡系統無關，也不跟前面任何一天的類別撞名
- Day 23（Observer）：`IWeatherObserver`、`TrainOperator`/`School`/`ConvenienceStore`（ConcreteObserver）、`WeatherBureau`（Subject，`Subscribe`/`Unsubscribe`/`IssueHeavyRainAlert`）——氣象局豪雨特報類比，跟柴咖啡系統無關，也不跟前面任何一天的類別撞名

---

## Day 3｜SRP

**故事狀態**：阿柴接手小黃留下的 `OrderService`，原本一個方法塞了算金額、印收據、傳 LINE 通知、存資料庫四件事，改收據格式時意外弄壞 LINE 通知

**這天新增的類別**
- `OrderCalculator.Calculate(Order order): decimal` — 純加總 `order.Items.Sum(i => i.Price * i.Quantity)`，還沒有促銷邏輯
- `ReceiptPrinter.Print(Order order, decimal total): void`
- `LineNotifier.Notify(decimal total): Task`
- `OrderRepository.Save(Order order): void`
- `OrderService` 重構後：建構子組合上面四個，`ProcessOrder(Order order): Task` 依序呼叫

**下一天會被動到的地方**：`OrderCalculator.Calculate()` 的內部實作，Day 4 會整個換掉（改成委派給 `IDrinkPricing`），但對外的方法簽章維持不變

---

## Day 4｜OCP

**故事狀態**：`OrderCalculator.Calculate()` 裡塞了 `switch(item.DrinkType)`，每上一款新飲料就要多一個 case，改共用邏輯時漏改美式的九折，出現定價 bug

**這天新增/變更的類別**
- `IDrinkPricing` interface：`CanHandle(DrinkType type): bool`、`Calculate(OrderItem item): decimal`
- `LattePricing`、`AmericanoPricing`、`MatchaLattePricing` 分別實作 `IDrinkPricing`
- `OrderCalculator` **改版**：建構子改為注入 `IEnumerable<IDrinkPricing>`，`Calculate(Order order): decimal` 方法簽章不變，但內部改成迴圈找對應的 `IDrinkPricing` 委派計算，另外保留 `rawTotal`（滿額門檻用，不受個別飲料折扣影響）

**下一天會被動到的地方**：無（`CheckoutService`、`IDiscount` 是 Day 5 全新引入，`OrderCalculator` 只被依賴、沒有再被改）

---

## Day 5｜LSP

**故事狀態**：阿柴想比照 OCP 的精神處理促銷邏輯，設計了 `Discount` 抽象類別，結果「買一送一」塞不進 `Apply(decimal price)` 這個介面，被迫繼承後硬丟例外、呼叫端要多寫 `is` 判斷——這是 LSP 違反的典型症狀

**這天新增/變更的類別**
- `Discount` 抽象類別（`Apply(decimal price): decimal`）——**故事裡示範用的錯誤設計，同一天結尾就被棄用**，不要在之後的天數把這個名字撿回來用
- `IDiscount` interface（**最終正確版本**）：`Apply(Order order, decimal currentTotal): decimal`，把訂單整體跟目前金額都傳進去，讓「買一送一」這種需要看數量的促銷也能吃得到資料
- `PercentageDiscount`、`FixedAmountDiscount`、`BuyOneGetOneDiscount` 都改實作 `IDiscount`
- `CheckoutService` 新增：依賴 `OrderCalculator`（Day 4 版本），`Checkout(Order order, IDiscount discount): decimal`

**下一天會被動到的地方**：`IDiscount` 在 Day 6 一度被擴大改名成 `IPromotion`，故事結尾又拆回來，最終還是叫 `IDiscount`

---

## Day 6｜ISP

**故事狀態**：小黑想加集點折抵，阿柴貪方便把 `IDiscount` 擴大改名成 `IPromotion`，塞進 `RedeemPoints()`，結果 `PercentageDiscount` 這種跟集點無關的類別被迫實作用不到的方法（回傳假的 0），`PointsRedemption` 也被迫對 `Apply()` 丟例外——編譯期完全看不出來，直到結帳時手滑選到 `PointsRedemption` 才在執行期炸開

**這天新增/變更的類別**
- `IPromotion` interface（`Apply(...)` + `RedeemPoints(...)`）——**故事裡示範用的錯誤設計，同一天結尾就被棄用**，不要在之後的天數把這個名字撿回來用
- 故事結尾拆回兩個小介面：`IDiscount`（`Apply(Order order, decimal currentTotal): decimal`，簽章跟 Day 5 一樣，只是重新確認它只服務結帳流程）、`IPointRedeemable`（新增，`RedeemPoints(int points): decimal`，只服務點數兌換頁面）
- `PercentageDiscount`、`BuyOneGetOneDiscount` 改回實作 `IDiscount`
- `PointsRedemption` 新增，實作 `IPointRedeemable`

**下一天會被動到的地方**：`CheckoutService`——Day 7 會發現它建構子裡直接 `new OrderRepository()`、`new LineNotifier()`，違反 DIP，並補上 `IOrderRepository`／`INotifier` 兩個新介面

---

## Day 7｜DIP

**故事狀態**：顧問要阿柴幫 `CheckoutService` 補單元測試，結果測試一跑，真的寫進資料庫、真的發了一則 LINE 通知把老闆半夜吵醒——因為 `CheckoutService` 建構子裡直接 `new OrderRepository()`、`new LineNotifier()`，高層模組（結帳邏輯）直接綁死了低層模組（資料庫、LINE API）的具體實作，測試沒辦法插入假的版本

**這天新增/變更的類別**
- `IOrderRepository` interface（新增）：`Save(Order order): void`
- `INotifier` interface（新增）：`Notify(decimal total): void`
- `OrderRepository` 改實作 `IOrderRepository`，`Save()` 簽章不變
- `LineNotifier` 改實作 `INotifier`；⚠️ 簽章從 Day 3 的 `async Task Notify(decimal total)` 變成 `void Notify(decimal total)`，內部呼叫 `PostAsync` 但沒有 `await`，這是本篇故事碼裡自己前後不一致的地方，回頭潤稿時要挑一個版本統一（用 `Task`／`async` 是比較正確的寫法）
- `CheckoutService` 改版：建構子從 `(OrderCalculator calculator)` 擴充成 `(OrderCalculator calculator, IOrderRepository repository, INotifier notifier)`，`Checkout(Order order, IDiscount discount): decimal` 內部新增呼叫 `_repository.Save(order)`、`_notifier.Notify(total)`（Day 5 版本的 `Checkout()` 只有算錢跟套用折扣，沒有存檔、沒有通知，這兩個副作用是 Day 7 才加進去的）
- `FakeOrderRepository`、`FakeNotifier`（測試替身，只在這篇示範用，分別實作 `IOrderRepository`／`INotifier`）

**下一天會被動到的地方**：無，Day 8 是純觀念整理（KISS/YAGNI/DRY），沒有動到任何柴咖啡系統的程式碼

---

## Day 8｜SOLID 的取捨與衝突

**故事狀態**：店休日，沒有新劇情，純粹整理 SOLID 五原則各自「什麼時候該用、什麼時候該等痛了再用」，並帶入 KISS、YAGNI、DRY 三個更老牌的原則來制衡「用力過猛的 SOLID」

**這天新增/變更的類別**：無，沒有任何程式碼異動

---

## Day 9｜Factory Method

**故事狀態**：柴咖啡開店，阿柴翻開小黃留下的 Simple Factory 版本（`DrinkFactory.CreateDrink(string type)`，`if-else` 判斷字串），複習完 SOLID 後意識到這違反 OCP——小黑要加燕麥拿鐵，就得回頭改 `DrinkFactory` 內部的 `if-else`，於是改用 Factory Method：讓每種飲料各自有一個工廠子類別，新增飲料只要新增子類別，不用碰既有的工廠程式碼

**這天新增/變更的類別**
- `IDrink` interface（新增）：`Name { get; }`、`Price { get; }`——⚠️ 這是柴咖啡系統第二次替「飲料」建模，Day 4 已經用 `DrinkType` enum + `IDrinkPricing` 建過一次模，兩者目前並存、沒有交代關係（CLAUDE.md 已知這是待修正項目，Day 4-8 是舊版故事設定，之後才會回頭統一，暫不用主動處理）
- `Latte`、`Americano`（新增）：皆實作 `IDrink`——⚠️ 已知 Day 11（Builder）會把 `Latte` 這個名字重新定義成完全不同的形狀（客製化選項物件，不再實作 `IDrink`），到時候要處理撞名
- `DrinkFactory`（新增，示範用）：`CreateDrink(string type): IDrink`，**同一天結尾就被棄用**，被下面的 Factory Method 版本取代，不要在之後的天數延用這個名字
- `DrinkStore` 抽象類別（新增）：`abstract IDrink CreateDrink()`、`OrderDrink(): IDrink`（骨架流程：呼叫 `CreateDrink()`，印出點餐資訊）
- `LatteStore`、`AmericanoStore`（新增）：皆繼承 `DrinkStore`
- `OatMilkLatte`（新增）：實作 `IDrink`，`Name => "燕麥拿鐵"`、`Price => 110`
- `OatMilkLatteStore`（新增）：繼承 `DrinkStore`
- `Cashier`（新增）：`ProcessCustomerOrder(DrinkStore store): void`，只依賴抽象的 `DrinkStore`

**下一天會被動到的地方**：`IDrink`、`Latte`、`Americano`、`ISideItem` 會被 Day 10 直接沿用（套餐需要「飲料 + 點心」成套出貨）

---

## Day 10｜Abstract Factory

**故事狀態**：小黑想推早餐套餐（美式+貝果）、下午茶套餐（拿鐵+蛋糕），阿柴一開始寫兩個各自獨立的方法 `GetDrink(comboType)`、`GetSide(comboType)`，呼叫端要手動呼叫兩次、自己保證兩次傳的 `comboType` 一致——結果字串打錯（`"breakfast"` vs `"breakFast"`），組出「美式+蛋糕」的幽靈組合，編譯不會抓到這種錯誤

**這天新增/變更的類別**
- `ISideItem` interface（新增）：`Name { get; }`
- `Bagel`、`Cake`（新增）：皆實作 `ISideItem`
- `ComboService`（新增，同一天內從壞版本改到好版本）：壞版本是 `GetDrink(string comboType): IDrink`、`GetSide(string comboType): ISideItem`，各自獨立判斷字串；最終版本改成 `ServeCombo(IComboFactory factory): void`，只依賴抽象工廠介面
- `IComboFactory` interface（新增）：`CreateDrink(): IDrink`、`CreateSide(): ISideItem`
- `BreakfastComboFactory`、`AfternoonTeaComboFactory`（新增）：皆實作 `IComboFactory`，內部沿用 Day 9 的 `Latte`／`Americano`，以及這天新增的 `Bagel`／`Cake`

**下一天會被動到的地方**：無，Day 11（Builder）換一個全新情境（客製化飲料），不會再動到 `IComboFactory`／`ComboService` 這組

---

## Day 11｜Builder

**故事狀態**：客製化選項一多（甜度、冰量、奶類），小黃留下的 `Latte` 建構子塞四個參數，客人要「大杯、少冰、半糖」，呼叫端要記參數順序又要傳一堆用不到的預設值；小黃後來又疊了三個 telescoping constructor（`this(...)` 一路疊上去），可讀性越來越差，且下錯順序編譯器也抓不出來。改用 Fluent Builder + enum 選項解決

**這天新增/變更的類別**
- 🔴 `Latte`：**這天把 Day 9/10 的 `Latte : IDrink`（`Name`、`Price`）整個重新定義**成客製化規格物件，不再實作 `IDrink`，改成 `Size: CupSize`、`IceLevel: IceLevel`、`SugarLevel: SugarLevel`、`MilkType: MilkType` 四個欄位，建構子改成 `Latte(CupSize size, IceLevel iceLevel, SugarLevel sugarLevel, MilkType milkType)`，另外多了 `ToString()`。這是本次要你確認的正式衝突點——同一個類別名稱在兩個不同的建模階段代表完全不同的東西
- `CupSize`、`IceLevel`、`SugarLevel`、`MilkType`（新增，enum）：把原本 `string` 選項換成 enum，讓編譯器擋掉不合法的值
- `LatteBuilder`（新增）：private 欄位存四個選項的預設值，`SetSize`/`SetIce`/`SetSugar`/`SetMilk` 皆回傳 `this`，`Build(): Latte`
- `SimpleLatte`（新增，「Builder 的取捨」段落示範用）：`SimpleLatte(string size = "中杯", string iceLevel = "正常冰")`，用具名引數＋預設值示範「選項不多時不必上 Builder」，跟主線的 `Latte`／`LatteBuilder` 無關

**下一天會被動到的地方**：無，Day 12（Singleton）換一個全新情境（咖啡機硬體連線），不會再動到 `Latte`／`LatteBuilder`

---

## Day 12｜Singleton

**故事狀態**：客滿時系統每次點餐都重新連線咖啡機，卡 5 秒，甚至撞到還沒釋放的舊連線噴「設備已被佔用」——因為 `MachineDriver` 每次點餐都被 `new` 一次，明明現實裡只有一台咖啡機

**這天新增/變更的類別**
- `MachineDriver`（新增，同一天內從壞版本改到好版本）：壞版本是普通 public 建構子，每次 `new` 都重新連線；最終版本改成 `sealed class`，`private static readonly Lazy<MachineDriver> _instance`、`public static MachineDriver Instance`、建構子改 `private`，業務方法 `Brew(string drinkName): void`
- `OrderCounter`（新增）：⚠️ `TakeOrder(string drinkName): void`，內部呼叫 `MachineDriver.Instance.Brew(...)`——這是柴咖啡系統目前第 5 個「前台接單」類的類別（`OrderService` Day3／`CheckoutService` Day5／`Cashier` Day9／`ComboService` Day10／`OrderCounter` Day12），彼此職責有點重疊但目前故事沒交代誰取代誰、誰跟誰並存，回頭潤稿可以考慮收斂或至少註明分工

**下一天會被動到的地方**：無，Day 13（Prototype）換一個全新情境（複製客製化訂單），不會再動到 `MachineDriver`／`OrderCounter`

---

## Day 13｜Prototype

**故事狀態**：科技公司團購 20 杯「招牌柴拿鐵」，每杯都要重新 `new` 再一行一行複製貼上將近十個繁瑣欄位（豆子產區、烘焙度、萃取秒數…），改用 Prototype 複製範本、只調整客製化欄位

**這天新增/變更的類別**
- `ChaiLatte`（新增，同一天內從淺拷貝版本改到深拷貝版本）：跟 Day 11 的 `Latte` 是不同概念，不衝突——`BeanOrigin`／`RoastLevel`／`BrewTemp`／`MilkRatio`／`FoamDensity`（固定基底參數）、`Sugar`／`Ice`／`MilkType`（客製化選項，這裡是 `string`）、`Toppings: List<string>`；先用 `MemberwiseClone()` 淺拷貝踩到「小明加肉桂粉，全部人都被加」的坑，最後改成手動對 `Toppings` 做深拷貝

⚠️ **這天文章內部有一處文字跟程式碼對不上**：「Prototype 的取捨」段落寫「`Latte` 的 `Size`、`IceLevel`、`SugarLevel`、`MilkType` 都是 enum（值型別），淺拷貝直接複製就沒問題」——但這天的程式碼類別叫 `ChaiLatte`，欄位也是 `Sugar`／`Ice`／`MilkType`（沒有 `Size`／`IceLevel` 這兩個欄位），而且都是 `string` 不是 enum。這段文字看起來是從 Day 11 的 `Latte` 複製過來、忘記改成 `ChaiLatte` 對應的欄位名稱，建議回頭把這段改成「`ChaiLatte` 的 `BeanOrigin`、`RoastLevel`、`Sugar`、`Ice`、`MilkType` 都是字串（不可變），淺拷貝就沒問題」之類的正確敘述

**下一天會被動到的地方**：無，Day 14 是創建型總結對比，純整理，不會新增/變更程式碼

---

## Day 14｜創建型總結對比

**故事狀態**：純整理，回顧 Factory Method／Abstract Factory／Builder／Singleton／Prototype 五個 Pattern 怎麼分工，附對照表跟決策流程，沒有新劇情

**這天新增/變更的類別**：無，全部引用前五天已經定案的類別（`LatteStore`、`IComboFactory`、`LatteBuilder`、`MachineDriver`、`ChaiLatte` 等），沒有新增或修改任何簽章

**下一天會被動到的地方**：無，Day 15 進入結構型 Pattern，換全新情境（付款流程），創建型這批類別（`DrinkStore` 家族、`IComboFactory`、`LatteBuilder`、`MachineDriver`、`ChaiLatte`）之後應該不會再被動到

---

## Day 15｜Decorator

**故事狀態**：外送平台上架審核要求「能開電子發票」，小黑同時想要「扣款成功後自動同步雲端報表」，阿柴用 Decorator 把這兩個收尾動作各自包成一層，不去動原本的 `PaymentService.Charge()` 邏輯

**這天新增/變更的類別**
- `PaymentResult`（新增）：`Success: bool`、`TransactionId: string`，後來加上 `InvoiceNumber: string`（開發票後才有值）
- `IPaymentProcessor` interface（新增）：`Charge(string orderId, int amount): PaymentResult`
- `PaymentService`（新增）：實作 `IPaymentProcessor`
- `PaymentDecorator` 抽象類別（新增）：實作 `IPaymentProcessor`，內部持有一個 `IPaymentProcessor`，預設把呼叫轉給它
- `InvoiceDecorator`、`SalesReportDecorator`（新增）：皆繼承 `PaymentDecorator`
- `CheckoutController`（新增）：⚠️ `(IPaymentProcessor paymentProcessor)`、`Checkout(string orderId, int amount): void`——這是這個名字第一次出現，之後 Day 17、Day 18 都會用同一個名字定義出完全不同的類別（依賴不同、方法不同），三次沒有互相銜接，回頭潤稿時要決定怎麼收斂

**下一天會被動到的地方**：`CheckoutController` 會在 Day 17、Day 18 分別被重新定義；`Order`／`OrderItem` 會在 Day 16 第一次被完整定義出來（跟 Day 3-4 隱含的形狀不一樣）

---

## Day 16｜Adapter

**故事狀態**：Qber Eats、foodDog 兩家外送平台傳來的訂單資料格式完全不同（一個用數字代碼、一個用英文字串表示狀態），阿柴一開始在 `OrderController` 裡用 `if-else` 各自解析塞進內部 `Order`，格式轉換邏輯跟業務邏輯全部黏在一起，改用 Adapter 讓每個平台各自有一個轉接器，只做格式轉換

**這天新增/變更的類別**
- 🔴 `Order`：**這天第一次給出完整定義**：`OrderId: string`、`Status: OrderStatus`、`TotalPrice: int`、`Items: List<OrderItem>`——跟 Day 3-4 隱含使用的 `Order`（`Items`、`Date`）對不上，**沒有 `Date` 欄位**，多了 `OrderId`／`Status`／`TotalPrice`
- 🔴 `OrderItem`：**這天第一次給出完整定義**：`DrinkName: string`、`Quantity: int`——跟 Day 4 隱含使用的 `OrderItem`（`Price`、`Quantity`、`DrinkType` enum、`OatMilkAddOn`）對不上，**沒有 `Price`／`DrinkType`／`OatMilkAddOn`**，用字串 `DrinkName` 取代 enum，這是柴咖啡系統第三種代表「一杯飲料」的方式（Day 4 的 `DrinkType` enum、Day 9 的 `IDrink` 多型類別、這天的純字串 `DrinkName`）
- `OrderStatus` enum（新增）：`Pending`、`Preparing`、`Completed`
- `KitchenService`（新增）：`ReceiveOrder(Order order): void`
- `OrderController`（新增，同一天內從壞版本改到好版本）：`HandleIncomingOrder(string platform, object payload): void`，壞版本內部塞 `if-else` 判斷平台字串；好版本改成呼叫對應的 `IOrderAdapter`
- `IOrderAdapter` interface（新增）：`ToOrder(): Order`
- `QberEatsOrderAdapter`、`FoodDogOrderAdapter`（新增）：皆實作 `IOrderAdapter`
- `QberEatsOrderPayload`、`QberEatsProduct`、`FoodDogOrderPayload`、`FoodDogItem`（新增）：外部平台原始格式，不是柴咖啡內部模型，不用跟其他天的類別對照

**下一天會被動到的地方**：`CheckoutController` 會在 Day 17（Facade）被重新定義

---

## Day 17｜Facade

**故事狀態**：結帳要依序做「算金額、扣款（順便處理發票、報表）、通知老闆」三件事，這三件事已經各自有專門類別在處理（`OrderCalculator`、`IPaymentProcessor`、`LineNotifier`），但門市 Controller 如果直接把三個子系統串起來，「結帳到底該按什麼順序做哪些事」就變成 Controller 自己得記住的細節，阿柴把這段協調邏輯收進一個新的 `CheckoutFacade`

**這天新增/變更的類別**
- `CheckoutFacade`（新增）：建構子 `(OrderCalculator calculator, IPaymentProcessor paymentProcessor, LineNotifier notifier)`，`Checkout(Order order): Task<PaymentResult>`，內部依序呼叫 `_calculator.Calculate(order)`、`_paymentProcessor.Charge(order.OrderId, (int)total)`、`await _notifier.Notify(total)`
  - ⚠️ `_paymentProcessor.Charge(order.OrderId, ...)` 有正確沿用 Day 16 的 `Order.OrderId` 欄位，也符合 Day 15 `IPaymentProcessor.Charge(string orderId, int amount)` 的簽章，這部分接得起來
  - ⚠️ 但 `_calculator.Calculate(order)` 這條線接不起來：`OrderCalculator`（Day 4 版本）內部要靠 `OrderItem.Price`／`OrderItem.DrinkType` 找對應的 `IDrinkPricing`，可是 Day 16 重新定義的 `OrderItem` 已經沒有這兩個欄位了（只剩 `DrinkName`、`Quantity`）
  - ⚠️ 建構子依賴的是**具體類別** `LineNotifier`，不是 Day 7 定義的 `INotifier` 介面，等於繞過了 Day 7 教的 DIP；而且 `await _notifier.Notify(total)` 把 `Notify` 當成回傳 `Task` 在用，跟 Day 7 把 `LineNotifier.Notify()` 改成 `void` 的版本又對不上
- `CheckoutController`（🔴 第 2 次重新定義）：建構子改成 `(CheckoutFacade checkoutFacade)`，`Checkout(Order order): Task<PaymentResult>` 直接委派給 `_checkoutFacade.Checkout(order)`——跟 Day 15 版本（依賴 `IPaymentProcessor`）完全不同

**下一天會被動到的地方**：`CheckoutController` 會在 Day 18（Proxy）第三次被重新定義

---

## Day 18｜Proxy

**故事狀態**：小黑想讓 Qber Eats 會員直接在門市用外送平台累積的點數換折扣，阿柴串了 Qber Eats 開放的會員查詢 API，上線後發現兩個問題：中午人潮多時常被 API 流量限制擋掉、任何人報一組手機號碼都查得到別人的點數（沒做身份驗證）。用 Proxy 在 `IMemberPointsService` 前面加一層，順便做快取跟權限檢查

**這天新增/變更的類別**
- `IMemberPointsService` interface（新增）：`GetPoints(string phoneNumber): int`
- `QberEatsMemberInfo`（新增）：`Name`、`Points`，模擬 Qber Eats API 回傳格式
- `QberEatsMemberService`（新增）：實作 `IMemberPointsService`，真的打外部 API
- `CheckoutController`（🔴 第 3 次重新定義）：建構子 `(IMemberPointsService memberPointsService)`，新方法 `ApplyMemberDiscount(string phoneNumber): void`——這次連 `Checkout(...)` 方法本身都不見了，只剩會員折扣查詢功能，跟 Day 15／Day 17 的 `CheckoutController` 完全是不同的職責
- `IBindingVerifier` interface（新增）：`IsVerified(string phoneNumber): bool`
- `MemberPointsProxy`（新增）：實作 `IMemberPointsService`，建構子 `(IMemberPointsService realService, IBindingVerifier bindingVerifier)`，內部做保護代理（驗證綁定）+ 快取代理（`Dictionary<string,int>` 記重複查詢結果）
- `StoreBindingVerifier`：文中只提到「查店內綁定資料表的實作」這個角色，沒有給出完整程式碼

⚠️ **這天文章內部有一處文字跟程式碼互相矛盾**：文中先說「`CheckoutController` 不用改，建構子拿到的是 `IMemberPointsService`，只是實際注入的物件換成了 Proxy `MemberPointsProxy`」，但緊接著的程式碼 diff 卻是把建構子參數型別從 `IMemberPointsService`（介面）直接改成 `MemberPointsProxy`（具體類別）：

```csharp
public CheckoutController(IMemberPointsService memberPointsService) { ... }
變成
public CheckoutController(MemberPointsProxy memberPointsService) { ... }
```

這剛好跟前一句「不用改」以及 Proxy 這個 Pattern 的賣點（呼叫端依賴介面，替換實作不用動呼叫端程式碼——也是 Day 7 DIP 教過的事）互相矛盾。建議回頭把第二個版本的參數型別改回 `IMemberPointsService`，才符合文字敘述跟 Proxy 的精神

**下一天會被動到的地方**：無，Day 19（Composite）換一個全新情境（菜單分類），不會再動到 `CheckoutController`／`MemberPointsProxy` 這組

---

## Day 19｜Composite

**故事狀態**：小黑想在電子看板上把菜單依「大分類 > 子分類」顯示（飲料>熱的/冰的、點心>鹹的/甜的），阿柴一開始把分類塞進 `MenuItem.Category` 一個字串欄位（`"飲料-熱"`），用 `Split('-')` 加兩層 `foreach` 分組印出來；小黑後來想再加第三層（甜的下面再分蛋糕類/餅乾類），`Category` 變成有時兩段、有時三段，兩層寫死的 `foreach` 罩不住這種深度不固定的情況，改用 Composite 把「品項」跟「分類」都當同一種節點看待

**這天新增/變更的類別**
- `MenuItem`（新增，柴咖啡系統第一次用這個名字，不跟之前任何類別衝突）：同一天內從壞版本（`Name`/`Price`/`Category: string` 公開屬性）演進到最終版本（實作 `IMenuComponent`，改成建構子注入的私有欄位 `_name`/`_price`，`Category` 欄位整個拿掉）
- `MenuPrinter`（新增，示範用）：`PrintMenu(List<MenuItem> items): void`，**同一天結尾就被棄用**，被下面的 Composite 版本取代，不要在之後的天數延用這個名字
- `IMenuComponent` interface（新增）：`Display(int depth): void`
- `MenuCategory`（新增）：實作 `IMenuComponent`，`_children: List<IMenuComponent>`，`Add(IMenuComponent child)`、`Display(int depth)`（遞迴呼叫每個子節點的 `Display`）

**下一天會被動到的地方**：無，Day 20（Bridge）換一個全新情境（報表種類 vs 匯出格式），不會再動到 `MenuItem`／`MenuCategory`／`IMenuComponent`

---

## Day 20｜Bridge

**故事狀態**：柴咖啡展店後，小黑要阿柴生出各種報表（營業額、庫存、會員積分），不同人要看的格式又不一樣（PDF、Excel、後來會計又要 CSV），阿柴一開始用單一繼承階層同時表達「報表種類」跟「匯出格式」兩個維度，類別數量隨兩者相乘成長；改用 Bridge 把「匯出格式」拆成獨立階層，`Report` 改成持有一個 `IExportFormat` 參考

**這天新增/變更的類別**
- `Report` 抽象類別（新增，同一天內從壞版本改到好版本）：壞版本是無參數 `Generate()`，每種「報表 × 格式」組合各自繼承出一個類別；最終版本改成 `protected IExportFormat _exportFormat`（建構子注入），`Generate()` 內部委派給它——這是柴咖啡系統第一次用 `Report` 這個名字，不跟前面任何一天衝突
- `SalesReport`、`InventoryReport`、`MemberPointsReport`（新增）：皆繼承最終版 `Report`，各自的 `Generate()` 呼叫 `_exportFormat.Export(...)` 帶入不同的報表內容字串
- `IExportFormat` interface（新增）：`Export(string content): void`
- `PdfExport`、`ExcelExport`、`CsvExport`（新增）：皆實作 `IExportFormat`

**非柴咖啡系統類別（生活化類比，交通工具）**：同一天內用了兩次 `Vehicle` 這個名字——先示範壞版本（`Vehicle`／`FuelCar`／`ElectricCar`… 繼承階層爆炸），再重新定義成最終版本（`Vehicle` 變成 Abstraction，持有 `IPowerSource`；`Car`／`Motorcycle` 是 RefinedAbstraction）。這組跟 Day 5（LSP）的 `Vehicle`／`Toyota`／`Honda`／`Tesla` 是兩個完全獨立的示範情境，剛好都借了「車輛」當生活化例子，彼此不影響，不算衝突

**下一天會被動到的地方**：無，Day 21（Flyweight）換一個全新情境（大量重複資料的記憶體共享），不會再動到 `Report`／`IExportFormat`／`Vehicle` 這組

---

## Day 21｜Flyweight

**故事狀態**：柴咖啡連鎖化之後，訂單量早就不是一間店的規模，阿柴一開始讓每一筆訂單的每一行都各自持有一份完整的飲品資料（名稱、基礎價格、圖示網址），這些內容不管哪張訂單、只要品項相同就完全一樣，卻被重複建立了幾千次；阿柴發現這不是「物件怎麼組織」的問題，而是「物件數量太多、其中一大部分內容都長得一模一樣」，改用 Flyweight 把「不會因訂單而改變」的部分抽出來共享

**這天新增/變更的類別**
- `DrinkSpec`（新增，柴咖啡系統第一次用這個名字）：Flyweight，只放內在狀態 `Name`／`BasePrice`／`IconUrl`，皆為建構子注入的唯讀屬性
- `DrinkSpecFactory`（新增）：FlyweightFactory，內部 `Dictionary<string, DrinkSpec>` 當共享池，`GetSpec(string name, int basePrice, string iconUrl): DrinkSpec` 依名稱查快取，查無才建立並存入
- `OrderLine`（新增，同一天內從壞版本演進到最終版本）：壞版本是 `DrinkName`/`BasePrice`/`IconUrl`/`Sugar`/`Ice`/`Quantity` 六個欄位各自持有一份完整資料；最終版本改成 `Spec: DrinkSpec`（共享的內在狀態）＋`Sugar`/`Ice`/`Quantity`（外在狀態）——⚠️ 刻意跟 Day 16 定案的 `OrderItem`（`DrinkName`/`Quantity`）取不同名字，避免跟已經定案的訂單品項模型混淆或衝突，兩者目前並存、互不影響

**非柴咖啡系統類別（生活化類比，遊戲森林場景）**：`TreeType`（Flyweight/ConcreteFlyweight，`Name`/`MeshData`/`TextureData`＋`Draw(int x, int y)`）、`TreeTypeFactory`（FlyweightFactory）、`Tree`（Context，`_x`/`_y` 外在狀態＋共享的 `TreeType` 參考）——跟柴咖啡系統無關，也不跟前面任何一天的類別撞名

**下一天會被動到的地方**：無，Day 22（Strategy）進入行為型 Pattern，換一個全新情境（促銷演算法），不會再動到 `DrinkSpec`／`DrinkSpecFactory`／`OrderLine` 這組

---

## Day 22｜Strategy

**故事狀態**：柴咖啡展店到第二家分店（台北打折扣戰、台中買一送一），阿柴一開始在 `StoreCheckoutService` 裡用 `storeCode` 判斷分店、if-else 各自算一次；後來開第三家店（高雄，主打集點折抵），阿柴得回頭補一個 `else if`，連方法簽章都要多加一個只有高雄店會用到的 `usedPoints` 參數；小黑又想抬高台中店業績，推出上午買一送一、下午改打折扣戰（台北維持原本的折扣戰不變），這支方法完全沒辦法在執行期動態切換促銷算法，才改用 Strategy 把三種算法拆成各自獨立、可替換的策略類別

**這天新增/變更的類別**
- `IPromotionStrategy`（新增）：`Apply(Order order): int`，直接沿用 Day 16 定案的 `Order.TotalPrice`／`Order.Items[].Quantity`，不依賴 `OrderItem.Price`（目前沒有這個欄位）；刻意跟 Day 5/6（LSP/ISP）的 `IDiscount` 命名錯開，兩組介面彼此無關、不算衝突，Day22 也沒有呼應 Day5/6 的舊故事
- `PercentageOffStrategy`、`BuyOneGetOneStrategy`、`PointsRedemptionStrategy`（新增）：皆實作 `IPromotionStrategy`
- `PromotionContext`（新增）：建構子注入 `IPromotionStrategy`，`SetStrategy(IPromotionStrategy strategy): void`（執行期可替換）、`CalculateFinalPrice(Order order): int`
- `StoreCheckoutService`（新增，示範用）：同一天內從壞版本（`CalculateFinalPrice(Order order, string storeCode): int`，只有台北/台中兩個分支）演進到再壞版本（新增高雄店分支後簽章變成 `CalculateFinalPrice(Order order, string storeCode, int usedPoints): int`），**同一天結尾就被棄用**，被 Strategy 版本取代，不要在之後的天數延用這個名字；這是柴咖啡系統第 6 個「結帳/前台」類的類別，但這次是總部端管理跨分店促銷計算，跟前面幾個門市端類別（`OrderService`/`CheckoutService`/`Cashier`/`ComboService`/`OrderCounter`）職責不同

**非柴咖啡系統類別（生活化類比，導航 App 選路線）**：`IRouteStrategy`、`DrivingStrategy`/`TransitStrategy`/`WalkingStrategy`（ConcreteStrategy）、`NavigatorContext`（Context）——跟柴咖啡系統無關，也不跟前面任何一天的類別撞名

**下一天會被動到的地方**：無，Day 23（Observer）換一個全新情境（庫存不足通知、訂單完成通知），不會再動到 `IPromotionStrategy`／`PromotionContext` 這組

---

## Day 23｜Observer

**故事狀態**：柴咖啡展店到三間分店規模，庫存不足這件事牽動的人越來越多——阿柴一開始在 `InventoryService.CheckStock()` 裡寫死依序呼叫「通知進貨窗口」，小黑陸續要求加上「推播通知店長」「同步總部庫存告急儀表板」，阿柴每次都得回頭改同一支方法；接著發現三家分店真正想要的通知組合又不一樣（台北多通知鮮乳廠商、高雄要把 LINE 通知換成簡訊），`CheckStock` 開始塞滿判斷分店代碼的 if-else，改用 Observer 把「收到通知後要做什麼」抽成獨立的觀察者，分店在啟動時自由訂閱要哪幾個，不用再改 `InventoryService` 本身

**這天新增/變更的類別**
- `IStockObserver` interface（新增）：`OnStockLow(string itemName, int quantity): void`
- `ProcurementNotifier`、`StoreManagerNotifier`、`HeadquartersDashboardNotifier`、`SmsProcurementNotifier`（新增）：皆實作 `IStockObserver`
- `InventoryService`（新增，同一天內從壞版本演進到最終版本）：壞版本是 `CheckStock(itemName, quantity)` 內部寫死依序呼叫三個 private 通知方法；最終版本改成 `_observers: List<IStockObserver>`，`Subscribe`/`Unsubscribe`，`CheckStock()` 只負責偵測庫存並通知已訂閱的觀察者
- `IOrderObserver` interface（新增）：`OnOrderCompleted(Order order): void`，沿用 Day 16 定案的 `Order`，作為「同一套 Observer 思路套用在另一個 Subject」的補充範例，文中沒有寫出對應 `OrderService` 訂閱端的完整程式碼
- `CustomerNotifier`、`DeliveryPlatformSyncer`（新增）：皆實作 `IOrderObserver`

**非柴咖啡系統類別（生活化類比，氣象局豪雨特報）**：`IWeatherObserver`、`TrainOperator`/`School`/`ConvenienceStore`（ConcreteObserver）、`WeatherBureau`（Subject，`Subscribe`/`Unsubscribe`/`IssueHeavyRainAlert`）——跟柴咖啡系統無關，也不跟前面任何一天的類別撞名

**下一天會被動到的地方**：無，Day 24（Command）換一個全新情境（點餐指令化，支援取消/重做），不會再動到 `IStockObserver`／`InventoryService`／`IOrderObserver` 這組

---

## 待辦：後續要補的天數

- [ ] Day 24 之後陸續補入
