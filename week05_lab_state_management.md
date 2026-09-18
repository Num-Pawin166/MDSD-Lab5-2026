# ใบงานปฏิบัติสัปดาห์ที่ 5: State Management ด้วย Provider และ Riverpod

**วิชา** การพัฒนาซอฟต์แวร์สำหรับอุปกรณ์เคลื่อนที่ | **เครื่องมือ** Flutter, Provider, Riverpod, Google AI Studio (Gemini API)


---

## วัตถุประสงค์การเรียนรู้

เมื่อทำใบงานนี้เสร็จสิ้น ผู้เรียนจะสามารถ

1. สร้าง Global State ด้วย `ChangeNotifier` และ `ChangeNotifierProvider` ให้หลายหน้าจอใช้ข้อมูลร่วมกันได้ถูกต้อง
2. แยกแยะและใช้งาน `context.watch()` กับ `context.read()` ได้ตรงตามสถานการณ์
3. รีแฟกเตอร์โค้ดที่เขียนด้วย `setState` แบบมี Prop Drilling ให้กลายเป็นโครงสร้างที่ใช้ Provider
4. ทดลองสร้าง Provider แบบเดียวกันด้วย Riverpod และเปรียบเทียบความแตกต่างของโค้ดจริง
5. ใช้ Google AI Studio (Gemini) ช่วยวิเคราะห์และให้เหตุผลในการเลือกเครื่องมือ State Management ให้เหมาะกับโจทย์
6. ออกแบบและตัดสินใจเลือกวิธีจัดการ State ด้วยตนเองสำหรับฟีเจอร์ใหม่ พร้อมทั้งอธิบายเหตุผลของการเลือกนั้นได้

## สิ่งที่ต้องเตรียมก่อนเริ่ม

- ติดตั้ง Flutter SDK และ VS Code เรียบร้อยจากสัปดาห์ที่ 1
- โปรเจกต์ Flutter `campus_marketplace` จากสัปดาห์ที่ 4 ที่มี Multi-screen Navigation ด้วย Go Router (หากยังไม่มี ให้สร้างโปรเจกต์ใหม่ชื่อ `campus_marketplace` ด้วยคำสั่ง `flutter create campus_marketplace`)
- บัญชี Google AI Studio ที่สร้างไว้ตั้งแต่สัปดาห์ที่ 1

---

## ทบทวนทฤษฎีก่อนเริ่มลงมือปฏิบัติ

ก่อนลงมือเขียนโค้ด ให้ทบทวนแนวคิดสำคัญต่อไปนี้ให้แม่นยำ เพราะทุกขั้นตอนในใบงานนี้อ้างอิงจากหลักการเหล่านี้โดยตรง 

### State สองชนิด: Ephemeral State กับ App State 

**Ephemeral State** คือข้อมูลที่มีความหมายเฉพาะภายใน Widget เดียวหรือกลุ่มเล็ก ๆ ที่อยู่ใกล้กัน ไม่มี Widget อื่นในแอปจำเป็นต้องรู้ (เช่น ค่าที่พิมพ์ค้างใน TextField, ตำแหน่งแท็บที่เลือกอยู่) เปรียบเหมือนโน้ตกระดาษบนโต๊ะทำงานของตัวเอง ใช้ `setState` จัดการก็เพียงพอ

**App State** คือข้อมูลที่ต้องใช้ร่วมกันโดยหลาย Widget ที่อาจอยู่คนละกิ่งของ Widget Tree หรือคนละ Route (เช่น สถานะล็อกอิน, รายการโปรด, ธีมสี) เปรียบเหมือนกระดานประกาศกลางที่ทุกแผนกต้องเห็นตรงกัน ต้องใช้ InheritedWidget, Provider หรือ Riverpod จัดการ

| มิติ | Ephemeral State | App State |
|---|---|---|
| ขอบเขตการมองเห็น | Widget เดียวหรือกลุ่มเล็ก ๆ | หลายหน้าจอ / ทั้งแอป |
| อายุการใช้งาน | สั้น มักหายเมื่อ Widget dispose | ยาว อยู่ตลอดการใช้งานแอป |
| เครื่องมือที่เหมาะสม | `setState` ภายใน StatefulWidget | InheritedWidget, Provider, Riverpod |

ในใบงานนี้ **รายการสินค้าที่บันทึกไว้ (Favorites)** คือตัวอย่างของ App State ชัดเจน เพราะต้องแสดงผลตรงกันทั้งที่ AppBar ของหน้า Home และในหน้า Favorites ที่แยก Route ออกไป — นี่คือเหตุผลที่ส่วนที่ 1 ของใบงานจะทำให้เห็นปัญหาก่อน แล้วส่วนที่ 2 จึงแก้ด้วย Provider

### Prop Drilling คือปัญหาอะไร 

เมื่อใช้ `setState` เก็บ App State ไว้ที่ Widget ต้นทาง (เช่น `HomePage`) แล้วต้องส่งค่า/ฟังก์ชันลงไปให้ Widget ลูกหลานที่อยู่ลึกหลายชั้นผ่าน constructor ทีละชั้น ทั้งที่ Widget ตัวกลางไม่ได้ใช้ค่าเหล่านั้นเองเลย เรียกปรากฏการณ์นี้ว่า **Prop Drilling** ยิ่ง Widget ตัวกลางเพิ่มขึ้น ยิ่งต้องแก้ constructor ซ้ำไปเรื่อย ๆ และปัญหาจะรุนแรงขึ้นไปอีกเมื่อต้องส่งข้อมูลข้าม Route (เพราะ constructor ส่งได้เฉพาะพ่อแม่ลูกในทรีเดียวกันเท่านั้น) — ส่วนที่ 1 ของใบงานนี้จะให้ลงมือสร้างปัญหานี้ด้วยตัวเองก่อน เพื่อให้เห็นภาพว่า Provider เข้ามาแก้อะไร

### หลักการของ Provider 

Provider คือแพ็กเกจที่ห่อหุ้ม `InheritedWidget` (กลไกกระจายข้อมูลลงทรีของ Flutter เอง) ให้ใช้งานง่ายขึ้น โดยผนวกกับ **ChangeNotifier** ซึ่งมีเมธอด `notifyListeners()` ทำหน้าที่เหมือน `setState` แต่ขยายขอบเขตจาก "Widget เดียว" เป็น "ผู้ฟังจำนวนเท่าใดก็ได้ทั่วทั้งแอป" หลักการสำคัญที่ต้องจำให้แม่นคือ

- **กฎทอง**: ทุกเมธอดที่แก้ไขข้อมูลใน Model ต้องเรียก `notifyListeners()` เสมอ ไม่งั้น UI จะไม่อัปเดต
- **`context.watch<T>()`**: อ่านค่าและสมัครเป็นผู้ติดตาม จะถูก rebuild ทุกครั้งที่ข้อมูลเปลี่ยน ใช้กับส่วนที่ต้อง *แสดงผล*
- **`context.read<T>()`**: อ่านค่าครั้งเดียวเพื่อเรียกเมธอด ไม่สมัครเป็นผู้ติดตาม ใช้กับการกดปุ่ม (`onPressed`) เพื่อไม่ให้ Widget นั้น rebuild โดยไม่จำเป็น

### หลักการของ Riverpod 

Riverpod คือวิวัฒนาการของ Provider ที่แก้ข้อจำกัดเรื่องการต้องพึ่งพา `BuildContext` โดยใช้ `WidgetRef` แทน (`ref.watch(...)` เทียบเท่า `context.watch<T>()` และ `ref.read(...)` เทียบเท่า `context.read<T>()`) และเพิ่มความปลอดภัยด้าน Type ตั้งแต่ตอนเขียนโค้ด (compile-time) แทนที่จะพังตอนรันจริง แนวคิดหลักเหมือนกับ Provider ทุกประการ เปลี่ยนแค่วิธีเข้าถึงข้อมูล

### กรอบการตัดสินใจเลือกเครื่องมือ

เริ่มจาก `setState` เสมอถ้าข้อมูลอยู่ในขอบเขต Widget เดียว เมื่อข้อมูลต้องใช้ข้ามหลายหน้าจอให้ยกระดับไปใช้ Provider ก่อน (เรียนรู้ง่ายกว่า) และเมื่อโปรเจกต์ต้องการ Unit Test ที่เข้มงวดหรือ Type Safety สูงขึ้น จึงค่อยพิจารณาย้ายไปใช้ Riverpod 

---

## ส่วนที่ 1: สร้างปัญหาให้เกิดขึ้น ก่อนแก้ไข (Prop Drilling Demo)

