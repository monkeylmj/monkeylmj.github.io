---
title: "Android NFC开发入门"
date: 2017-02-28 13:08:50 +0800
tags:
 - NFC
---



## NFC的几个概念

* **接触式IC卡** 例如手机SIM卡、金融IC卡。

* **非接触式IC卡** 又称射频卡，将无线射频识别技术和IC卡结合起来，免接触。

* **RFID** 无线射频识别，一种无线通讯技术。

  基本原理：阅读器将电信号转换为无线电信号（电磁波的一个频带）发给标签，标签使用接收到的无线电波能量供电，然后将存储在自身数据以无线电信号的形式应答给阅读器，以读取到标签中的数据。

* **NFC**（Near Field Communication） 短距离无线通讯技术，基于RFID，一般在10cm之内使用13.56MHz频率通讯

<!-- more -->

---




## Android如何使用NFC

###权限声明

在AndroidManifest.xml中如下申请NFC权限:

```xml
<uses-permission android:name="android.permission.NFC" />
```

添加下面一行，保证在GooglePlay商店此应用只显示给带NFC硬件的设备

```xml
<uses-feature android:name="android.hardware.nfc" android:required="true" />
```

在代码中动态判断设备是否支持NFC：

```java
if(NfcAdapter.getDefaultAdapter() == null){
  Log.v("monkey","设备不支持NFC.");
}
```



### Android设备支持三种模式的NFC操作
1. **Reader/writer mode** 通过Android设备对NFC标签进行读写数据操作。
2. **P2P mode** NFC设备之间交换数据，AndroidBeam使用这种模式。
3. **Card emulation mode** NFC设备模拟成NFC标签。

### Reader/write mode

Android通过标签分发系统分析发现的NFC标签，封装成对应的Intent发给Android上层进行对应的处理。

Android中关于NFC有三种类型的Action（优先级由高到低）：

`ACTION_NDEF_DISCOVERED` `ACTION_TECH_DISCOVERED` `ACTION_TAG_DISCOVERED` 

#### 这里的*优先级*可以通过标签分发系统的机制来解释:

1. 如果一个包含**NDEF**格式数据的NFC标签被发现，Android优先发送带`ACTION_NDEF_DISCOVERED`action的Intent。
2. 如果没有Activity处理`ACTION_NDEF_DISCOVERED`类型的intent *或者* NFC标签不包含**NDEF**格式的数据（使用了其他已知技术）*或者* **NDEF**格式的标签不能被标签分发系统映射成正确的Intent，则会发送优先级较低的`ACTION_TECH_DISCOVERED`intent来尝试启动Activity（不会发送`ACTION_NDEF_DISCOVERED`）。
3. 如果仍然没有Activity响应上述两个Action，则系统会发送优先级最低的`ACTION_TAG_DISCOVERED`的Intent。




图为标签分发系统：

