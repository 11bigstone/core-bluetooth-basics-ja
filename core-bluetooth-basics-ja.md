# AppleのCore Bluetoothの究極ガイド

このドキュメントは以下の記事の翻訳です。  
[The Ultimate Guide to Apple’s Core Bluetooth By Gretchen Walker](
https://punchthrough.com/core-bluetooth-basics/)

---
この記事は、Bluetooth Low Energy（BLE）とiOSのプログラミングの基本（多くのiOSネイティブAPIに共通する非同期呼び出しのデリゲーションパターンを含む）を知っていることを前提としており、iOSのCore Bluetoothライブラリの内部と外部の包括的なガイドとなっています。 BLE周辺機器をスキャンし、接続し、操作するための基本的な手順に加え、よくある落とし穴やiOSのBLEについて知っておくべきことなど、APIの主要コンポーネントを順に説明します。

---
## **アプリのアクセス許可**
アプリがBluetoothを使用できるように、特定のアクセス許可を構成する必要があります。
Bluetoothの使用状況に応じて、アプリの Info.plist に 2つの異なるキーを含める必要があります。

- キー : `Privacy - Bluetooth Always Usage Description`
- 値 : `User-facing description of why your app uses Bluetooth.`  
※iOS 13以降を対象としたアプリに必要です。

このキーは、アプリの初回起動時にユーザーに提示され、アプリがBluetoothにアクセスすることを許可するよう促します。  
例えば、"This app uses Bluetooth to find and maintain connections to your [proprietary device]"（このアプリはBluetoothを使用して、あなたの[proprietary device]への接続を見つけて維持します）のように、明確かつ正直に記述します。  
このキーを無効にすると、iOS 13以降を実行している場合、アプリの起動時にクラッシュし、アプリはApp Storeから拒否されます。

- キー : `Privacy – Bluetooth Peripheral Usage Description`
- 値 : `User-facing description of why your app uses Bluetooth.`  
※Bluetoothを使用するアプリケーションで、最低でもiOS 12以前を対象とする場合に必要です。

上記と同じルールです。iOS 12 以前のデバイスでは、このキーを探してユーザーにメッセージを表示しますが、iOS 13 以降のデバイスでは、上記の最初のキーを使用します。  
iOS 12以前をターゲットにしたアプリは、Info.plistに両方のキーを記述する必要があります。

- キー : `Required background modes`
- 値 : `Array that includes item “App communicates using Core Bluetooth”`  
※スキャンや接続の維持など、バックグラウンドでBluetoothを使用するすべてのアプリケーションに必要です。

---
## **セントラルマネージャー(CBCentralManager)の初期化**
セントラルマネージャーは、Bluetooth接続を確立するために最初にインスタンス化する必要があるオブジェクトです。このオブジェクトは、デバイスのBluetooth状態の監視、Bluetoothペリフェラルのスキャン、およびペリフェラルとの接続と切断を行います。

CBCentralManagerを初期化する際には、CBCentralManagerDelegateプロトコルの非同期メソッドコールを受け取るためのデリゲートを含める必要があります。また、セントラルマネージャーのアクティビティがスケジュールされるべきキューを指定することもできます。実際には、Bluetoothアクティビティ用に別のキューを指定するのがベストですが、それはこの記事の範囲外なので、コード例ではキューのデフォルトをmainにしておきます。

```Swift
class BluetoothViewController: UIViewController {
    private var centralManager: CBCentralManager!

    override func viewDidLoad() {
        super.viewDidLoad()
        centralManager = CBCentralManager(delegate: self, queue: nil)
    }
}
```

このコードの例では、簡単にするために、デリゲートはselfに設定されています。つまり、セントラルマネージャーオブジェクトを格納しているのと同じクラスです。

### **セントラルマネージャーの状態を監視する**
CBCentralManager オブジェクトを単にインスタンス化するだけでは、使用を開始するには十分ではありません。  
実際、初期化コードの直後に `scanForPeripherals(withServices:options:)` を呼ぼうとすると、Xcode のデバッガに警告が表示されます。  
CBCentralManagerDelegateは、`centralManagerDidUpdateState()`メソッドを実装する必要があり、そこからフローを進めることができます。

`centralManagerDidUpdateState()`メソッドは、電話機やアプリ内のBluetoothの状態が更新されるたびにCore Bluetoothから呼び出されます。通常であれば、セントラルマネージャーを初期化した直後に、delegateオブジェクトへのdidUpdateState()コールを受け取り、含まれている状態は.poweredOnになっているはずです。

iOS 10の時点で、使用可能な状態は以下の通りです。

- `poweredOn` : Bluetoothが有効で、認証されており、アプリの使用が可能です。
- `poweredOff` : ユーザーがBluetoothをオフにしている場合は、「設定」またはコントロールセンターからBluetoothをオンにする必要があります。
- `resetting` : Bluetoothサービスとの接続が中断されました。
- `unauthorized` : ユーザーがアプリにBluetoothの使用を拒否しています。ユーザーはアプリの設定メニューから再度許可する必要があります。
- `unsupported` : iOSデバイスはBluetoothに対応していません。
- `unknown` : マネージャーとアプリのBluetoothサービスへの接続の状態が不明です。

iOS には、アプリが Bluetooth を必要とすることをユーザーに通知し、アクセスを要求するプロンプトが組み込まれています。  
iOS のシステムレベルのプロンプトや許可設定のほとんどがそうであるように、アプリは基本的に制御できません。ユーザーがアプリのBluetoothアクセスを拒否すると、CBStateが「`.unauthorized`」となります。このとき、設定のアプリのページでBluetoothの許可を有効にするよう、ユーザーに何らかの指示を出すことができます。  
アプリの設定ページを直接開くためのディープリンクを提供することもできます。

Bluetoothを無効にしているユーザーへの指示も同様です。残念ながら、この記事を書いている時点では、Bluetoothのように「設定」のアプリ以外のページにディープリンクするための、Appleが承認したAPIはもはや存在しません。

---
**注意**  
Appleのプロンプトの動作、UI、メッセージは、iOSのバージョン間で一貫していない可能性があることを想定し、独自のディレクティブの中でそれらを具体的に参照したり、その動作を予測しようとしたりすることは避けてください。

---
```Swift
extension BluetoothViewController: CBCentralManagerDelegate {
 
    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        switch central.state {
            case .poweredOn:
                startScan()
            case .poweredOff:
                // ユーザーにBluetoothをオンにするよう通知
            case .resetting:
                // 次の状態の更新を待ち、Bluetoothサービスの中断を検討する
            case .unauthorized:
                // アプリの「設定」でBluetooth許可を有効にするようユーザーに通知
            case .unsupported:
                // デバイスがBluetoothをサポートしていないことをユーザーに警告し、且つアプリが期待通りに動作しない
            case .unknown:
               // 次の状態の更新を待つ
        }
    }
}
```
---

## **周辺機器(ペリフェラル)のスキャン**
`didUpdateState()`の呼び出しと`.poweredOn`フラグを受け取ったら、スキャンの開始に進むことができます。  
セントラルマネージャーオブジェクトで `scanForPeripherals(withServices:options:)` を呼び出します。フィルタリングしたい特定のサービスを表すCBUUID(上記の定義を参照)の配列を渡すことができます。  
セントラルマネージャーは、`centralManager(_:didDiscover:advertisementData:rssi:)`デリゲートメソッドを介して、これらのサービスの1つ以上を持っていることをアドバタイズ*しているデバイスのみを返します。  
このメソッドには、0個以上の以下のキーを含むディクショナリーを使って、いくつかのオプションを渡すことができます。

- `CBCentralManagerScanOptionAllowDuplicatesKey`

このキーはBoolを値として受け取り、trueの場合、スキャンセッションで最初に受信したアドバタイジングパケットだけではなく、指定されたデバイスの検出されたすべてのアドバタイジングパケットに対してデリゲートコールを行います。  
デフォルト値（この記事が書かれた時点）はfalseで、Appleはこの設定を可能な限りオフにしておくことを推奨しています。なぜなら、この設定は代替手段よりもはるかに少ない電力とメモリしか使用しないからです。  
しかし、スキャンセッション中に更新されたRSSI値を受信するために、この設定をオンにする必要があることがよくあります。

- `CBCentralManagerScanOptionSolicitedServiceUUIDKey`

一般的には使用されていませんが、GAP周辺機器がGAPセンターにアドバタイズする場合、センター機器がクライアントではなくGATTサーバーとして機能する場合に便利です（通常はその逆です）。周辺機器は、中央装置のGATTテーブルに表示されることを期待して、特定のサービスをアドバタイジング（勧誘）することができます。  
CBCentralManagerScanOptionSolicitedServiceUUIDKeyを使用して、周辺機器のサービスをスキャンすることができます。

---
**注意**  
周辺機器は、通常、含まれるサービスのすべて、あるいはほとんどを宣伝するものではありません。
その代わり、デバイスは通常、特定のタイプのBLEデバイスにしか興味がない場合に、セントラルが探すべき特定のカスタムサービスを宣伝します。

---

### **周辺機器(ペリフェラル)の識別子**
Androidとは異なり、iOSではセキュリティの観点から周辺機器のMACアドレスがアプリ開発者から見えません。  
周辺機器には、CBPeripheral オブジェクトの identifier プロパティにあるランダムに生成された UUID が割り当てられます。このUUIDは、スキャンセッション間で同じであることが保証されていないため、周辺機器の再識別に100％依存するべきではありません。  
しかし、私たちは、主要なデバイス設定のリセットが発生していないと仮定した場合、これは比較的安定しており、長期的に信頼できることを確認しています。  
代替手段が用意されていれば、デバイスが見えなくなった時の接続要求などにも頼ることができました。

### **スキャン結果**
デリゲートメソッド`centralManager(_:didDiscover:advertisementData:rssi:)`への各呼び出しは、範囲内にあるBLE周辺機器の検出されたアドバタイズメントパケットを反映しています。上述したように、あるスキャンセッションでのデバイスごとの呼び出し回数は、提供されるスキャンオプションや、周辺機器自体の範囲やアドバタイジングの状態によって異なります。

メソッドのシグネチャーは以下のようになります。
```Swift
optional func centralManager(_ central: CBCentralManager, 
                didDiscover peripheral: CBPeripheral, 
                    advertisementData: [String : Any], 
                            rssi RSSI: NSNumber)
```
上記のメソッドのパラメータを分解してみましょう。

- `central: CBCentralManager`  
スキャン中にデバイスを発見したセントラルマネージャーオブジェクト。

- `peripheral: CBPeripheral`  
発見されたBLEの周辺機器を表すCBPeripheralオブジェクトです。このタイプについては、後のセクションで詳しく説明します。

- `advertisementData: [String: Any]`  
検出されたアドバタイズメントパケットに含まれるデータの辞書的表現。Core Bluetoothは、内蔵されたキーセットを使って、このデータを解析し、整理するという素晴らしい仕事をしてくれます。

|アドバタイズキー名と関連する値のタイプ|
| :--- |
|`CBAdvertisementDataManufacturerDataKey`<br>*NSData*<br>周辺機器メーカーが提供するカスタムデータです。ペリフェラルでは、デバイスのシリアル番号やその他の識別情報の保存など、さまざまな用途に使用できます。|
|`CBAdvertisementDataServiceDataKey`<br>*[CBUUID : NSData]*<br>サービスを表すCBUUIDキーと、そのサービスに関連するカスタムデータを持つ辞書。これは通常、ペリフェラルが接続前に使用するためのカスタム識別データを保存するのに最適な場所です。|
|`CBAdvertisementDataServiceUUIDsKey`<br>*[CBUUID]*<br>サービス UUID の配列で、通常、デバイスの GATT テーブルに含まれる 1 つ以上のサービスを表します。|
|`CBAdvertisementDataOverflowServiceUUIDsKey`<br>*[CBUUID]*<br>アドバタイジングデータのオーバーフロー領域（スキャンレスポンスパケット）のサービスUUIDの配列です。メインのアドバタイジングパケットに収まらなかったアドバタイジングサービス用。|
|`CBAdvertisementDataTxPowerLevelKey`<br>*NSNumber*<br>アドバタイジングパケットに含まれる場合は、周辺機器の送信パワーレベル。|
|`CBAdvertisementDataIsConnectable`<br>*NSNumber*<br>NSNumber形式のブール値（0または1）で、周辺機器が現在接続可能であれば1となります。|
|`CBAdvertisementDataSolicitedServiceUUIDsKey`<br>*[CBUUID]*<br>勧誘されたサービスのUUIDの配列です。CBCentralManagerScanOptionSolicitedServiceUUIDKeyセクションの勧誘サービスの説明を参照してください。|
---
- rssi: NSNumber  
アドバタイズメントパケットを受信した時点での、ペリフェラルの相対的な信号品質をデシベルで表したもの。RSSIは相対的な指標であるため、中央で解釈される値はチップセットによって異なります。ほとんどのiOSデバイスがCore Bluetoothを介して返すRSSIは、一般的に-30～-99の範囲で、-30が最も強い値となります。

```Swift
// メインクラス内
var discoveredPeripherals = [CBPeripheral]()
func startScan() {
    centralManager.scanForPeripherals(withServices: nil, options: nil)
}
…
…
…
// CBCentralManagerDelegate クラス/エクステンション内
func centralManager(_ central: CBCentralManager, didDiscover peripheral: CBPeripheral, advertisementData: [String : Any], rssi RSSI: NSNumber) {
    self.discoveredPeripherals.append(peripheral)
}
```

上の例では、発見されたペリフェラルを単に内部配列に格納していますが、これを行うとデリゲートメソッドコールで返されるRSSIとアドバタイズメントデータが失われます。CBPeripheral はそれ自体ではサポートしていないので、後でアクセスしたい場合は、これらの項目のストレージを含む CBPeripheral オブジェクトのラッパークラスまたは構造体を作成することがしばしば役に立ちます。

---

## **接続・切断**
### **ペリフェラルへの接続**
目的の CBPeripheral オブジェクトへの参照を取得したら、単にセントラルマネージャーの `connect(_:options:)` メソッドを呼び出して、ペリフェラルを渡すことで、それに接続しようとすることができます。いくつかの接続オプションがあり、それについてはここで読むことができますが、この記事では説明しません。

接続が成功した場合は、`centralManager(_:didConnect:)`のデリゲートコールを受け取り、接続が失敗した場合は、`centralManager(_:didFailToConnect:error:)`を受け取り、その中には、周辺機器と発生した特定のエラーの両方が含まれています。

圏外になってしまった特定の周辺オブジェクトに対して`connect`を呼び出すことがあります。そうすると、「接続要求」が確立され、iOSは、接続を行うデバイスを見つけて`didConnect`デリゲートメソッドを呼び出すまで、（Bluetoothが中断されるか、ユーザーによってアプリが手動で終了されない限り）無期限に待機します。

```Swift
// メインクラス内
var connectedPeripheral: CBPeripheral?

func connect(peripheral: CBPeripheral) {
    centralManager.connect(peripheral, options: nil)
 }
…
…
… 
// CBCentralManagerDelegate クラス/エクステンション内
func centralManager(_ central: CBCentralManager, didConnect peripheral: CBPeripheral) {
    // 接続に成功しました。まだ行われていなければ、ペリフェラルへの参照を保存します。
    self.connectedPeripheral = peripheral
}
 
func centralManager(_ central: CBCentralManager, didFailToConnect peripheral: CBPeripheral, error: Error?) {
    // エラーハンドリング
}
```
---
**注意**  
scanコールバックがCBPeripheralオブジェクトを返したら、コードの中でそれへの強い参照を保持しなければなりません。単に didDiscover デリゲートメソッドから connect をすぐに呼び出し、ペリフェラルを別の場所に強く保存せずにその関数ブロックを完了させた場合、ペリフェラルオブジェクトは解放され、あらゆる接続や保留中の接続が解除されます。中央管理者は、接続されたペリフェラルとの強力な接続を内部的に保持しません。

---
**注意**  
iOSでは、didConnectデリゲートメソッドは、基本的な接続が確立された直後、そしてペアリングやボンディングが試みられる前に呼び出されることに注意してください。多くのiOS開発者にとって残念なことに、Core Bluetoothは、暗号化されたサービスやcharacteristicを介して推論したり起動したりすること以外に、周辺機器のボンディングプロセスに関する実際のパブリックAPIレベルの洞察や制御を与えませんが、これについては後述します。

---
**注意**  
Core Bluetooth では、CBPeripheral オブジェクトの状態が `.connected` であるためには、iOS の BLE レベルとアプリレベルの両方で接続されている必要があります。ペリフェラルは、他のアプリを通して、または自動再接続をトリガするHIDのようなプロファイルを含んでいるために、iOSデバイスに接続されているかもしれません。しかし、ペリフェラルとやりとりするためには、ペリフェラルへの参照を取得し、アプリから `connect()` を呼び出す必要があります。詳細については、後述のボンディング/ペアリングの説明を参照してください。

---

### **CBPeripheralsの識別と参照**
AppleがCore Bluetoothで行ったもう一つのセキュリティ上の選択は、（良くも悪くも）AndroidのBluetooth APIとは異なるもので、BLEペリフェラルの固有のMACアドレスを不明瞭にすることでした。カスタムアドバタイズメントデータ、デバイス名、またはcharacteristicなど、周辺機器のファームウェアによって別の場所に隠されていない限り、Core Bluetoothからこのアドレスにアクセスすることはできません。その代わり、Appleは、アプリのコンテキスト外では無意味ですが、特定のデバイスをスキャンして接続を開始するために使用できるユニークなUUIDを割り当てます（「バックグラウンド処理」の項を参照）。Appleは、このUUIDが常に同じであることを保証するものではなく、周辺機器を識別する唯一の方法であってはならないと明示しています。しかし、私たちの経験では、ユーザーがネットワークやその他の工場出荷時の設定をリセットした場合を除いて、Appleが割り当てたUUIDは長期的にはかなり信頼できるものであると考えています。

アドバタイジングの段階で周辺機器を識別するための他のオプションは、名前またはカスタムアドバタイジングサービスデータです。上述したように、アドバタイズデータは、特定のブランドを識別するためのカスタムサービスUUIDを含むことができ、さらに、特定のデバイスまたはデバイスのセットをさらに識別するために、アドバタイジングパケット内のそれらのサービスにリンクされたカスタムデータを含むことができます。

### **ペリフェラルとの接続解除**
接続を解除するには、`cancelPeripheralConnection(_:)`を呼び出すか、ペリフェラルへの強力な参照をすべて削除して`cancel`メソッドを暗黙的に呼び出すだけです。応答として`centralManager(_:didDisconnectPeripheral:error:)`のデリゲートコールを受け取る必要があります。
```Swift
// メインクラス内
func disconnect(peripheral: CBPeripheral) {
    centralManager.cancelPeripheralConnection(peripheral)
}
…
…
…
// ICBCentralManagerDelegate クラス/エクステンション内
func centralManager(_ central: CBCentralManager, didDisconnectPeripheral peripheral: CBPeripheral, error: Error?) {
    if let error = error {
        // Handle error
        return
    }
    // 切断に成功
}
```

### **iOSからの接続解除**
iOSは、周辺機器からの通信がない状態がしばらく続くと、接続を解除することがあります（30秒と言われていますが、動作は保証されていません）。これは通常、周辺機器が何らかのハートビートで対処しますが、iOSアプリ層では見えない場合もあります。

---

## **Services と Characteristics の探索**
ペリフェラルへの接続が成功すると、そのサービスを発見し、次にそのcharacteristicを発見することがあります。  
この時点で、`CBCentralManager` と `CBCentralManagerDelegate` メソッドの使用から、`CBPeripheral` と `CBPeripheralDelegate` メソッドに移行します。この時点で、ペリフェラルオブジェクトのデリゲートプロパティを割り当てて、それらのデリゲートコールバックを受け取れるようにしておきましょう。
```Swift
func centralManager(_ central: CBCentralManager, didConnect peripheral: CBPeripheral) {
    self.connectedPeripheral = peripheral
    peripheral.delegate = self
}
```
(繰り返しになりますが、ここではシンプルにするために、1つのクラスですべてのデリゲートコールを処理していますが、これは大規模なコードベースでは最良の設計方法ではありません。)

### **Services の探索**
CBPeripheral オブジェクトを初めて発見して接続したとき、それが `[CBService]?` タイプの `services` プロパティを持っていることに気づくでしょう。現時点では、それは `nil` です。単に `discoverServices([CBUUID]?)` を呼ぶことで、周辺機器のサービスを発見する必要があります。オプションでサービス UUID の配列を渡すことができ、発見されるサービスを制限することができます。これは、アプリが気にしない多くのサービスを含むペリフェラルの場合に便利で、これらを無視する方がはるかに効率的（特に時間的に）です。

同様のメソッドとして、`discoverIncludedServices([CBUUID]?, for: CBService)`がある。サービスは、他の関連するサービスを "含む "ことを示すかもしれません。これは、周辺機器のファームウェアが、それらが何らかの関連性を持つことを示したいということ以外には、あまり意味がありません。2番目のパラメータには、すでに発見されたサービスを指定し、1番目のパラメータでは、前の段落で説明したように、返されたサービスを任意にフィルタリングすることができます。パンチスルーでは、BLEのこの機能をあまり利用したことがありませんが、大きなGATTテーブルがあり、グループごとにサービスの発見をずらしたい場合や、興味のあるサービスをすべて明示的にリストアップしなくてもよい場合には、便利かもしれません。もちろん、そのためには、周辺機器側が協力して、含まれるサービスを表示する必要があります。

サービスが発見されると、`CBPeripheralDelegate`は `peripheral(_:didDiscoverServices:)`を呼び出し、discoverメソッドに渡されたオプションで提供される配列から発見されたサービスUUIDを表すCBServiceオブジェクトの配列を受け取ります（配列がnil/emptyだった場合は、すべてのサービスが返されます）。また、CBPeripheralのservicesプロパティをアンラップしてみると、これまでに発見されたサービスの配列が含まれていることに気づくでしょう。

### **Characteristics の探索**
characteristicはサービスの下にグループ化されています。ペリフェラルのサービスを発見したら、それらのサービスのそれぞれのcharacteristicを発見することができます。`CBPeripheral`の`services`プロパティと同様に、`CBService`にも`[CBCharacteristic]`型の`characteristics`プロパティがあり、最初は空であることがわかります。

各サービスに対して、CBPeripheralオブジェクトの`discoverCharacteristics([CBUUID?], for: CBService)`を呼び出し、サービスに対して行ったのと同様に、オプションで特定のcharacteristicUUIDを指定します。発見コールを行った各サービスについて、`peripheral(_:didDcaverCharacteristicsFor:error:)`へのコールを受け取る必要があります。

ニーズによっては、発見の時点で興味のあるcharacteristicへの参照を保存しておくと、各サービスのCaracteristicsプロパティに入力された配列を検索する必要がなくなり、便利です。  
これはまた、特定のCBPeripheralDelegateコールバックを受け取るたびに、どのcharacteristicが参照されているかを素早く識別することを容易にします。
```Swift
// メインクラス内
// ペリフェラル接続語に呼び出す
func discoverServices(peripheral: CBPeripheral) {
    peripheral.discoverServices(nil)
}
 
// Services探索後に呼び出す
func discoverCharacteristics(peripheral: CBPeripheral) {
    guard let services = peripheral.services else {
        return
    }
    for service in services {
        peripheral.discoverCharacteristics(nil, for: service)
    }
}
…
…
…
// CBPeripheralDelegate クラス/エクステンション内
func peripheral(_ peripheral: CBPeripheral, didDiscoverServices error: Error?) {
    guard let services = peripheral.services else {
        return
    }
    discoverCharacteristics(peripheral: peripheral)
}
 
func peripheral(_ peripheral: CBPeripheral, didDiscoverCharacteristicsFor service: CBService, error: Error?) {
    guard let characteristics = service.characteristics else {
        return
    }
    // 重要なcharacteristicsを内部に保存し、後で簡単にアクセスして同等性をチェックできるようにすることを検討してください。
    // ここから、必要に応じてcharacteristicsの読み書きや通知の購読を行うことができます。
}
```

### **ディスクリプタ**
Bluetoothでは、characteristicが持つ値についてより多くの情報を提供するために、特定のcharacteristicsと一緒にcharacteristics記述子をオプションで提供することができます。  
これらは、例えば、人間が読める説明文字列や、値を解釈するために意図されたデータフォーマットなどです。Core Bluetoothでは、これらはCBDescriptorオブジェクトで表現されます。CBDescriptorはcharacteristicsと同様の機能を持ち、読み取る前にまず発見されなければなりません。

与えられたcharacteristicに対して、単純にdiscoverDescriptors(for characteristic)を呼び出します。 CBCharacteristic)をターゲットペリフェラルに設定し、非同期コールバックペリフェラル(_ peripheral.CBCharacteristic)を待ちます。 CBPeripheral, didDiscoverDescriptorsFor characteristic: CBCharacteristic, error: エラー？）。) このとき、CBCharacteristicオブジェクトのdescriptorsプロパティは非nilで、CBDescriptorオブジェクトの配列を格納する必要があります。