ก่อนใช้ Provider จะสร้างสถานการณ์ปัญหาขึ้นมาก่อน เพื่อให้เห็นภาพว่า Provider แก้ปัญหาอะไร ให้ทำตามทีละขั้นตอนต่อไปนี้ให้ครบ หลังจากนั้นจะได้แอปที่รันได้จริงและเห็นปัญหา Prop Drilling ชัดเจน

### ขั้นตอนที่ 1.1: สร้างโมเดล Item
สร้างโปรเจกต์ใหม่ชื่อ `campus_marketplace` ด้วยคำสั่ง `flutter create campus_marketplace` หลังจากนั้น
สร้างไฟล์ `lib/models/item.dart` และเพิ่มโค้ดต่อไปนี้ 

```dart
class Item {
  final String id;
  final String title;
  final double price;

  const Item({required this.id, required this.title, required this.price});
}

// ข้อมูลจำลอง (mock) ไว้ใช้ก่อน 
final catalog = <Item>[
  const Item(id: 'i1', title: 'หนังสือ Calculus มือสอง', price: 150),
  const Item(id: 'i2', title: 'หูฟังไร้สาย (สภาพดี 90%)', price: 450),
  const Item(id: 'i3', title: 'โคมไฟตั้งโต๊ะหอพัก', price: 120),
];
```

### ขั้นตอนที่ 1.2: สร้าง ItemCard (Widget ชั้นในสุด)

สร้างไฟล์ `lib/widgets/item_card.dart` — Widget นี้อยู่ชั้นล่างสุดของทรี เป็นตัวที่ **ใช้งานจริง** ทั้ง `savedItems` (เพื่อเช็คว่าไอเทมนี้ถูกบันทึกไปแล้วหรือยัง) และ `onSave` (เพื่อเรียกตอนกดปุ่ม)

```dart
import 'package:flutter/material.dart';
import '../models/item.dart';

class ItemCard extends StatelessWidget {
  final Item item;
  final List<Item> savedItems; // ต้องรับมาเพื่อเช็คว่าไอเทมนี้ถูกบันทึกแล้วหรือยัง (Prop Drilling)
  final void Function(Item item) onSave; // ฟังก์ชันที่ถูกส่งทอดมาจาก HomePage ผ่าน ItemListSection

  const ItemCard({
    super.key,
    required this.item,
    required this.savedItems,
    required this.onSave,
  });

  @override
  Widget build(BuildContext context) {
    // เช็คว่าไอเทมนี้ถูกบันทึกไปแล้วหรือยัง โดยเทียบ id กับรายการที่ส่งเข้ามา
    final alreadySaved = savedItems.any((i) => i.id == item.id);

    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      child: ListTile(
        title: Text(item.title),
        subtitle: Text('฿${item.price.toStringAsFixed(0)}'),
        trailing: ElevatedButton(
          // ปิดปุ่ม (onPressed: null) ถ้าบันทึกไปแล้ว ป้องกันการกดซ้ำสร้างรายการซ้ำ
          onPressed: alreadySaved ? null : () => onSave(item),
          child: Text(alreadySaved ? '❤️ บันทึกแล้ว' : '🤍 บันทึกเป็นรายการโปรด'),
        ),
      ),
    );
  }
}
```

### ขั้นตอนที่ 1.3: สร้าง ItemListSection (Widget ชั้นกลาง — จุดที่เกิด Prop Drilling)

สร้างไฟล์ `lib/widgets/item_list_section.dart` — สังเกตให้ดีว่า Widget นี้ **ไม่ได้ใช้** `savedItems` หรือ `onSave` โดยตรงเลย มันแค่รับพารามิเตอร์มาแล้วส่งต่อให้ `ItemCard` แต่ละใบเท่านั้น นี่คือปัญหา Prop Drilling

```dart
import 'package:flutter/material.dart';
import '../models/item.dart';
import 'item_card.dart';

class ItemListSection extends StatelessWidget {
  final List<Item> catalog;
  final List<Item> savedItems; // รับมาจาก HomePage แล้วต้อง "ส่งทอด" ต่อให้ ItemCard ทุกใบ
  final void Function(Item item) onSave; // ฟังก์ชันเดียวกันที่ต้องส่งทอดต่อเช่นกัน

  const ItemListSection({
    super.key,
    required this.catalog,
    required this.savedItems,
    required this.onSave,
  });

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      shrinkWrap: true,
      physics: const NeverScrollableScrollPhysics(),
      itemCount: catalog.length,
      itemBuilder: (context, index) {
        final item = catalog[index];
        // ตัวมันเองไม่แตะ savedItems/onSave เลย แค่ "ส่งผ่าน" ไปให้ ItemCard เท่านั้น
        return ItemCard(item: item, savedItems: savedItems, onSave: onSave);
      },
    );
  }
}
```

### ขั้นตอนที่ 1.4: สร้าง HomePage (Widget ชั้นบนสุด — เจ้าของ State ตัวจริง)

สร้าง/แก้ไขไฟล์ `lib/home_page.dart` ให้เป็น `StatefulWidget` ที่เก็บรายการสินค้าที่บันทึกไว้ (Favorites) ไว้ในตัวเอง แล้ว ใส่ค่าและฟังก์ชันลงไปให้ Widget ลูกทั้งสองชั้นที่สร้างไว้ในขั้นตอนที่ 1.2-1.3

```dart
import 'package:flutter/material.dart';
import 'models/item.dart';
import 'widgets/item_list_section.dart';

class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  final List<Item> _savedItems = []; // เก็บรายการโปรดไว้ใน State ของ HomePage เอง (ยังไม่ใช้ Provider)

  void _onSave(Item item) {
    setState(() {
      _savedItems.add(item); // แก้ไข List แล้วสั่ง rebuild ทั้งทรีที่อยู่ใต้ HomePage
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Campus Marketplace'),
        actions: [
          Padding(
            padding: const EdgeInsets.only(right: 12),
            child: Center(child: Text('❤️ ${_savedItems.length}')),
          ),
        ],
      ),
      body: ItemListSection(
        catalog: catalog,       // มาจาก item.dart ที่สร้างไว้ในขั้นตอนที่ 1.1
        savedItems: _savedItems, // ต้องส่งลงไปให้ ItemListSection แม้มันไม่ได้ใช้เอง
        onSave: _onSave,         // ส่งฟังก์ชันลงไปเช่นกัน — รวมเป็น "Prop Drilling" 2 ชั้น
      ),
    );
  }
}
```

แก้ไขไฟล์ `lib/main.dart` ให้เป็นดังนี้ เพื่อให้แอปเริ่มทำงานที่ `HomePage` (ยังไม่มี Provider ในขั้นตอนนี้ จะเพิ่มในส่วนที่ 2)

```dart
import 'package:flutter/material.dart';
import 'home_page.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      title: 'Campus Marketplace',
      debugShowCheckedModeBanner: false, // ปิดริบบิ้น DEBUG มุมขวาบน ไม่ให้บังไอคอนหัวใจใน AppBar
      home: HomePage(), // เรียก HomePage ที่สร้างไว้ในขั้นตอนที่ 1.4 เป็นหน้าแรกของแอป
    );
  }
}
```

> ✅ **Checkpoint 1.1** รันแอปและกดปุ่ม "🤍 บันทึกเป็นรายการโปรด" ที่สินค้าชิ้นใดก็ได้ ทดสอบว่า (ก) ตัวเลขในไอคอนหัวใจที่ AppBar เพิ่มขึ้นถูกต้อง และ (ข) ปุ่มของสินค้าที่กดไปแล้วเปลี่ยนเป็น "❤️ บันทึกแล้ว" และกดซ้ำไม่ได้ ถ่ายภาพหน้าจอที่เห็นทั้งสองอย่างนี้พร้อมกัน แล้วเปิดไฟล์ `item_card.dart` และ `item_list_section.dart` ให้เห็น constructor ที่ต้องรับพารามิเตอร์ส่งต่อ (Prop Drilling) ชัดเจน แนบส่งในรายงาน

**คำถาม**: ถ้าต้องเพิ่มหน้าจอ `FavoritesPage` ที่ต้องแสดงรายการที่บันทึกไว้ชุดเดียวกัน แต่ถูก push แยกออกไปเป็นอีก Route หนึ่ง จะเกิดปัญหาอะไรกับโค้ดแบบ Prop Drilling นี้ จงเขียนคำตอบสั้น ๆ 