![](https://i.loli.net/2019/03/02/5c7a184962704.png)





#### NFC数据格式

NFC的数据格式可以分为NDEF和非NDEF两种。**NDEF**是NFC Forum定义的一种标准格式，在Android中得到最大范围的支持，是Android最推荐使用的格式。

* **NDEF数据**在AndroidSDK中被封装成一个包含多个**NdefRecord**的**NdefMessage**，**android.nfc.tech.Ndef**类封装了各种便捷的读写数据的操作。

  前面说过当Android设备发现NDEF格式的标签之后会发出`ACTION_NDEF_DISCOVERED`类型的Intent。通常在intent中还会带上更加具体的mime type & URI等数据，以便筛选更加具体的Activity来进行处理（这个是Android推荐使用这个格式的原因之一）。下面就来说说，NDEF具体的数据格式以及Android如何分析NDEF数据来生成对应的Intent。

  **NDEF数据格式**：

  ![](https://i.loli.net/2019/03/02/5c7a185c4e7fc.png)

  1. **TNF** 3bits的字段，标识了如何解析第二个字段type。包含的值见表一：

  表一，TNF字段以及映射：

  | TNF               | 映射的Intent                                |
  | :---------------- | ---------------------------------------- |
  | TNF_ABSOLUTE_URI  | 基于*type*的URI                             |
  | TNF_EMPTY         | 无映射，降级到`ACTION_TECH_DISCOVERED`          |
  | TNF_EXTERNAL_TYPE | 基于*type*的URI. type形式为 <domain>:<service>。映射后的URI形式为： `vnd.android.nfc://ext/<domain>:<service>`. |
  | TNF_MIME_MEDIA    | 基于type的MIME类型                            |
  | TNF_UNCHANGED     | 无效，降级为`ACTION_TECH_DISCOVERED`           |
  | TNF_UNKNOWN       | 降级为`ACTION_TECH_DISCOVERED`              |
  | TNF_WELL_KNOWN    | 基于type（RTD)的MIME type or URI ，需要再根据type的值进行映射（见表二） |

  表二，RTDs for TNF_WELL_KNOWN以及映射：

  | RTD                     | 映射的Intent                   |
  | ----------------------- | --------------------------- |
  | RTD_ALTERNATIVE_CARRIER | 降级为`ACTION_TECH_DISCOVERED` |
  | RTD_HANDOVER_CARRIER    | 降级为`ACTION_TECH_DISCOVERED` |
  | RTD_HANDOVER_REQUEST    | 降级为`ACTION_TECH_DISCOVERED` |
  | RTD_HANDOVER_SELECT     | 降级为`ACTION_TECH_DISCOVERED` |
  | RTD_SMART_POSTER        | 基于解析payload的URI             |
  | RTD_TEXT                | MIME类型 text/p               |
  | RTD_URI                 | 基于payloadd的URI              |

  2. **type**,描述Record的类型，如果TNF为TNF_WELL_KNOWN，则这个字段指定RTD，见上表二。
  3. **ID**，字段的唯一标识，一般不常用，可以用来标识一张标签。
  4. **payload**，有效荷载，保存真实的数据。

  

  **创建各种类型的NDEF数据&响应对应NDEF 数据的IntentFilter实例**

  1. 创建TNF_ABSOLUTE_URI类型数据:

```java
	NdefRecord uriRecord = new NdefRecord(
    NdefRecord.TNF_ABSOLUTE_URI ,
    "http://developer.android.com/index.html".getBytes(Charset.forName("US-ASCII")),
    new byte[0], new byte[0]);
```

​	响应的intent filter：

```xml
<intent-filter>
    <action android:name="android.nfc.action.NDEF_DISCOVERED" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:scheme="http"
        android:host="developer.android.com"
        android:pathPrefix="/index.html" />
</intent-filter>
```
​	2. 创建TNF_MIME_MEDIA类型的数据：

```java
  NdefRecord mimeRecord = NdefRecord.createMime("application/vnd.com.example.android.beam",
      "Beam me up, Android".getBytes(Charset.forName("US-ASCII")));
```

  	对应的intent filter:

```xml
  <intent-filter>
      <action android:name="android.nfc.action.NDEF_DISCOVERED" />
      <category android:name="android.intent.category.DEFAULT" />
      <data android:mimeType="application/vnd.com.example.android.beam" />
  </intent-filter>
```
​	3. 创建TNF_EXTERNAL_TYPE类型的数据:

```java
  byte[] payload; //assign to your data
  String domain = "com.example"; //usually your app's package name
  String type = "externalType";
  NdefRecord extRecord = NdefRecord.createExternal(domain, type, payload);
```

  	对应的intent filter:

```xml
  <intent-filter>
      <action android:name="android.nfc.action.NDEF_DISCOVERED" />
      <category android:name="android.intent.category.DEFAULT" />
      <data android:scheme="vnd.android.nfc"
          android:host="ext"
          android:pathPrefix="/com.example:externalType"/>
  </intent-filter>
```

​	4. 创建TNF_WELL_KNOWN with RTD_TEXT类型的数据：

```java
public NdefRecord createTextRecord(String payload, Locale locale, boolean encodeInUtf8) {
    byte[] langBytes = locale.getLanguage().getBytes(Charset.forName("US-ASCII"));
    Charset utfEncoding = encodeInUtf8 ? Charset.forName("UTF-8") : Charset.forName("UTF-16");
    byte[] textBytes = payload.getBytes(utfEncoding);
    int utfBit = encodeInUtf8 ? 0 : (1 << 7);
    char status = (char) (utfBit + langBytes.length);
    byte[] data = new byte[1 + langBytes.length + textBytes.length];
    data[0] = (byte) status;
    System.arraycopy(langBytes, 0, data, 1, langBytes.length);
    System.arraycopy(textBytes, 0, data, 1 + langBytes.length, textBytes.length);
    NdefRecord record = new NdefRecord(NdefRecord.TNF_WELL_KNOWN,
    NdefRecord.RTD_TEXT, new byte[0], data);
    return record;
}
```

​	对应的Intent filter:

```xml
<intent-filter>
    <action android:name="android.nfc.action.NDEF_DISCOVERED" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:mimeType="text/plain" />
</intent-filter>

```

​	

​	**读写NDEF格式的NFC数据**：

​	1. Intent中包含三个相关的数据：

​	**EXTRA_TAG** 扫描到的标签对象

​	**EXTRA_NDEF_MESSAGES** 从标签中获取到的NDEF数据对象，只有ACTION_NDEF_DISCOVERED类型的				Intent包含

​	**EXTRA_ID** 标签的ID（可选）

​	2 .解析NDEF数据的代码一般为：

```java
@Override
protected void onNewIntent(Intent intent) {
    super.onNewIntent(intent);
    ...
    if (intent != null && NfcAdapter.ACTION_NDEF_DISCOVERED.equals(intent.getAction())) {
        Parcelable[] rawMessages =
            intent.getParcelableArrayExtra(NfcAdapter.EXTRA_NDEF_MESSAGES);
        if (rawMessages != null) {
            NdefMessage[] messages = new NdefMessage[rawMessages.length];
            for (int i = 0; i < rawMessages.length; i++) {
                messages[i] = (NdefMessage) rawMessages[i];
            }
            // Process the messages array.
            ...
        }
    }
}
```

​	写NDEF数据的代码一般为：

```java
 @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        Tag tag = intent.getParcelableExtra(NfcAdapter.EXTRA_TAG);
        NdefRecord ndefRecord = NdefRecord.createExternal("domain", "service", "content".getBytes());
        NdefMessage ndefMessage = new NdefMessage(new NdefRecord[] {ndefRecord});
        Ndef ndef = Ndef.get(tag); //获取Ndef tech的对象
        if (ndef != null) { //非NDEF数据
            try {
                ndef.connect();
                ndef.writeNdefMessage(ndefMessage);
            } catch (IOException e) {
                e.printStackTrace();
            } catch (FormatException e) {
                e.printStackTrace();
            }
        } else { //可格式化为NDEF数据
            NdefFormatable ndefFormatable = NdefFormatable.get(tag);
            if (ndefFormatable != null) {
                try {
                    ndefFormatable.connect();
                    ndefFormatable.format(ndefMessage);
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (FormatException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```

​	

  

* **非NDEF数据**可以配合**android.nfc.tech**包下的对应的类来进行使用。


  每个NFC标签可能支持不同种类的**technologies**(不同格式数据的读写），可以通过如下代码获取标签支持的**techonologies**:

```java
  mTag = getIntent().getParcelableExtra(NfcAdapter.EXTRA_TAG);
  String[] teches = mTag.getTechList();
```

  Android支持的常见Technology如下表：

| Class            | Description                              |
| ---------------- | ---------------------------------------- |
| `TagTechnology`  | The interface that all tag technology classes must implement. |
| `NfcA`           | Provides access to NFC-A (ISO 14443-3A) properties and I/O operations. |
| `NfcB`           | Provides access to NFC-B (ISO 14443-3B) properties and I/O operations. |
| `NfcF`           | Provides access to NFC-F (JIS 6319-4) properties and I/O operations. |
| `NfcV`           | Provides access to NFC-V (ISO 15693) properties and I/O operations. |
| `IsoDep`         | Provides access to ISO-DEP (ISO 14443-4) properties and I/O operations. |
| `Ndef`           | Provides access to NDEF data and operations on NFC tags that have been formatted as NDEF. |
| `NdefFormatable` | Provides a format operations for tags that may be NDEF formattable. |

Android设备可选支持的Technology：

| Class              | Description                              |
| ------------------ | ---------------------------------------- |
| `MifareClassic`    | Provides access to MIFARE Classic properties and I/O operations, if this Android device supports MIFARE. |
| `MifareUltralight` | Provides access to MIFARE Ultralight properties and I/O operations, if this Android device supports MIFARE. |



前面讲解过，标签在某些情况下会降级为`ACTION_TECH_DISCOVERED`类型的Intent，对应的xml文件为:

```xml
<activity>
...
<intent-filter>
    <action android:name="android.nfc.action.TECH_DISCOVERED"/>
</intent-filter>

<meta-data android:name="android.nfc.action.TECH_DISCOVERED"
    android:resource="@xml/nfc_tech_filter" />
...
</activity>
```

其中**android:resource="@xml/nfc_tech_filter"**指定NFC technology的过滤文件，放在`res/xml`目录下任意命名。其中指定的technologies必须是所识别的标签支持的technologies的子集，才能匹配到。例如：

```xml
<resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">
    <tech-list>
        <tech>android.nfc.tech.IsoDep</tech>
        <tech>android.nfc.tech.NfcA</tech>
        <tech>android.nfc.tech.NfcB</tech>
        <tech>android.nfc.tech.NfcF</tech>
        <tech>android.nfc.tech.NfcV</tech>
        <tech>android.nfc.tech.Ndef</tech>
        <tech>android.nfc.tech.NdefFormatable</tech>
        <tech>android.nfc.tech.MifareClassic</tech>
        <tech>android.nfc.tech.MifareUltralight</tech>
    </tech-list>
</resources>
```

或者分开多个**<tech-list>**,他们之间相互独立，匹配一个就好。

```xml
<resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">
    <tech-list>
        <tech>android.nfc.tech.NfcA</tech>
        <tech>android.nfc.tech.Ndef</tech>
    </tech-list>
</resources>

<resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">
    <tech-list>
        <tech>android.nfc.tech.NfcB</tech>
        <tech>android.nfc.tech.Ndef</tech>
    </tech-list>
</resources>
```



**各种类型的非NDEF数据格式的读写**

查看对应的规格书按照协议来使用，在developer.android.com中可以简单地查看一种数据格式的说明：如 [MIFAREClassic](https://developer.android.com/reference/android/nfc/tech/MifareClassic.html)



### P2P mode

即AndroidBeam功能，使两台Android设备进行快速的数据交换。发送数据的设备打开发送数据的应用（相当于一张标签），接受数据的设备解锁靠近发送数据的设备之后，发送数据的设备UI显示“Touch to Beam",点击即可将数据发送给接收端（接收端设备接受到数据通过标签分发系统决定打开的应用进行处理）。

相关的API有两个：

**NfcAdapter.setNdefPushMessage()** :发送端设置要发送的NDEF数据。两台设备靠近之后点击”Touch to Beam"发送此数据。

**NfcAdapter.setNdefPushMessageCallback()**:此方法接受一个**NfcAdapter.CreateNdefMessageCallback**类型的回调接口，此接口在两台设备靠近时回调进行NDEF数据的创建，然后点击“Touch to Beam"发送数据。

两个接口的唯一不同在于：接口1一开始便确定了要发送的数据；接口2在两个设备靠近时才创建要发送的NDEF数据。

*代码示例*(Demo中onCreate中为发送端逻辑，onResume中为接收端处理逻辑):

```java
public class MainActivity extends AppCompatActivity {

    private NfcAdapter mNfcAdapter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mNfcAdapter = NfcAdapter.getDefaultAdapter(this);
        if (mNfcAdapter == null) {
            Toast.makeText(this, "not support nfc", Toast.LENGTH_SHORT).show();
            finish();
            return;
        }

        final NdefRecord ndefRecord = 		 NdefRecord.createMime("application/vnd.com.example.android.beam", "hello android".getBytes());
        mNfcAdapter.setNdefPushMessageCallback(new NfcAdapter.CreateNdefMessageCallback() {
            @Override
            public NdefMessage createNdefMessage(NfcEvent nfcEvent) {
                Log.v("seewo", "createNdefMessage");
                return new NdefMessage(ndefRecord);
            }
        }, this);
//        mNfcAdapter.setNdefPushMessage(new NdefMessage(ndefRecord), this); //接口一的实现
    }

    @Override
    protected void onResume() {
        super.onResume();
        //接收端的处理逻辑
        if (NfcAdapter.ACTION_NDEF_DISCOVERED.equals(getIntent().getAction())) {
            processIntent(getIntent());
        }
    }

    @Override
    public void onNewIntent(Intent intent) {
        setIntent(intent);
    }

    void processIntent(Intent intent) {
        Parcelable[] rawMessages = 		     intent.getParcelableArrayExtra(NfcAdapter.EXTRA_NDEF_MESSAGES);
        NdefMessage msg = (NdefMessage) rawMessages[0];
        String info = new String(msg.getRecords()[0].getPayload());
        Toast.makeText(this, info, Toast.LENGTH_SHORT).show();
    }
}

```



###  Card emulation mode

Android4.4之前通过在Android设备中内置一个芯片来实现。通信路径如下图：

![](https://i.loli.net/2019/03/02/5c7a1893b60b8.png)



Android4.4之后可以通过纯软件模拟NFC卡。通信路径如下图：

![](https://i.loli.net/2019/03/02/5c7a189e94ec6.png)



在Android上层程序只需实现一个服务，并且给服务绑定一个ID，便可以模拟成NFC卡，根据读卡器发出的ID响应数据。



```java
public class MyHostApduService extends HostApduService {
    @Override
    public byte[] processCommandApdu(byte[] apdu, Bundle extras) {
       ...
    }
    @Override
    public void onDeactivated(int reason) {
       ...
    }
}
```



```xml
<service android:name=".MyHostApduService" android:exported="true"
         android:permission="android.permission.BIND_NFC_SERVICE">
    <intent-filter>
        <action android:name="android.nfc.cardemulation.action.HOST_APDU_SERVICE"/>
    </intent-filter>
    <meta-data android:name="android.nfc.cardemulation.host_apdu_service"
               android:resource="@xml/apduservice"/>
</service>
```



```xml
<host-apdu-service xmlns:android="http://schemas.android.com/apk/res/android"
           android:description="@string/servicedesc"
           android:requireDeviceUnlock="false">
    <aid-group android:description="@string/aiddescription"
               android:category="other">
        <aid-filter android:name="F0010203040506"/>
        <aid-filter android:name="F0394148148100"/>
    </aid-group>
</host-apdu-service>
```



---



#### 相关资料

1.NDEF数据解析实例：[http](http://note.youdao.com/share/?id=8b45d342e34d2bce0fade2218bafd79c&type=note)[://note.youdao.com/share/?id=8b45d342e34d2bce0fade2218bafd79c&type=note#](http://note.youdao.com/share/?id=8b45d342e34d2bce0fade2218bafd79c&type=note)[/](http://note.youdao.com/share/?id=8b45d342e34d2bce0fade2218bafd79c&type=note)

2.AndnroidNFC官网介绍：[https](https://developer.android.com/guide/topics/connectivity/nfc/index.html)[://](https://developer.android.com/guide/topics/connectivity/nfc/index.html)[developer.android.com/guide/topics/connectivity/nfc/index.html](https://developer.android.com/guide/topics/connectivity/nfc/index.html)

3.NFC相关的应用：NXPTagWriter、NXPTagInfo、MIFARE经典工具