CBDescriptorsのさまざまなタイプは、uuidプロパティ（CBUUIDタイプ）によって区別され、これらはCore Bluetoothのドキュメントであらかじめ定義されています。

CBDescriptorのvalueプロパティは、readValue(for descriptor)のいずれかを使用して、明示的に読み書きされるまでnilになります。 CBDescriptor）や、writeValue（_ data: データ、デスクリプタ用。 CBPeripheralのCBDescriptor）メソッドです。

これらはコールバックメソッドで対応します。  
`peripheral(_ peripheral: CBPeripheral, didUpdateValueFor descriptor: CBDescriptor, error: Error?)`  
と  
`peripheral(_ peripheral: CBPeripheral, didWriteValueFor descriptor: CBDescriptor, error: Error?)`

```Swift
// メインクラス内
func discoverDescriptors(peripheral: CBPeripheral, characteristic: CBCharacteristic) {
    peripheral.discoverDescriptors(for: characteristic)
}
…
…
… 
// CBPeripheralDelegate クラス/エクステンション内
func peripheral(_ peripheral: CBPeripheral, didDiscoverDescriptorsFor characteristic: CBCharacteristic, error: Error?) {
    guard let descriptors = characteristic.descriptors else { return }
 
    // ユーザーの説明descriptorを取得
    if let userDescriptionDescriptor = descriptors.first(where: {
        return $0.uuid.uuidString == CBUUIDCharacteristicUserDescriptionString
    }) {
        // characteristicのあるユーザーの説明を読む
        peripheral.readValue(for: userDescriptionDescriptor)
    }
}
 
func peripheral(_ peripheral: CBPeripheral, didUpdateValueFor descriptor: CBDescriptor, error: Error?) {
    // 与えられたcharacteristicに対するユーザーの説明を取得してprintする
    if descriptor.uuid.uuidString == CBUUIDCharacteristicUserDescriptionString,
        let userDescription = descriptor.value as? String {
        print("Characterstic \(descriptor.characteristic.uuid.uuidString) is also known as \(userDescription)")
    }
}
```
---
**注意**  
Core Bluetoothでは、iOSで（AndroidのBLEアプリ開発では明示的に）、そのcharacteristicに関する通知／表示を購読／解除するために水面下で使用されているクライアントcharacteristic構成記述子（CBUUIDClinaterCharacteristicConfigurationString）の値を書き込むことはできません。 代わりに、次のセクションで説明する`setNotifyValue(_:for:)`メソッドを使用します。