```text
Constructor ส่งค่าได้เฉพาะจากแม่ไปลูกใน Widget Tree เดียวกันเท่านั้น แต่ FavoritesPage ถูก push
เป็นอีก Route หนึ่ง ซึ่ง Navigator จะสร้างมันเป็น "ทรีคนละกิ่ง" ที่แขวนอยู่ใต้ Navigator ไม่ใช่ใต้
HomePage จึงไม่มีสายพ่อ-ลูกให้ส่งค่าลงไปได้ตามธรรมชาติ ผลที่ตามมาคือ

1. ต้องส่ง _savedItems และ _onSave เข้าไปทาง constructor ของ FavoritesPage ตอน push เอง
   (MaterialPageRoute(builder: (_) => FavoritesPage(savedItems: _savedItems, onSave: _onSave)))
   ทำให้ HomePage ผูกติดกับ FavoritesPage แน่นขึ้นอีก
2. ที่แย่กว่านั้นคือ "ค่าจะไม่ซิงค์กัน" — FavoritesPage ได้รับ List ไปตอน push ครั้งเดียว
   เมื่อผู้ใช้กดลบในหน้านั้น setState ของ HomePage ไม่ถูกเรียก ตัวเลขที่ AppBar หน้า Home จึงค้าง
   และในทางกลับกัน ถ้าแก้ที่ Home หน้า Favorites ที่เปิดค้างอยู่ก็ไม่ rebuild ตาม
   เพราะ setState ของ HomePage สั่ง rebuild ได้แค่ทรีที่อยู่ใต้ตัวเองเท่านั้น
3. ถ้าจะให้ซิงค์จริงต้องส่ง callback กลับขึ้นมา (หรือ await ค่าที่ Navigator.pop ส่งกลับ)
   แล้ว setState ซ้ำอีกรอบ ซึ่งซับซ้อนขึ้นทุกครั้งที่เพิ่มหน้าจอใหม่

สรุป: Prop Drilling พังทันทีที่ข้อมูลต้องข้าม Route เพราะทั้ง "เส้นทางส่งค่า" และ "การแจ้งเตือน
ให้ rebuild" ถูกจำกัดอยู่แค่ในทรีเดียว — ซึ่งคือปัญหาที่ Provider เข้ามาแก้ในส่วนที่ 2
```

---

## ส่วนที่ 2: รีแฟกเตอร์ด้วย Provider

### ขั้นตอนที่ 2.1: ติดตั้งแพ็กเกจ

เพิ่ม dependency ในไฟล์ `pubspec.yaml`

```yaml
dependencies:
  flutter:
    sdk: flutter
  provider: ^6.1.2
```

รันคำสั่งในเทอร์มินัลของ VS Code

```bash
flutter pub get
```

### ขั้นตอนที่ 2.2: สร้าง FavoritesModel

สร้างไฟล์ `lib/models/favorites_model.dart` — นี่คือคลาสที่จะเป็น "Single Source of Truth" ของรายการโปรดทั้งแอป แทนที่การเก็บ State กระจัดกระจายแบบส่วนที่ 1

```dart
import 'package:flutter/foundation.dart';
import 'item.dart';

class FavoritesModel extends ChangeNotifier {
  final List<Item> _items = []; // ตั้งเป็น private (ขึ้นต้นด้วย _) เพื่อไม่ให้ภายนอกแก้ไขตรง ๆ ได้

  List<Item> get items => List.unmodifiable(_items); // เปิดให้อ่านได้ แต่แก้ไขผ่าน list นี้ไม่ได้
  int get itemCount => _items.length;
  double get totalValue => _items.fold(0, (sum, i) => sum + i.price);

  void add(Item item) {
    _items.add(item);
    notifyListeners(); // กฎทองของ ChangeNotifier: แก้ข้อมูลแล้วต้องแจ้งทุกครั้ง ไม่งั้น UI จะไม่อัปเดต
  }

  void remove(Item item) {
    _items.remove(item);
    notifyListeners();
  }

  void clear() {
    _items.clear();
    notifyListeners();
    // หมายเหตุ: เมธอดนี้ยังไม่ถูกเรียกใช้จากที่ใดในใบงานส่วนที่ 1-4
    // จะถูกนำไปใช้จริงในส่วนที่ 5 (ทำด้วยตนเอง)
  }
}
```

### ขั้นตอนที่ 2.3: ลงทะเบียน Provider ที่ราก main.dart

แก้ไข `lib/main.dart` ให้ครอบทั้งแอปด้วย `ChangeNotifierProvider` ตั้งแต่จุดสูงสุด

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'models/favorites_model.dart';
import 'home_page.dart';

void main() {
  runApp(
    // สร้าง FavoritesModel ขึ้นมาหนึ่งตัว แล้วให้ทุก Widget ใต้ MyApp เข้าถึงได้
    ChangeNotifierProvider(
      create: (context) => FavoritesModel(),
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Campus Marketplace',
      debugShowCheckedModeBanner: false, // ปิดริบบิ้น DEBUG มุมขวาบน ไม่ให้บังไอคอนหัวใจใน AppBar
      home: const HomePage(),
    );
  }
}
```

### ขั้นตอนที่ 2.4: เขียน ItemCard ใหม่ให้ดึง Provider เอง (ลบ Prop Drilling ชั้นล่าง)

แทนที่เนื้อหาทั้งหมดใน `lib/widgets/item_card.dart` ด้วยเวอร์ชันนี้ สังเกตว่าพารามิเตอร์ `savedItems` และ `onSave` หายไปจาก constructor ทั้งคู่

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../models/item.dart';
import '../models/favorites_model.dart';

class ItemCard extends StatelessWidget {
  final Item item; // เหลือแค่พารามิเตอร์เดียว ไม่ต้องรับ savedItems/onSave อีกต่อไป

  const ItemCard({super.key, required this.item});

  @override
  Widget build(BuildContext context) {
    // .watch ที่นี่เพื่อให้ปุ่มอัปเดตสถานะ "บันทึกแล้ว" ทันทีที่ FavoritesModel เปลี่ยนจากจุดใดก็ตาม
    final favorites = context.watch<FavoritesModel>();
    final alreadySaved = favorites.items.any((i) => i.id == item.id);

    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      child: ListTile(
        title: Text(item.title),
        subtitle: Text('฿${item.price.toStringAsFixed(0)}'),
        trailing: ElevatedButton(
          onPressed: alreadySaved
              ? null
              : () {
                  // .read ที่นี่เพราะเป็นคำสั่งครั้งเดียวตอนกด ไม่ต้องการสมัครรับการอัปเดตซ้ำ
                  context.read<FavoritesModel>().add(item);
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('บันทึก ${item.title} ไว้ในรายการโปรดแล้ว')),
                  );
                },
          child: Text(alreadySaved ? '❤️ บันทึกแล้ว' : '🤍 บันทึกเป็นรายการโปรด'),
        ),
      ),
    );
  }
}
```

### ขั้นตอนที่ 2.5: เขียน ItemListSection ใหม่ (ลบ Prop Drilling ชั้นกลาง)

แทนที่เนื้อหาทั้งหมดใน `lib/widgets/item_list_section.dart` — ตอนนี้เหลือแค่พารามิเตอร์เดียวคือ `catalog` เพราะ `ItemCard` แต่ละใบไปดึง `FavoritesModel` เองโดยตรงแล้ว ไม่ต้องพึ่ง Widget แม่ส่งต่อให้อีก

```dart
import 'package:flutter/material.dart';
import '../models/item.dart';
import 'item_card.dart';

class ItemListSection extends StatelessWidget {
  final List<Item> catalog; // เหลือพารามิเตอร์เดียว เพราะ ItemCard ไปดึง FavoritesModel เอง

  const ItemListSection({super.key, required this.catalog});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      shrinkWrap: true,
      physics: const NeverScrollableScrollPhysics(),
      itemCount: catalog.length,
      // สังเกตว่าตอนนี้ ItemListSection ไม่ต้องรู้จัก FavoritesModel เลยด้วยซ้ำ
      itemBuilder: (context, index) => ItemCard(item: catalog[index]),
    );
  }
}
```

### ขั้นตอนที่ 2.6: สร้างหน้า FavoritesPage แยก Route

สร้างไฟล์ `lib/favorites_page.dart` เป็นหน้าจอใหม่ที่จะถูก push แยกออกไปจาก `HomePage` ให้แสดงรายการสินค้าที่บันทึกไว้พร้อมมูลค่ารวม โดยดึงข้อมูลจาก `context.watch<FavoritesModel>()` **ห้ามส่งข้อมูลรายการโปรดผ่าน constructor ของหน้านี้โดยเด็ดขาด** — นี่คือจุดที่พิสูจน์ว่า Provider แก้ปัญหาข้าม Route ได้จริง

