# NSP өөрчлөлтүүд

Энэ баримт нь `Next-Social-Platform/openim-sdk-core` fork-ийн `nsp` branch дээр upstream
[`openimsdk/openim-sdk-core`](https://github.com/openimsdk/openim-sdk-core)-оос ялгарах өөрчлөлтүүдийг тайлбарлана:

1. [Message reaction](#1-message-reaction)
2. [Flutter SDK-д зориулсан build](#2-flutter-sdk-д-зориулсан-build-makefile)

---

## 1. Message reaction

Хэрэглэгч мессеж дээр reaction (emoji гэх мэт) дарах боломж. Энэ логикийг хуучин `im-chat-sdk-core`
төслөөс авчирсан.

### Ажиллах зарчим

Reaction нь **тусдаа мессеж** хэлбэрээр илгээгддэг:

- `contentType = 121` (`constant.ReactionMessageModifier`)
- Content нь `ReactionElem` бөгөөд аль мессежид reaction дарсныг `clientMsgID`-аар заана
- Local DB-д reaction мессежийн `parent_msg_id` баганад зорилтот мессежийн `clientMsgID` хадгалагдана
- History list авахад мессеж бүрийн `reactions` талбарт **хэрэглэгч бүрийн хамгийн сүүлийн**
  reaction-ийг нэгтгэж өгнө

Reaction-ийг устгахдаа `status = 0`-той шинэ reaction мессеж илгээнэ. Хэрэглэгчийн сүүлийн
reaction нь `status = 0` бол client тал түүнийг харуулахгүй.

```
Мессеж A (clientMsgID = "abc")
  ├─ reaction мессеж (sendID = user1, parentMsgID = "abc", {type: "like", status: 1})
  ├─ reaction мессеж (sendID = user2, parentMsgID = "abc", {type: "love", status: 1})
  └─ reaction мессеж (sendID = user1, parentMsgID = "abc", {type: "like", status: 0})  ← user1-ийн сүүлийнх

GetAdvancedHistoryMessageList → A.reactions = [
  {user: user1, type: "like", status: 0},
  {user: user2, type: "love", status: 1},
]
```

### Өгөгдлийн бүтэц

`sdk_struct/sdk_struct.go`:

```go
// Reaction мессежийн content
type ReactionElem struct {
	Status      int32  `json:"status,omitempty"`      // 1 - идэвхтэй, 0 - устгасан
	Type        string `json:"type,omitempty"`        // reaction-ийн төрөл
	ClientMsgID string `json:"clientMsgID,omitempty"` // reaction дарсан мессежийн ID
	Info        string `json:"info,omitempty"`
}

// History list дээр мессежид нэгтгэгдэх reaction
type ReactionItem struct {
	Status   int32    `json:"status,omitempty"`
	Type     string   `json:"type,omitempty"`
	User     CardElem `json:"user"`     // reaction дарсан хэрэглэгч (userID, nickname, faceURL)
	SendTime int64    `json:"sendTime"`
}
```

`MsgStruct`-д нэмэгдсэн талбарууд:

| Талбар         | Төрөл             | Тайлбар                                           |
|----------------|-------------------|---------------------------------------------------|
| `parentMsgID`  | `string`          | Reaction мессеж бол зорилтот мессежийн ID         |
| `reactionElem` | `*ReactionElem`   | Reaction мессежийн content (`contentType = 121`)  |
| `reactions`    | `[]*ReactionItem` | Энэ мессежид дарагдсан reaction-ууд               |
| `isReact`      | `bool`            | Хуучин SDK-тай нийцүүлэхэд зориулсан              |

`pkg/constant/constant.go`:

```go
ReactionMessageModifier = 121
ReactionMessageDeleter  = 122 // одоогоор ашиглагдаагүй
```

### Public API

```go
// open_im_sdk/conversation_msg.go
func CreateReactionMessage(operationID, clientMsgID, reaction string, status int32) string
```

Үүсгэсэн мессежийг энгийн мессеж шиг `SendMessage`-ээр илгээнэ. wasm талд `createReactionMessage`
нэрээр гарсан.

Flutter (`open-im-sdk-flutter`):

```dart
final msg = await OpenIM.iMManager.messageManager.createReactionMessage(
  clientMsgID: target.clientMsgID!, // reaction дарах мессеж
  reaction: 'like',
  status: 1,                        // 1 - нэмэх, 0 - устгах
);
await OpenIM.iMManager.messageManager.sendMessage(
  message: msg,
  offlinePushInfo: OfflinePushInfo(),
  userID: userID,   // ганцаарчилсан чат бол
  groupID: groupID, // групп чат бол
);

// History list дээр
for (final r in message.reactions ?? []) {
  print('${r.user?.nickname}: ${r.type} (${r.status})');
}
```

### Өөрчлөгдсөн файлууд

| Файл | Өөрчлөлт |
|------|----------|
| `pkg/constant/constant.go` | `ReactionMessageModifier`, `ReactionMessageDeleter` constant |
| `sdk_struct/sdk_struct.go` | `ReactionElem`, `ReactionItem`, `MsgStruct`-ийн шинэ талбарууд |
| `pkg/db/model_struct/data_model_struct.go` | `LocalChatLog.ParentMsgID` (`parent_msg_id` багана) |
| `pkg/db/chat_log_model.go` | Хүснэгт үүсгэх SQL-д `parent_msg_id`, хуучин хүснэгтийн migration, `GetReactionMessagesByParentMsgIDs` |
| `pkg/db/db_init.go` | Migration шалгасан хүснэгтүүдийн cache (`reactionColumnChecked`) |
| `pkg/db/db_interface/databse.go` | Interface-д `GetReactionMessagesByParentMsgIDs` |
| `pkg/converter/conversation.go`, `internal/conversation_msg/conversion.go` | `ReactionElem` ↔ content хөрвүүлэлт, `ParentMsgID` дамжуулах |
| `internal/conversation_msg/create_message.go` | `CreateReactMessage` |
| `internal/conversation_msg/api.go` | `sendMessage` / `sendMessageNotOss` дээр 121 төрлийг дэмжих |
| `internal/conversation_msg/conversation_msg.go` | Ирсэн reaction мессежийг parse хийх (`doMsgNew`) |
| `internal/conversation_msg/conversation.go` | `fillMessageReactions`: history list дээр reaction нэгтгэх |
| `open_im_sdk/conversation_msg.go` | `CreateReactionMessage` public API |
| `wasm/...` | `createReactionMessage` export, IndexedDB stub |

### DB migration

Upstream SDK chat log хүснэгтүүдийг raw SQL-ээр үүсгэдэг (AutoMigrate хийдэггүй). Иймээс:

- Шинэ хүснэгтийн `CREATE TABLE`-д `parent_msg_id CHAR(64)` нэмсэн.
- Өмнө үүссэн хүснэгтэд `initChatLog` анх дуудагдахад `parent_msg_id` багана байхгүй бол
  `ALTER TABLE ... ADD COLUMN` хийж, `index_parent_msg_id_<conversationID>` index үүсгэнэ.
  Шалгалт нэг хүснэгтэд процесс бүрт нэг л удаа хийгдэнэ.

Хуучин хэрэглэгчид DB-ээ устгах шаардлагагүй.

### Хуучин хэрэгжүүлэлтээс ялгаатай нь

- Хуучин код gorm-ийн `Preload("Reactions")` relation ашигладаг байсан. Шинэ SDK хүснэгтээ raw SQL-ээр
  үүсгэдэг тул `parent_msg_id`-аар шууд хайдаг тусдаа query (`GetReactionMessagesByParentMsgIDs`) ашигласан.
- Устгагдсан (`status >= 4`) reaction мессежийг тооцохгүй.
- `reactions` жагсаалт тогтмол дараалалтай (хамгийн сүүлийнх эхэндээ). Хуучин код map ашигладаг
  байсан тул дараалал санамсаргүй байв.

### Анхаарах зүйлс

- **Web (wasm):** IndexedDB талд `GetReactionMessagesByParentMsgIDs` хоосон stub, тиймээс web дээр
  `reactions` хоосон ирнэ. Хэрэгтэй бол JS талын DB-д хэрэгжүүлэх ёстой.
- **Reaction мессеж history list дээр харагдана.** Хуучин SDK-тай адилаар шүүгээгүй. Client тал
  `contentType == 121` мессежийг чатын жагсаалтаас нууна.
- **Reaction нэгтгэлт зөвхөн `GetAdvancedHistoryMessageList` (болон `...Reverse`) дээр** хийгддэг.
  Шинээр ирсэн мессежийн event-д `reactions` ирэхгүй. Шинэ reaction ирэхэд client тал `parentMsgID`-аар
  нь холбогдох мессежийг шинэчилнэ.

---

## 2. Flutter SDK-д зориулсан build (Makefile)

Go SDK-г `gomobile`-оор iOS / Android library болгон build хийж,
[`open-im-sdk-flutter`](https://github.com/Next-Social-Platform/open-im-sdk-flutter) төсөл рүү шууд хуулна.

### Командууд

| Команд | Үйлдэл |
|--------|--------|
| `make flutter` | Android болон iOS хоёуланг build хийж хуулна |
| `make flutter-android` | Зөвхөн Android AAR |
| `make flutter-ios` | Зөвхөн iOS xcframework |
| `make gomobile-install` | `gomobile` / `gobind` байхгүй бол суулгана (бусад target автоматаар дууддаг) |

Үр дүн хуулагдах газар:

| Платформ | Build гаралт | Flutter төсөл дэх байршил |
|----------|--------------|---------------------------|
| Android | `build/mobile/android/open_im_sdk.aar` | `android/libs/io/openim/core-sdk/0.0.1/core-sdk-0.0.1.aar` |
| iOS | `build/mobile/ios/OpenIMCore.xcframework` | `ios/Framework/OpenIMCore.xcframework` |

`build/mobile/` нь `.gitignore`-д орсон.

### Тохиргоо (хувьсагч)

Бүх хувьсагчийг командын мөрөөс өөрчилж болно, жишээ нь `make flutter FLUTTER_SDK_DIR=~/work/open-im-sdk-flutter`.

| Хувьсагч | Default | Тайлбар |
|----------|---------|---------|
| `FLUTTER_SDK_DIR` | `/Users/batxac/dev/nsp/open-im-sdk-flutter` | Flutter plugin төслийн зам |
| `MOBILE_BUILD_DIR` | `build/mobile` | Түр build гаралт |
| `ANDROID_HOME` | `~/Library/Android/sdk` | Android SDK |
| `ANDROID_NDK_HOME` | `$ANDROID_HOME/ndk/` доторх хамгийн шинэ хувилбар | Android NDK |
| `ANDROID_API` | `21` | Android min API (Flutter plugin-ийн `minSdkVersion`-тэй ижил) |

### Шаардлага

- Go (`go.mod`-д заасан хувилбар буюу түүнээс шинэ)
- **Android:** Android SDK болон NDK (Android Studio → SDK Manager → NDK). NDK r26+ нь API 21-ээс доош дэмждэггүй.
- **iOS:** Xcode болон Command Line Tools
- **Дискний зай:** iOS build 3 архитектур (device arm64, simulator arm64/amd64) хийдэг тул хэдэн GB түр зай шаардана.

### Дотооддоо хэрхэн ажилладаг вэ

1. `gomobile-install`: `$(go env GOPATH)/bin/gomobile` байхгүй бол `gomobile`, `gobind`-ийг суулгаад `gomobile init` хийнэ.
2. `go.mod` / `go.sum`-ыг `build/mobile/` руу нөөцөлнө.
3. Суулгасан `gomobile`-тэй **ижил хувилбарын** `golang.org/x/mobile/bind`-ийг түр `go get` хийнэ
   (шинэ `gomobile` нь module-д x/mobile байхыг шаарддаг).
4. `gomobile bind` ажиллана.
5. Амжилттай болсон ч, алдаа гарсан ч `trap` нь `go.mod` / `go.sum`-ыг **анхны байдалд нь сэргээнэ**.
   `gomobile` нь Go хувилбар болон dependency-уудыг өөрчилдөг тул ингэхгүй бол repo бохирдоно.
6. Үр дүнг Flutter төсөл рүү хуулна.

### Flutter төсөл дэх тохиргоо

Flutter plugin нь remote биш, энд build хийсэн local SDK-г ашиглахаар тохируулагдсан:

`android/build.gradle`:

```groovy
rootProject.allprojects {
    repositories {
        maven { url "$dir/libs" }   // local AAR
        ...
    }
}
dependencies {
    implementation 'io.openim:core-sdk:0.0.1@aar'
}
```

`ios/flutter_openim_sdk.podspec`:

```ruby
# s.dependency 'OpenIMSDKCore','3.8.1'
s.vendored_frameworks = 'Framework/OpenIMCore.xcframework'
```

SDK-д өөрчлөлт хийсний дараа `make flutter` ажиллуулаад, Flutter app-аа дахин build хийнэ
(iOS дээр `cd ios && pod install` хийх шаардлагатай байж магадгүй).

### Хуучин `make ios` / `make android` target-ууд

Upstream-ийн эдгээр target хэвээр байгаа. Гэхдээ Flutter-д **`make flutter*`-ийг ашиглана уу**:

- Үр дүнг Flutter төсөл рүү хуулдаггүй.
- `go get` хийж `go.mod`-ыг өөрчлөөд буцааж сэргээдэггүй.
- `make ios` нь `open_im_sdk/ws_wrapper/` зэрэг source файлыг **устгадаг**.

`make android` дээр `ANDROID_NDK_HOME` болон `-androidapi 21` тохиргоог нэмж зассан (өмнө нь API 16
ашиглаад шинэ NDK дээр унадаг байсан).

### Түгээмэл алдаа

| Алдаа | Шийдэл |
|-------|--------|
| `errno=28` / `write64() failed` | Дискний зай дууссан. `go clean -cache`, эсвэл temp-ийг өөр диск рүү чиглүүлэх: `TMPDIR=/path/tmp GOCACHE=/path/cache make flutter` |
| `no usable NDK ... unsupported API version 16` | Хуучин `make android`-ийг ашигласан. `make flutter-android` ашиглах |
| `Android NDK not found` | SDK Manager-ээс NDK суулгах, эсвэл `ANDROID_NDK_HOME=...` зааж өгөх |
| `gomobile bind requires golang.org/x/mobile` | `gomobile`-ийг Makefile-аас гадуур ажиллуулсан. `make flutter*` ашиглах |
| `missing go.sum entry` | `go.mod` / `go.sum` бохирдсон: `git checkout -- go.mod go.sum` |