---
## **通知や表示を購読する**
characteristicがそれをサポートしている場合（「characteristicのプロパティ」のセクションを参照）、単に`setNotifyValue(_ enabled: Bool, for characteristic: CBCharacteristic)`を呼び出すことによって、通知/表示を購読することができる。  
通知と表示は、BLEスタックレベルでは機能的に異なりますが、Core Bluetoothでは区別されません。 あるcharacteristicにサブスクライブしている場合、そのcharacteristicの値が変化して周辺機器から通知や表示が送られてくるたびに、そのcharacteristicの`peripheral(_:didUpdateValueFor:error:)`への呼び出しが行われます。 配信を停止するには、`setNotifyValue(false, for:characteristic)`を呼び出すだけです。  
この設定を変更するたびに、`peripheral(_ peripheral: CBPeripheral, didUpdateNotificationStateFor characteristic: CBCharacteristic, error: Error?)`への呼び出しが発生します。

characteristicの`isNotifying`プロパティをチェックすることで、いつでも通知購読状態を確認することができます。

```Swift
// メインクラス内
func subscribeToNotifications(peripheral: CBPeripheral, characteristic: CBCharacteristic) {
    peripheral.setNotifyValue(true, for: characteristic)
 }
…
…
…
// CBPeripheralDelegate クラス/エクステンション内
func peripheral(_ peripheral: CBPeripheral, didUpdateNotificationStateFor characteristic: CBCharacteristic, error: Error?) {
    if let error = error {
        // エラーハンドリング
        return
    }
    // characteristicに関する通知／表示の購読／解除に成功した
}
```
---
## **characteristicから読み込む**
characteristicに「read」プロパティが含まれている場合は、CBPeripheralオブジェクトの`readValue(for:CBCharacteristic)`を呼び出すことで、その値を読み取ることができます。  
値はCBPeripheralDelegateの`peripheral(_:didUpdateValueFor:error:)`メソッドを通じて返されます。
```Swift
// メインクラス内
func readValue(characteristic: CBCharacteristic) {
    self.connectedPeripheral?.readValue(for: characteristic)
}
… 
…
…
// CBPeripheralDelegate クラス/エクステンション内
func peripheral(_ peripheral: CBPeripheral, didUpdateValueFor characteristic: CBCharacteristic, error: Error?) {
    if let error = error {
        // エラーハンドリング
        return 
    }
    guard let value = characteristic.value else {
        return
    }
    // データで何かする
}
```
---
## **characteristicに書き込む**
characteristicに書き込むには、CBPeripheralオブジェクトの`writeValue(_ data: Data, for characteristic: CBCharacteristic, type: CBCharacteristicWriteType)`を呼び出します。