⚠️ **ต้องสร้างไฟล์นี้ก่อนขั้นตอนที่ 2.7 เสมอ** เพราะ `home_page.dart` เวอร์ชันถัดไปจะ `import 'favorites_page.dart'` และเรียกใช้ `FavoritesPage()` ตรง ๆ ถ้าสร้าง `home_page.dart` ก่อนไฟล์นี้จะยังไม่มีอยู่จริง แอปจะไม่รันได้เลย

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'models/favorites_model.dart';

class FavoritesPage extends StatelessWidget {
  const FavoritesPage({super.key});

  @override
  Widget build(BuildContext context) {
    // .watch เพราะหน้านี้ต้อง rebuild ทุกครั้งที่รายการโปรดเปลี่ยน (เช่น กดลบจากหน้านี้เอง)
    final favorites = context.watch<FavoritesModel>();

    return Scaffold(
      appBar: AppBar(title: const Text('รายการโปรดของฉัน')),
      body: favorites.items.isEmpty
          ? const Center(child: Text('ยังไม่มีสินค้าที่บันทึกไว้'))
          : ListView.builder(
              itemCount: favorites.items.length,
              itemBuilder: (context, index) {
                final item = favorites.items[index];
                return ListTile(
                  title: Text(item.title),
                  subtitle: Text('฿${item.price.toStringAsFixed(0)}'),
                  trailing: IconButton(
                    icon: const Icon(Icons.delete_outline),
                    // .read เพราะเป็นการกดปุ่มครั้งเดียว ไม่ใช่การอ่านค่าต่อเนื่องแบบ .watch
                    onPressed: () => context.read<FavoritesModel>().remove(item),
                  ),
                );
              },
            ),
      bottomNavigationBar: Padding(
        padding: const EdgeInsets.all(12),
        child: Text('มูลค่ารวม: ฿${favorites.totalValue.toStringAsFixed(0)}'),
      ),
    );
  }
}
```

### ขั้นตอนที่ 2.7: เขียน HomePage ใหม่ (จาก StatefulWidget กลับมาเป็น StatelessWidget)

แทนที่เนื้อหาทั้งหมดใน `lib/home_page.dart` — จุดที่น่าสังเกตที่สุดคือ `HomePage` ไม่ต้องเก็บ State อะไรไว้เองอีกแล้ว จึงเปลี่ยนกลับจาก `StatefulWidget` เป็น `StatelessWidget` ธรรมดาได้ (ไฟล์นี้ import `favorites_page.dart` ที่สร้างไว้ในขั้นตอนที่ 2.6 ดังนั้นต้องทำขั้นตอนที่ 2.6 ให้เสร็จก่อนเสมอ)

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'models/item.dart';
import 'models/favorites_model.dart';
import 'widgets/item_list_section.dart';
import 'favorites_page.dart';

class HomePage extends StatelessWidget {
  // เปลี่ยนจาก StatefulWidget เป็น StatelessWidget ได้เลย เพราะไม่ต้องเก็บ State ใด ๆ ไว้เองอีกแล้ว
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Campus Marketplace'),
        actions: [
          IconButton(
            icon: Row(
              mainAxisSize: MainAxisSize.min,
              children: [
                const Icon(Icons.favorite),
                // .watch ทำให้ตัวเลขนี้อัปเดตเองทุกครั้งที่ FavoritesModel เปลี่ยน ไม่ว่าจะเปลี่ยนจากจุดไหน
                Text(' ${context.watch<FavoritesModel>().itemCount}'),
              ],
            ),
            onPressed: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const FavoritesPage()),
            ),
          ),
        ],
      ),
      body: ItemListSection(catalog: catalog),
    );
  }
}
```

> ✅ **Checkpoint 2.1** รันแอปใหม่ ทดสอบกดบันทึกสินค้าจากหลายจุด แล้วตรวจว่าตัวเลขที่ AppBar อัปเดตถูกต้องทุกครั้ง โดยที่ไฟล์ `item_list_section.dart` และ `item_card.dart` **ไม่มีพารามิเตอร์ savedItems หรือ onSave หลงเหลือใน constructor แล้ว**

> ✅ **Checkpoint 2.2** ทดสอบว่าเมื่อบันทึกสินค้าจากหน้า Home แล้วกดไปหน้า Favorites ตัวเลขและรายการสินค้าตรงกันทันที ลองกดปุ่มถังขยะลบสินค้าออกจากหน้า Favorites แล้วย้อนกลับไปหน้า Home ดูว่าปุ่มของสินค้านั้นกลับมากดซ้ำได้อีกครั้ง ถ่ายภาพหน้าจอทั้งสองหน้าเทียบกันแนบส่ง

```image

```

---

## ส่วนที่ 3: ใช้ AI ช่วยเลือกแนวทาง State Management

### ขั้นตอนที่ 3.1