書き込みタイプは2種類あります。 `.withResponse`と`.withoutResponse`です。 これらはそれぞれ、BLEでいうところの「書き込み要求」と「書き込みコマンド」に相当します。

書き込み要求（`.withResponse`）の場合、BLE層は、書き込み要求が受信され、正常に完了したことを示す確認応答を周辺機器に送り返すことを要求します。 Core Bluetooth層では、リクエストの完了またはエラー時に、`peripheral(_:didUpdateValueFor:error:)`の呼び出しを受けます。

書き込みコマンド(`.withoutResponse`)の場合、書き込まれた値を受け取っても確認応答は送信されず、iOSの観点から書き込みが成功したとしても、デリゲートコールバックは発生しません。 つまり、iOSは内部的な問題を抱えたまま、正常に書き込み操作を行うことができたのです。

書き込みリクエストは、配信が保証されているか、そうでなければ明示的なエラーが発生するため、より堅牢です。 一般的には、1回限りの書き込み操作に適しています。 しかし、大容量のバルクデータを複数回の書き込み操作で連続して送信する場合、各操作で確認応答を待つと、全体の処理速度が大幅に低下します。 その代わりに、FW-モバイルプロトコルにパケットトラッキングを組み込むことを検討してください。

```Swift
// メインクラス内
func write(value: Data, characteristic: CBCharacteristic) {
    self.connectedPeripheral?.writeValue(value, for: characteristic, type: .withResponse)
    // OR
   self.connectedPeripheral?.writeValue(value, for: characteristic, type: .withoutResponse)
 }
…
…
…
// CBPeripheralDelegate クラス/エクステンション内
// 書き込みタイプが.withResponseの場合のみ呼び出されます。
func peripheral(_ peripheral: CBPeripheral, didWriteValueFor characteristic: CBCharacteristic, error: Error?) {
    if let error = error {
        // エラーハンドリング
        return
    }
    // characteristicに値を書き込むことに成功した
}
```

### **連続した書き込みコマンドの最大化 : どのくらいの速度が速いのか？**
書き込みコマンド（応答なし）の性質上、相手側でのパケット配送を保証することはできませんが、とはいえ、iOS内部の専用キューバッファを圧迫しないよう、適度な速度でレスポンスを送信することを心がけたいものです。  
iOS 11までは、これは単なる推測でしかありませんでしたが、CBPeripheralとCBPeripheralDelegate APIに追加機能を導入して、これを緩和しました。

`CBPeripheral property canSendWriteWithoutResponse: Bool`  
と  
`CBPeripheralDelegate method peripheralIsReady(toSendWriteWithoutResponse: CBPeripheral)`

書き込みコマンドを送信する前には、必ず`canSendWriteWithoutResponse`をチェックし、falseの場合は`peripheralIsReady(toSendWriteWithoutResponse:)`の呼び出しを待ってから実行する必要があります。  
なお、`canSendWriteWithoutResponse`をtrueに設定し、実際にデリゲートメソッドを呼び出した場合、特に復元されたペリフェラルの場合には、信頼性にばらつきがあることが確認されていますので、書き込みコマンドの実行をこのAPIだけに頼るのは避けた方がよいでしょう。  
タイマーで応答のない書き込みを実装することもできますが、データのサイズに応じて保守的にする必要があります。
```Swift
// メインクラス内
func write(value: Data, characteristic: CBCharacteristic) {
    if connectedPeripheral?.canSendWriteWithoutResponse {
        self.connectedPeripheral?.writeValue(value, for: characteristic, type: .withoutResponse)
    }
}
…
…
…
// CBPeripheralDelegate クラス/エクステンション内
func peripheralIsReady(toSendWriteWithoutResponse peripheral: CBPeripheral) {
    // ペリフェラルが再び応答のない書き込みを送信する準備ができたときに呼び出されます。
    // 何らかの対象となるcharacteristicに何らかの値を書き込む。
    write(value: someValue, characteristic: someCharacteristic)
}
```