เปิด Google AI Studio (https://aistudio.google.com) แล้วสร้าง Prompt นี้กับ Gemini

```
ฉันกำลังพัฒนาแอป Flutter ตลาดนัดออนไลน์ (Campus Marketplace) ที่มีฟีเจอร์ต่อไปนี้:
1. Dark Mode / Light Mode ที่ต้องส่งผลต่อทุกหน้าจอในแอป
2. ตัวนับ "มีคนถูกใจแล้วกี่คน" ของประกาศขายสินค้า ที่ต้องซิงค์ระหว่างหน้ารายการประกาศกับหน้ารายละเอียดสินค้า
3. Animation กระพริบของไอคอนหัวใจตอนกดถูกใจ ที่ใช้เฉพาะใน widget เดียว

ช่วยวิเคราะห์ว่าแต่ละฟีเจอร์ควรใช้ setState, Provider หรือ Riverpod
และอธิบายเหตุผลของแต่ละข้อสั้น ๆ
```

บันทึกคำตอบที่ได้จาก Gemini

```text
สรุปสั้น: 2 ข้อแรกเป็น App State ใช้ Provider / ข้อ 3 เป็น Ephemeral State ใช้ setState
ไม่มีข้อไหนใน 3 ข้อนี้ที่ "จำเป็นต้อง" ใช้ Riverpod สำหรับโปรเจกต์ขนาดนี้

------------------------------------------------------------
1. Dark Mode / Light Mode  ->  Provider (App State)
------------------------------------------------------------
ธีมถูกอ่านโดยทุกหน้าจอพร้อมกัน และตัวที่ต้องรู้ค่าจริง ๆ คือ MaterialApp ซึ่งอยู่สูงกว่าทุก Route
จึงต้องเก็บ state ไว้ "เหนือ" MaterialApp แล้วให้ MaterialApp watch ค่านั้น

  class ThemeModel extends ChangeNotifier {
    ThemeMode _mode = ThemeMode.system;
    ThemeMode get mode => _mode;
    void toggle() {
      _mode = _mode == ThemeMode.dark ? ThemeMode.light : ThemeMode.dark;
      notifyListeners();
    }
  }
  // main.dart: MaterialApp(themeMode: context.watch<ThemeModel>().mode, ...)

setState ทำไม่ได้เพราะปุ่มสลับธีมมักอยู่ในหน้า Settings ซึ่งเป็นคนละ Route กับหน้าที่ต้องเปลี่ยนสี
และ setState สั่ง rebuild ได้แค่ทรีใต้ตัวเองเท่านั้น จึงไปสั่ง MaterialApp ที่อยู่เหนือขึ้นไปไม่ได้

ข้อควรระวัง: การ "จำธีมไว้หลังปิดแอป" เป็นคนละเรื่องกับ State Management — ต้องใช้
SharedPreferences/storage เพิ่ม โดย State Management ทำหน้าที่แค่กระจายค่าในหน่วยความจำ

------------------------------------------------------------
2. ตัวนับ "มีคนถูกใจแล้วกี่คน"  ->  Provider (App State)
------------------------------------------------------------
หน้ารายการประกาศกับหน้ารายละเอียดเป็นคนละ Route กัน ถ้าส่งตัวเลขผ่าน constructor ตอน push
หน้ารายละเอียดจะได้ "ภาพนิ่ง ณ วินาทีที่เปิด" พอกดถูกใจในหน้ารายละเอียด หน้ารายการที่ค้างอยู่
ข้างหลังจะไม่รู้เรื่องและแสดงเลขเก่า — นี่คืออาการ state desync ที่ Provider แก้ด้วยการมี
Single Source of Truth ก้อนเดียวที่ทั้งสองหน้า watch อยู่

จุดที่ต่างจาก FavoritesModel ในใบงาน: รายการโปรดเป็น "ลิสต์ก้อนเดียวของทั้งแอป" แต่ตัวนับนี้เป็น
"ค่าต่อประกาศแต่ละชิ้น" จึงควรเก็บเป็น Map ที่ key ด้วย id ไม่ใช่ตัวแปรเดี่ยว

  class LikeModel extends ChangeNotifier {
    final Map<String, int> _likes = {};
    int likesOf(String id) => _likes[id] ?? 0;
    void like(String id) { _likes[id] = likesOf(id) + 1; notifyListeners(); }
  }

ข้อจำกัดที่ต้องรู้: ChangeNotifier แจ้งเตือนแบบ "ทั้งก้อน" ทุก Widget ที่ watch LikeModel จะ rebuild
ทั้งหมดแม้ถูกใจแค่ประกาศเดียว ถ้าในหน้ารายการมีการ์ดหลายร้อยใบจะเริ่มกระตุก ทางแก้คือใช้ Selector
หรือ Consumer ล้อมเฉพาะตัวเลข ไม่ใช่ watch ทั้งการ์ด

------------------------------------------------------------
3. Animation หัวใจกระพริบ  ->  setState (Ephemeral State)
------------------------------------------------------------
ค่าความคืบหน้าของ animation มีความหมายเฉพาะใน widget ใบนั้นใบเดียว ไม่มีหน้าจออื่นต้องรู้ว่า
หัวใจกำลังกระพริบถึงเฟรมไหน และค่านี้ควรตายไปพร้อม widget จึงเข้านิยาม Ephemeral State เต็มตัว

  class _HeartIconState extends State<HeartIcon> with SingleTickerProviderStateMixin {
    late final AnimationController _c = AnimationController(
      vsync: this, duration: const Duration(milliseconds: 300));
    ...
  }

เหตุผลด้านประสิทธิภาพที่สำคัญกว่าเรื่องขอบเขตเสียอีก: AnimationController เปลี่ยนค่าทุกเฟรม
(~60 ครั้ง/วินาที) ถ้าเอาไปใส่ ChangeNotifier จะกลายเป็น notifyListeners() 60 ครั้งต่อวินาที
ลากทุก Widget ที่ watch อยู่ให้ rebuild ตามไปด้วย ซึ่งเป็นการใช้ผิดประเภทอย่างชัดเจน
(วิธีที่ถูกคือใช้ AnimatedBuilder/AnimatedWidget เพื่อจำกัดขอบเขต rebuild ให้แคบที่สุด)

------------------------------------------------------------
แล้วเมื่อไหร่ถึงควรใช้ Riverpod
------------------------------------------------------------
ทั้ง 3 ข้อนี้ยังไม่จำเป็นต้องใช้ Riverpod เพราะ Provider แก้ปัญหาได้ครบแล้ว และหลักการคือเลือก
เครื่องมือที่เบาที่สุดที่เพียงพอ Riverpod จะเริ่มคุ้มเมื่อเจอสถานการณ์เหล่านี้
- ต้องเขียน Unit Test ของ logic โดยไม่อยากสร้าง Widget Tree ขึ้นมาก่อน (Riverpod อ่าน provider ได้
  โดยไม่ต้องมี BuildContext)
- state มีการพึ่งพากันเป็นลูกโซ่ เช่น ตัวนับถูกใจต้องดึงจาก API ตาม user ที่ล็อกอินอยู่
  (FutureProvider/AsyncNotifier + ref.watch provider อื่นต่อกันได้เลย)
- อยากให้คอมไพเลอร์จับตอนเขียนโค้ด แทนที่จะพังตอนรันด้วย ProviderNotFoundException
โดยเฉพาะข้อ 2 ถ้าในโปรเจกต์จริงตัวนับถูกใจมาจากเซิร์ฟเวอร์ (async + loading + error) นั่นคือจุดที่
Riverpod เริ่มได้เปรียบ Provider อย่างเห็นได้ชัด
```

### ขั้นตอนที่ 3.2: ประเมินคำตอบของ AI

เปรียบเทียบคำตอบของ Gemini กับกรอบการตัดสินใจในบทหนังสือเรียนหัวข้อ 5.7 แล้วตอบคำถามต่อไปนี้

- Gemini แนะนำตรงกับกรอบการตัดสินใจในบทเรียนหรือไม่ มีจุดใดที่ต่างกัน


```text
ข้อสรุปตรงกันทั้ง 3 ข้อ แต่ "วิธีให้เหตุผล" ไม่ตรงกันเสียทีเดียว

ส่วนที่ตรงกับกรอบในหัวข้อ 5.7
- ข้อ 1 และ 2 ตอบว่าเป็น App State ใช้ Provider ตรงกับบทเรียน และให้เหตุผลตรงจุดเดียวกับที่บทเรียน
  เน้น คือ setState สั่ง rebuild ได้แค่ทรีใต้ตัวเอง จึงข้าม Route ไม่ได้
- ข้อ 3 ตอบว่าเป็น Ephemeral State ใช้ setState ตรงกับบทเรียน
- ไม่ได้เชียร์ Riverpod แบบเหมารวม แต่ระบุเงื่อนไขว่าเมื่อไหร่ถึงคุ้มที่จะย้าย ซึ่งตรงกับหลัก
  "เริ่มจากเครื่องมือที่เบาที่สุดที่เพียงพอ" ของบทเรียน

จุดที่ต่างออกไป (AI พูดเกินบทเรียน — และเป็นประโยชน์)
- บทเรียนใช้ "ขอบเขตการมองเห็นของข้อมูล" เป็นเกณฑ์หลักเกณฑ์เดียว แต่ AI เพิ่มเกณฑ์ที่สองเข้ามาใน
  ข้อ 3 คือ "ความถี่ในการเปลี่ยนค่า" (animation เปลี่ยน 60 ครั้ง/วินาที) ซึ่งบทเรียนไม่ได้พูดถึงเลย
  ประเด็นนี้สำคัญจริง เพราะต่อให้ animation ต้องใช้ข้ามหน้าจอ ก็ยังไม่ควรยัดลง ChangeNotifier อยู่ดี
- AI ชี้ว่าข้อ 2 มีรูปร่างต่างจาก FavoritesModel ในใบงาน เพราะเป็นค่า "ต่อประกาศแต่ละชิ้น" ต้องเก็บ
  เป็น Map ที่ key ด้วย id บทเรียนไม่ได้ครอบคลุมเคสนี้ ทำให้ถ้าลอกโครง FavoritesModel ไปตรง ๆ จะผิด
- AI เตือนเรื่อง ChangeNotifier แจ้งเตือนแบบทั้งก้อน ทำให้การ์ดทุกใบ rebuild และแนะนำ Selector
  ซึ่งเป็นข้อจำกัดของ Provider ที่บทเรียนสัปดาห์นี้ยังไม่ได้สอน
- AI แยกให้ชัดว่า "การจำธีมไว้หลังปิดแอป" ไม่ใช่งานของ State Management แต่เป็นงานของ storage
  เป็นการกันความเข้าใจผิดที่บทเรียนไม่ได้ระบุไว้

จุดที่ AI ยังตอบไม่ครบ ถ้าไม่ถามต่อ
- ไม่ได้เขียนเกณฑ์ตัดสินออกมาเป็นประโยคเดียวชัด ๆ ตั้งแต่ต้น ว่า "ดูขอบเขตการมองเห็นของข้อมูล
  ก่อนเสมอ" แต่ไปไล่ตอบทีละฟีเจอร์ ผู้เรียนที่ยังไม่แม่นอาจจำเป็น "ท่ามาตรฐานของแต่ละฟีเจอร์"
  แทนที่จะจับหลักการได้
- ไม่ได้ยกตัวอย่างกรณีที่ฟีเจอร์เดิมเปลี่ยนคำตอบเมื่อขอบเขตเปลี่ยน ซึ่งเป็นบททดสอบว่าเข้าใจหลักจริง
  จึงถามต่อในช่องถัดไป

ข้อสรุปของผู้เรียนเอง: รับคำตอบทั้ง 3 ข้อ เพราะตรวจแล้วสอดคล้องกับเกณฑ์ "ขอบเขตการมองเห็นของ
ข้อมูล" ในหัวข้อ 5.7 ด้วยตัวเอง ไม่ได้รับเพราะ AI บอก และรับข้อเสริมเรื่องความถี่การเปลี่ยนค่า
เพิ่มเข้ามาเป็นเกณฑ์ที่สอง เพราะตรวจสอบแล้วว่าอธิบายได้ด้วยเหตุผลเชิงประสิทธิภาพที่จับต้องได้
```
- หากคำตอบของ AI ดูสมเหตุสมผลแต่ยังไม่ครบถ้วน (เช่น ไม่ได้พูดถึงขอบเขตของ Widget) ให้ลองถามคำถามต่อเพื่อขอเหตุผลเพิ่มเติม แล้วบันทึกบทสนทนาไว้ด้วย
```text
[ถาม] ข้อ 3 บอกว่า animation หัวใจเป็น Ephemeral State เพราะอยู่ใน widget เดียว
      แล้วถ้าโจทย์เปลี่ยนเป็น "กดถูกใจหนึ่งครั้ง แล้วหัวใจต้องกระพริบพร้อมกันทุกใบในหน้าจอ
      รวมถึงไอคอนบน AppBar ด้วย" คำตอบยังเป็น setState เหมือนเดิมไหม

[ตอบ] คำตอบเปลี่ยน แต่ไม่ได้เปลี่ยนเป็น "ยัด AnimationController ลง Provider"
      ต้องแยกให้ออกว่ามีข้อมูลสองก้อนคนละชนิดกันอยู่ในฟีเจอร์เดียว
      - "เพิ่งมีการกดถูกใจเกิดขึ้น" = App State (ทุกใบต้องรู้)  -> Provider
      - "ตอนนี้หัวใจใบนี้ขยายอยู่กี่เปอร์เซ็นต์" = Ephemeral State (แต่ละใบคำนวณของตัวเอง)
        -> AnimationController ใน StatefulWidget ของแต่ละใบเหมือนเดิม
      วิธีทำคือแต่ละใบ watch สัญญาณจาก Provider แล้วใน didChangeDependencies สั่ง _c.forward(from: 0)
      ของตัวเอง ผลคือสิ่งที่วิ่งข้ามหน้าจอคือ "เหตุการณ์" ไม่ใช่ "ค่าทุกเฟรม"
      บทเรียนสรุปว่า "animation = ephemeral" ซึ่งถูกในกรณีทั่วไป แต่เกณฑ์ที่แม่นกว่าคือถามว่า
      ข้อมูลก้อนไหนกันแน่ที่ต้องข้ามขอบเขต แล้วยกขึ้นไปเฉพาะก้อนนั้น

[ถาม] แปลว่าเกณฑ์ตัดสินจริง ๆ ไม่ใช่ "ฟีเจอร์นี้คืออะไร" ใช่ไหม

[ตอบ] ใช่ และนี่คือจุดที่คำตอบรอบแรกยังไม่ได้พูดให้ชัด เกณฑ์คือ "ข้อมูลชิ้นนี้มีใครต้องเห็นบ้าง"
      ไม่ใช่ "ฟีเจอร์นี้ซับซ้อนแค่ไหน" ฟีเจอร์เดียวกันอาจตอบต่างกันได้ถ้าขอบเขตต่างกัน
      และฟีเจอร์ที่โค้ดยาวกว่าอาจใช้เครื่องมือที่เบากว่าก็ได้ ลำดับคำถามที่ควรถามตัวเองคือ
      1) มี Widget นอกกิ่งนี้หรือนอก Route นี้ต้องอ่านค่านี้ไหม ถ้าไม่มี -> setState จบ
      2) ถ้ามี ค่านี้เปลี่ยนถี่แค่ไหน ถ้าถี่ระดับทุกเฟรม -> ส่ง "เหตุการณ์" ข้ามไป ไม่ใช่ส่ง "ค่า"
      3) ถ้าข้ามขอบเขตจริงและไม่ถี่ -> Provider
      4) ถ้าเริ่มมี async / provider พึ่งกันเป็นลูกโซ่ / ต้องเทสต์โดยไม่มี Widget Tree -> Riverpod

[ถาม] ข้อ 2 ที่บอกว่าต้องใช้ Map แทน List — ถ้าลอกโครง FavoritesModel จากใบงานไปตรง ๆ
      จะพังตรงไหน

[ตอบ] FavoritesModel เก็บ List<Item> คือ "ไอเทมนี้ถูกบันทึกหรือไม่" ซึ่งเป็นค่า boolean ต่อไอเทม
      แต่ตัวนับถูกใจเป็นจำนวนเต็มต่อไอเทม ถ้าใช้ List จะนับซ้ำได้แต่หาจำนวนต้องวน .where().length
      ทุกครั้งที่ rebuild ซึ่งเป็น O(n) ต่อการ์ดหนึ่งใบ พอมีการ์ด n ใบก็กลายเป็น O(n^2) ต่อหนึ่งเฟรม
      ใช้ Map<String,int> อ่านเป็น O(1) และสื่อความหมายตรงกว่า

[สรุปของผู้เรียน] การถามต่อทำให้ได้เกณฑ์ที่ใช้ตัดสินเองได้จริง (ไล่ 4 ข้อข้างบน) แทนที่จะจำแค่
ว่าฟีเจอร์ไหนใช้อะไร และได้เห็นด้วยว่าคำตอบรอบแรกของ AI ถึงจะถูก แต่ยังไม่ได้ให้ "หลัก" มาให้
ต้องขุดเอาเองด้วยการตั้งคำถามที่ท้าทายคำตอบเดิม
```

⚠️ **ข้อควรระวัง**: AI เป็นเครื่องมือช่วยคิด ไม่ใช่คำตอบสุดท้าย ผู้เรียนต้องอธิบายเหตุผลของการเลือกใช้เครื่องมือได้ด้วยตัวเองเสมอ ตามหลักการใช้ AI ในการพัฒนาซอฟต์แวร์ของวิชานี้

---

## ส่วนที่ 4 (เพิ่มเติม/ทดลอง): แปลง FavoritesModel เป็น Riverpod

ส่วนนี้เป็นแบบฝึกหัดเสริมเพื่อให้เห็นความแตกต่างของโค้ดจริงระหว่าง Provider และ Riverpod **ไม่บังคับเปลี่ยนโปรเจกต์หลัก** `campus_marketplace` แต่ให้สร้าง**โปรเจกต์ทดลองใหม่แยกต่างหากทั้งหมด** เพื่อเทียบโค้ดแบบเดียวกันที่เขียนด้วย Provider (ส่วนที่ 2) กับ Riverpod แบบเคียงข้างกัน

**ภาพรวมก่อนเริ่ม**: โปรเจกต์ทดลองนี้จะมีแค่ 3 ไฟล์เท่านั้น ต่างจากโปรเจกต์หลักที่แยกเป็นหลายไฟล์/โฟลเดอร์ (ไม่มี `ItemCard`, `ItemListSection`, `FavoritesPage` แยก) เพราะจุดประสงค์คือเทียบไวยากรณ์ให้เห็นชัด ไม่ใช่สร้างแอปสมบูรณ์อีกรอบ

```
campus_marketplace_riverpod_trial/
└── lib/
    ├── item.dart               ← โมเดลข้อมูล เหมือนขั้นตอนที่ 1.1
    ├── favorites_notifier.dart ← เทียบเท่า FavoritesModel แต่เขียนแบบ Riverpod
    └── main.dart                ← รวม MyApp + HomePage ไว้ในไฟล์เดียว เพื่อความกระชับ
```

### ขั้นตอนที่ 4.1: สร้างโปรเจกต์ใหม่

เปิด Terminal แล้วรันคำสั่งนี้ **นอกโฟลเดอร์ `campus_marketplace` เดิม** (อย่าสร้างโปรเจกต์ซ้อนในโปรเจกต์)

```bash
flutter create campus_marketplace_riverpod_trial
```

`cd` เข้าไปในโฟลเดอร์ที่เพิ่งสร้าง แล้วเปิดด้วย VS Code ผ่านเมนู **File → Open Folder...** (เลือกโฟลเดอร์ `campus_marketplace_riverpod_trial` ที่มี `pubspec.yaml` อยู่ข้างใน) จากนั้นเปิดไฟล์ `lib/main.dart` ที่ Flutter สร้างให้อัตโนมัติ แล้ว **ลบโค้ด Counter Demo เริ่มต้นทั้งหมดทิ้งให้เหลือไฟล์ว่าง** เราจะเขียนเนื้อหาใหม่ทั้งไฟล์ในขั้นตอนที่ 4.5

### ขั้นตอนที่ 4.2: ติดตั้งแพ็กเกจ

เปิดไฟล์ `pubspec.yaml` (อยู่ที่ root ของโปรเจกต์ทดลองนี้ คนละไฟล์กับ `pubspec.yaml` ของ `campus_marketplace`) แล้วเพิ่มบรรทัดนี้ในส่วน `dependencies:`

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.5.1
```

รันคำสั่งในเทอร์มินัล (ต้องอยู่ในโฟลเดอร์ `campus_marketplace_riverpod_trial` ตามหลักการเดียวกับที่อธิบายไว้ในหัวข้อ Troubleshooting เรื่อง `pubspec.yaml`)

```bash
flutter pub get
```

### ขั้นตอนที่ 4.3: สร้าง Item Model

เนื่องจากนี่คือโปรเจกต์ Flutter ใหม่แยกต่างหาก จึงยังไม่มีคลาส `Item` หรือ `catalog` อยู่เลย (ไม่ได้สืบทอดไฟล์จากโปรเจกต์หลักให้อัตโนมัติ) ให้สร้างไฟล์ใหม่ชื่อ `lib/item.dart` แล้วใส่เนื้อหาเดียวกับที่สร้างไว้ในขั้นตอนที่ 1.1 ของโปรเจกต์หลัก (วางไว้ที่ `lib/item.dart` ตรง ๆ ไม่ต้องมีโฟลเดอร์ `models/` ซ้อนอีกชั้น เพราะโปรเจกต์ทดลองนี้ตั้งใจให้มีโครงสร้างเรียบง่ายที่สุด)

```dart
class Item {
  final String id;
  final String title;
  final double price;

  const Item({required this.id, required this.title, required this.price});
}

final catalog = <Item>[
  const Item(id: 'i1', title: 'หนังสือ Calculus มือสอง', price: 150),
  const Item(id: 'i2', title: 'หูฟังไร้สาย (สภาพดี 90%)', price: 450),
  const Item(id: 'i3', title: 'โคมไฟตั้งโต๊ะหอพัก', price: 120),
];
```

### ขั้นตอนที่ 4.4: สร้าง FavoritesNotifier (เทียบเท่า FavoritesModel)

สร้างไฟล์ใหม่ชื่อ `lib/favorites_notifier.dart` — คลาสนี้ทำหน้าที่เดียวกับ `FavoritesModel` ในส่วนที่ 2 ทุกเมธอด (add, remove, totalValue) เพียงแต่เปลี่ยนวิธีเก็บ State จากตัวแปร mutable ภายในคลาสมาเป็นการ "แทนที่ state ก้อนใหม่" ทั้งหมดทุกครั้งที่แก้ไข ตารางนี้ช่วยให้เห็นว่ากำลังแปลงอะไรเป็นอะไร

| ในโปรเจกต์หลัก (Provider) | ในโปรเจกต์ทดลองนี้ (Riverpod) |
|---|---|
| `class FavoritesModel extends ChangeNotifier` | `class FavoritesNotifier extends StateNotifier<List<Item>>` |
| `final List<Item> _items = []` + `notifyListeners()` | `state = [...state, item]` (แทนที่ก้อนใหม่ทั้งหมด) |
| ลงทะเบียนด้วย `ChangeNotifierProvider` | ลงทะเบียนด้วย `StateNotifierProvider` |

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'item.dart';

class FavoritesNotifier extends StateNotifier<List<Item>> {
  FavoritesNotifier() : super([]); // ค่าเริ่มต้นคือลิสต์ว่าง เทียบเท่า _items = [] ใน ChangeNotifier

  // ใช้ spread operator [...state, item] สร้างลิสต์ใหม่ทั้งก้อน แทนการ mutate ลิสต์เดิม
  void add(Item item) => state = [...state, item];

  // เช่นเดียวกัน ใช้ .where() สร้างลิสต์ใหม่ที่ไม่มีไอเทมนี้อยู่ แทนการ remove ตรง ๆ
  void remove(Item item) => state = state.where((i) => i.id != item.id).toList();

  double get totalValue => state.fold(0, (sum, i) => sum + i.price);
}

// ประกาศ Provider เป็นตัวแปร global เทียบเท่ากับการลงทะเบียน ChangeNotifierProvider ใน main.dart
final favoritesProvider = StateNotifierProvider<FavoritesNotifier, List<Item>>(
  (ref) => FavoritesNotifier(),
);
```

### ขั้นตอนที่ 4.5: เขียน main.dart ใหม่ทั้งไฟล์

ต่างจากโปรเจกต์หลักที่แยก `HomePage` ไว้คนละไฟล์ ในโปรเจกต์ทดลองนี้ให้รวม `MyApp` และ `HomePage` ไว้ใน `main.dart` ไฟล์เดียวเพื่อความกระชับ เปิดไฟล์ `lib/main.dart` ที่ลบโค้ดเดิมทิ้งไว้แล้วตั้งแต่ขั้นตอนที่ 4.1 แล้วพิมพ์โค้ดนี้ลงไปทั้งหมด

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'item.dart';
import 'favorites_notifier.dart';

void main() {
  // ครอบแอปทั้งหมดด้วย ProviderScope เพียงครั้งเดียวที่จุดเริ่มต้น เทียบเท่า ChangeNotifierProvider
  runApp(const ProviderScope(child: MyApp()));
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  @override
  Widget build(BuildContext context) => const MaterialApp(
        debugShowCheckedModeBanner: false, // ปิดริบบิ้น DEBUG มุมขวาบน ไม่ให้บังไอคอนหัวใจใน AppBar
        home: HomePage(),
      );
}

// ใช้ ConsumerWidget แทน StatelessWidget เพื่อรับพารามิเตอร์ "ref" เข้ามาใน build()
class HomePage extends ConsumerWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ref.watch อ่านค่าปัจจุบันและสมัครรับการอัปเดตอัตโนมัติ เทียบเท่า context.watch
    final savedItems = ref.watch(favoritesProvider);

    return Scaffold(
      appBar: AppBar(title: Text('❤️ ${savedItems.length}')),
      body: ListView(
        children: catalog.map((item) => ListTile(
          title: Text(item.title),
          trailing: ElevatedButton(
            // ref.read(...notifier) ใช้เรียกแก้ไขค่า เทียบเท่า context.read
            onPressed: () => ref.read(favoritesProvider.notifier).add(item),
            child: const Text('บันทึก'),
          ),
        )).toList(),
      ),
    );
  }
}
```

> ✅ **Checkpoint 4.1** รันแอปด้วย `flutter run` (หรือกด F5 ใน VS Code) แล้วทดสอบกดปุ่ม "บันทึก" ที่สินค้าชิ้นใดก็ได้ ตรวจว่าตัวเลข ❤️ ที่ AppBar เพิ่มขึ้นถูกต้อง ถ่ายภาพหน้าจอแนบส่ง

> ✅ **Checkpoint 4.2** เขียนตารางเปรียบเทียบสั้น ๆ ว่าตอนแปลงจาก Provider เป็น Riverpod ต้องเปลี่ยนอะไรบ้าง (เช่น `ChangeNotifier` → `StateNotifier`, `StatelessWidget` → `ConsumerWidget`, `context.watch` → `ref.watch`) อย่างน้อย 4 คู่เทียบ

| # | ประเด็น | Provider (โปรเจกต์หลัก) | Riverpod (โปรเจกต์ทดลอง) |
|---|---|---|---|
| 1 | คลาสเก็บ State | `class FavoritesModel extends ChangeNotifier` | `class FavoritesNotifier extends StateNotifier<List<Item>>` |
| 2 | วิธีแก้ข้อมูล | `_items.add(item);` แล้วตามด้วย `notifyListeners()` | `state = [...state, item];` (สร้างลิสต์ใหม่ทั้งก้อน แจ้งเตือนอัตโนมัติ) |
| 3 | ที่เก็บ/ลงทะเบียน Provider | `ChangeNotifierProvider(create: ...)` ครอบใน `main()` | `final favoritesProvider = StateNotifierProvider(...)` เป็นตัวแปร global + `ProviderScope` ครอบใน `main()` |
| 4 | คลาสฐานของ Widget | `StatelessWidget` (build รับ `BuildContext` อย่างเดียว) | `ConsumerWidget` (build รับ `BuildContext` + `WidgetRef`) |
| 5 | อ่านค่าแบบติดตาม | `context.watch<FavoritesModel>()` | `ref.watch(favoritesProvider)` |
| 6 | สั่งแก้ค่าครั้งเดียว | `context.read<FavoritesModel>().add(item)` | `ref.read(favoritesProvider.notifier).add(item)` |
| 7 | ความปลอดภัยของ Type | หา Provider ไม่เจอ = พังตอนรัน (`ProviderNotFoundException`) | ผูก Type ตั้งแต่ประกาศตัวแปร = คอมไพเลอร์จับได้ตั้งแต่ตอนเขียน |
| 8 | ถ้าต้องมี Ephemeral State ด้วย | `StatefulWidget` + `context.watch` ใน build | `ConsumerStatefulWidget` / `ConsumerState` (ใช้ `ref` ได้ทั้งคลาส) |

**ข้อสังเกตที่ได้จากการแปลงจริง**: แนวคิดไม่เปลี่ยนเลย (ยังเป็น Single Source of Truth + ผู้ติดตามที่ rebuild เอง) สิ่งที่เปลี่ยนคือ (ก) ช่องทางเข้าถึงข้อมูล จาก `BuildContext` เป็น `WidgetRef` ทำให้ไม่ต้องมี Widget อยู่ในทรีก็อ่านค่าได้ และ (ข) รูปแบบการแก้ข้อมูล จาก mutate-แล้วแจ้ง เป็น replace-ทั้งก้อน (immutable) ซึ่งทำให้เทียบค่าเก่า/ใหม่และเขียน Unit Test ได้ง่ายกว่า

---

## ส่วนที่ 5 (ทำด้วยตนเอง): ออกแบบฟีเจอร์เพิ่มด้วยตัวเอง

ส่วนนี้**ไม่มีโค้ดต้นแบบให้ทั้งหมด**เหมือนส่วนก่อนหน้า เพราะจุดประสงค์คือให้ผู้เรียนนำหลักการ Ephemeral State vs App State และการใช้ Provider ที่เรียนมาทั้งบท ไปประยุกต์ออกแบบและเขียนโค้ดด้วยตนเอง จำลองสถานการณ์ทำงานจริงที่ไม่มีใบสั่งงานบอกทุกขั้นตอน

### โจทย์ที่ 1: ช่องค้นหาสินค้า (Search Box)

เพิ่ม `TextField` ที่หน้า Home สำหรับพิมพ์คำค้นหา แล้วกรองรายการที่ส่งให้ `ItemListSection` ให้เหลือเฉพาะสินค้าที่ `title` มีคำค้นหาอยู่ (ไม่สนตัวพิมพ์เล็ก-ใหญ่)

**ข้อกำหนด**

- ต้องตัดสินใจเองว่าค่าคำค้นหาควรเป็น Ephemeral State หรือ App State พร้อมให้เหตุผลสั้น ๆ ไว้ในช่องด้านล่าง
  ```text
  เลือก: Ephemeral State (ใช้ setState ใน _HomePageState ไม่ใช้ Provider)

  เหตุผล
  - ขอบเขตการมองเห็น: มีแค่ body ของ HomePage หน้าเดียวที่ต้องรู้คำค้นหา ไม่มีหน้าจออื่น
    (FavoritesPage, ItemCard) ที่ต้องใช้ค่านี้ จึงไม่เข้าเกณฑ์ App State
  - อายุการใช้งาน: คำค้นหาควรหายไปเมื่อออกจากหน้านี้ ถ้ายกขึ้นไปไว้ใน Provider ระดับ root
    ค่าจะค้างอยู่ตลอดอายุแอปโดยไม่มีใครต้องการ กลายเป็นสถานะส่วนเกิน
  - หลักเครื่องมือที่เบาที่สุดที่เพียงพอ: การใส่ลง Provider จะทำให้ทุก Widget ที่ watch
    FavoritesModel/SearchModel ถูกดึงเข้ามาเกี่ยวข้องโดยไม่จำเป็น เพิ่มความซับซ้อนโดยไม่ได้อะไรคืน

  ข้อสังเกต: HomePage จึงกลับไปเป็น StatefulWidget อีกครั้ง แต่คนละเหตุผลกับส่วนที่ 1 —
  รอบนี้ State ที่เก็บเป็น Ephemeral จริง ๆ ส่วนรายการโปรด (App State) ยังอยู่ใน Provider เหมือนเดิม
  ไฟล์: campus_marketplace/lib/home_page.dart
  ```
- ถ้าตัดสินใจว่าเป็น Ephemeral State ห้ามใช้ Provider สำหรับฟีเจอร์นี้ ให้ฝึกเลือกใช้เครื่องมือที่เบาที่สุดที่เพียงพอ (`setState` ธรรมดา)

### โจทย์ที่ 2: ปุ่ม "ล้างรายการโปรดทั้งหมด"

สังเกตว่า `FavoritesModel` มีเมธอด `clear()` เตรียมไว้ให้แล้วตั้งแต่ขั้นตอนที่ 2.2 แต่ยังไม่เคยถูกเรียกใช้งานจากที่ใดเลย ให้เพิ่มปุ่มในหน้า `FavoritesPage` ที่เรียกใช้เมธอดนี้ พร้อมแสดง Dialog ยืนยันก่อนล้างข้อมูลจริง (ใช้ `showDialog` + `AlertDialog`)

**ข้อกำหนด**

- ต้องใช้ `context.read` หรือ `context.watch` ให้ถูกต้องตามหลักการ และอธิบายเหตุผลการเลือก ในช่องด้านล่าง
  ```text
  ใช้ทั้งสองอย่าง คนละหน้าที่กัน

  1) context.watch<FavoritesModel>() ใน build() ของ FavoritesPage
     เพราะต้องใช้ favorites.itemCount มาตัดสินว่าจะ "แสดงหรือซ่อนปุ่ม" (if (favorites.itemCount > 0))
     เงื่อนไขนี้ต้องถูกคำนวณใหม่ทุกครั้งที่รายการเปลี่ยน ปุ่มจึงจะหายไปเองทันทีหลังกดล้าง
     ถ้าใช้ .read ตรงนี้ Widget จะไม่สมัครรับการอัปเดต ปุ่มจะยังค้างอยู่บนจอทั้งที่รายการว่างแล้ว

  2) context.read<FavoritesModel>() ในเมธอด _confirmClear()
     เพราะเป็นการ "สั่งงานครั้งเดียว" ตอนผู้ใช้กดยืนยัน ไม่ได้ต้องการติดตามค่าต่อเนื่อง
     อีกทั้งโค้ดส่วนนี้อยู่นอก build() — การเรียก .watch นอก build เป็นสิ่งที่ห้ามทำอยู่แล้ว
     และการ .read ยังช่วยไม่ให้เกิดการสมัคร listener ซ้ำซ้อนโดยไม่จำเป็น

  หมายเหตุการเขียนโค้ด: อ่าน context.read และ ScaffoldMessenger.of(context) เก็บไว้ในตัวแปร
  ก่อนเรียก await showDialog เพราะหลัง await แล้ว context อาจไม่ valid (use_build_context_synchronously)
  ไฟล์: campus_marketplace/lib/favorites_page.dart
  ```
- ปุ่มต้องแสดงเฉพาะเมื่อมีรายการโปรดอย่างน้อย 1 รายการเท่านั้น (ถ้ารายการว่างอยู่แล้วไม่ต้องแสดงปุ่มนี้)

### โจทย์ที่ 3 (ท้าทายเพิ่ม ไม่บังคับ)

ทำโจทย์ที่ 1 และ 2 ซ้ำอีกครั้งในโปรเจกต์ทดลอง Riverpod (ส่วนที่ 4) 

> ✅ **Checkpoint 5.1** ถ่ายภาพหน้าจอฟีเจอร์ค้นหาที่กรองสินค้าได้ถูกต้อง และภาพ Dialog ยืนยันการล้างรายการโปรด เขียนอธิบายเหตุผลการเลือกชนิด State ของทั้งสองฟีเจอร์ ในช่องด้านล่าง
```text
สรุปเหตุผลการเลือกชนิด State ของทั้งสองฟีเจอร์