### **最大書き込みサイズ**
iOS 9の時点で、Core Bluetoothは、特定の特性に対する最大長（バイト）を決定する便利な方法を提供しています。これは、レスポンス付きの書き込みとそうでない書き込みでは異なる場合があります。 なしの場合:  
`maximumWriteValueLength(for type: CBCharacteristicWriteType) -> Int`  
通信がサポートする実際の最大書き込みサイズは、中央デバイスと周辺デバイスのそれぞれのBLEスタックによって異なります。 このメソッドは、単純に、iOSデバイスがその操作でサポートする最大の書き込みサイズを返します。 片側または両側の既知の制限を超える書き込みを試みたときの動作は未定義です。

---
## **ペアリングとボンディング**
iOS 9では、ペアリング（1回限りの安全な接続のための一時的な鍵の交換）は、ボンディング（ペアリング後、将来の接続のための長期的な鍵の追加の安全な交換と保存）なしでは許可されません。

ペリフェラルのBLEセキュリティ設定に応じて、ペアリング／ボンディングプロセスは、接続時、または暗号化された特性の読み取り、書き込み、購読を試みたときにトリガされます。 Appleのアクセサリーデザインガイドラインでは、実際にBLE機器メーカーに対して、ボンディングのトリガーとして不十分な認証方法（暗号化された特性からの読み取り）を使用することを推奨していますが、我々の調査によると、Android機器でも概ね成功しているものの、最も確実に機能する方法はメーカーやモデルによって異なる傾向にあります。

繰り返しますが、ペリフェラルの構成に応じて、ペアリングプロセス中のUIフローは次のいずれかです。
* ユーザーには、PINの入力を促すアラートが表示されます。 ペアリング／ボンディングを行うには、ユーザーが正しいPINを入力する必要があります。
* ユーザーには、ペアリングを継続するための許可を求めるシンプルなYes/Noダイアログが表示されます。
* ユーザーには何のダイアログも表示されず、水面下でペアリング／ボンディングが完了します。

多くのiOS開発者にとって残念なことに、Core Bluetoothは、ペアリングやボンディングのプロセスや、周辺機器のペアリング/ボンディングの状態についての情報を提供しません。 APIは、デバイスが接続されているかどうかを開発者に知らせるだけです。

HIDデバイスのように、周辺機器を一度ボンディングすると、iOSが周辺機器の広告を見るたびに自動的に接続するケースもあります。 この動作はどのアプリとも無関係に発生し、周辺機器がiOSデバイスに接続されていても、最初に接続を確立したアプリには接続されていないことがあります。  
接合されたペリフェラルがiOSデバイスから切断され、その後iOSレベルで再接続された場合、アプリはペリフェラルオブジェクトを取得し`(retrieveConnectedPeripherals(with[Services/Identifiers]:)`、CBCentralManagerを介して明示的に再接続し、アプリレベルの接続を確立する必要があります。  
このメソッドでデバイスを取得するには、先に返されたCBPeripheralオブジェクトからAppleが割り当てたデバイス識別子を指定するか、それに含まれる少なくとも1つのサービスを指定する必要があります。