ฟีเจอร์ที่ 1 — ช่องค้นหา (_query) = Ephemeral State -> setState
  มีเพียง HomePage หน้าเดียวที่ต้องรู้ค่านี้ ไม่มีหน้าจออื่นหรือ Widget ต่างกิ่งต้องอ่าน
  และค่าควรหายไปพร้อมหน้าจอ จึงใช้เครื่องมือที่เบาที่สุดที่เพียงพอคือ setState
  การยกขึ้นไปเป็น Provider จะได้สถานะที่อายุยืนเกินความจำเป็นและเพิ่ม rebuild โดยเปล่าประโยชน์

ฟีเจอร์ที่ 2 — รายการโปรด (FavoritesModel) = App State -> Provider (ChangeNotifier)
  ข้อมูลชุดเดียวกันถูกใช้พร้อมกันที่ AppBar ของ HomePage, ปุ่มในทุก ItemCard และหน้า
  FavoritesPage ที่อยู่คนละ Route การกดล้างทั้งหมดที่หน้าหนึ่งต้องสะท้อนไปอีกหน้าทันที
  ซึ่ง setState ทำไม่ได้เพราะสั่ง rebuild ได้แค่ทรีใต้ตัวเอง จึงต้องใช้ Provider

ข้อสรุปเชิงหลักการ: ไม่ได้เลือกเครื่องมือตาม "ความซับซ้อนของฟีเจอร์" แต่เลือกตาม "ขอบเขตการ
มองเห็นของข้อมูล" — ฟีเจอร์ค้นหาดูซับซ้อนกว่าในแง่โค้ด แต่กลับใช้เครื่องมือที่เบากว่า
เพราะข้อมูลของมันไม่ต้องข้ามขอบเขต Widget เดียว
```