---
**注意**  
iOSでは、開発者向けアプリが周辺機器の結合状態をキャッシュから消去することはできません。 結合を解除するには、iOSの「設定」の「Bluetooth」の項目で、周辺機器を明示的に「忘却」する必要があります。 ほとんどのユーザーはこのことを知らないので、ユーザーエクスペリエンスに影響を与える場合は、アプリのUIにこの情報を含めるとよいでしょう。

---
## Core Bluetoothのエラー
CBCentralManagerDelegateとCBPeripheralDelegateプロトコルのほぼすべてのメソッドには、`Error?` 型のパラメータで、エラーが発生した場合は非nilとなります。 これらのエラーは、CBErrorまたはCBATTErrorのいずれかのタイプであることが予想されます。 Appleは、どのメソッドがどのようなエラーを返すのか、またどのような型のエラーを返すのかについては明確にしていないため、個々のCore Bluetoothのエラーについて、どのような動作が考えられるのかは不明です。

通常、CBATTErrorsは、ATTレイヤで何か問題が発生したときに返されます。 これには、暗号化された特性へのアクセス問題、サポートされていない操作（例：読み取り専用の特性への書き込み操作）、その他多くのエラーが含まれますが、これらは通常、iOSデバイスをペリフェラルとして設定するためにCBPeripheralManager APIを使用している場合にのみ適用されます（通常、ほとんどのBluetoothデバイスではATTサーバが存在します）。

---
## **あとがき**
BLE開発に精通している方も、これから始める方も、この記事がお役に立ち、情報を得られたことを願っています。 今回は、セントラルロールの観点からCore Bluetoothの基本を説明しましたが、まだまだ説明したいことがたくさんあります。 今後、iOSのCore BluetoothやBLE全般に関する記事をお届けします。また、iOS開発に関する他の記事もご覧ください。

[How to Handle iOS 13’s new Bluetooth Permissions](https://punchthrough.com/how-to-handle-ios-13s-new-bluetooth-permissions/)

[Leveraging Background Bluetooth for a Great User Experience](https://punchthrough.com/leveraging-background-bluetooth-for-a-great-user-experience/)