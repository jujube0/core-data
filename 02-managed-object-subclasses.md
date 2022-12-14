
# 2. NSManagedObject Subclasses

<aside>
๐ก - subclassing of NSManagedObject to make your own classes for each data entity, creating a direct one-to-one mapping between entities in the data model editor and classes in the code.
- data types available in Core Data entities 
- validating data automatically before saving

</aside>

- data types available in Core Data entities
    - String, Boolean, UUID, URIโฆ
    - Integer 16, Integer 32, Integer 64
        - 16-bit integer: -32768 to 32767
        - 32-bit integer: -21์ต to 21djr
        - 64-bit integer: 920๊ฒฝ
    - Binary Data (image, pdf filesโฆ)
        - **Allows External Storage** ์ต์์ ์ด์ฉํด์, data๋ฅผ db์ ์ง์  ์ ์ฅํ๊ฑฐ๋ URI ๋ง์ ์ ์ฅํ๋ ๊ฒ ์ค ์ ํ ๊ฐ๋ฅํ๊ฒ ํ  ์ ์๋ค. binary data type์๋ง ๊ฐ๋ฅํ ์ต์์ผ๋ก, ์ผ์ ธ์๋ค๋ฉด ํด๋น attribute๋ฅผ ์ด์ฉํด์ ์ฟผ๋ผํ ๋์ด ๋ถ๊ฐํ๋ค.

## Storing non-standard data types in Core Data

<aside>
๐ก UIColor ์ฒ๋ผ attributes ํ์์ ์กด์ฌํ์ง ์๋ ํ์์ ์ ์ฅํ  ๋์๋ ์ด๋ป๊ฒ ํด์ํ ๊น?

</aside>

- Data๋ก ํด๋น ์ธ์คํด์ค๋ฅผ ์๋ฆฌ์ผ๋ผ์ด์ฆํ ํ binary data ํ์์ผ๋ก ์ ์ฅํ  ์ ์๋ค. ์ด ๊ฒฝ์ฐ Data๋ฅผ ๊ฐ์ ธ์ฌ ๋ ๋ค์ UIColor๋ก ๋ฐ๊ฟ์ฃผ๋ ์์์ด ํ์ํ๋ค.
    
    ```swift
    // Data ๋ก ์ง์  ๋ณํํ๊ธฐ
    let entity = NSEntityDescription.entity(forEntityName: "BowTie", in: managedContext)!
    let bowtie = BowTie(entity: entity, insertInto: managedContext)
    
    let image = UIImage(named: imageName!)
    bowtie.photoData = image?.pngData()
    ```
    
- Transformable ์ ์ด์ฉํ  ์๋ ์๋ค. ํ์๊ณผ Data๊ฐ ๋ณํํ  ๋์ ๋ํ ์ ๋ณด๋ฅผ iOS ์ ์ ๊ณตํ๋ค.
    - attribute๋ฅผ transformableํ๊ฒ ๋ง๋ค๊ธฐ ์ํด์๋ ์ธ๊ฐ์ง ์กฐ๊ฑด์ด ํ์ํ๋ค.
        1. Add `NSSecureCoding` protocol conformance to the backing data type.
        2. Create and Register an `NSSecureUnarchiveFromDataTransformer` subclass.
        3. Associate the custom data transformer subclass with the Transformable attribute in the Data Model Editor.
    - UIColor๋ฅผ ํฌํจํ, Apple framework์ ์กด์ฌํ๋ ๋๋ถ๋ถ์ data type๋ค์ ๋ชจ๋ ์ด๋ฏธ NSSecureCoding์ ๋ง์กฑํ๊ณ  ์๋ค.
    

```swift
class ColorAttributeTransformer: NSSecureUnarchiveFromDataTransformer {
  
  //1
  override class var allowedTopLevelClasses: [AnyClass] {
    [UIColor.self]
  }
  
  //2
  static func register() {
    let className = String(describing: ColorAttributeTransformer.self)
    let name = NSValueTransformerName(className)
    
    let transformer = ColorAttributeTransformer()
    ValueTransformer.setValueTransformer(transformer, forName: name)
  }
}
```

1. ํด๋น data transformer๊ฐ decode ๊ฐ๋ฅํ class list๋ฅผ ๋ฆฌํดํ๋ค. ( ์ด ๊ฒฝ์ฐ UIColor ๋ง ๋์ฝ๋ ๊ฐ๋ฅํจ)
2. `register` ๋ transformer class์ ์ด๋ฆ (key)๊ณผ transformer instance๋ฅผ key-value๋ก ๋งคํํด์ ์ ์ฅํ๋ค.
3. AppDelegate ์ `application(_:didFinishLaunchingWithOptions:)` ๋ด์์ `register` ๋ฅผ ํธ์ถํ๋ค. ์ฌ์ค application์ด core data stack์ ๊ตฌ์ฑํ๊ธฐ ์ ์ด๋ผ๋ฉด ์ด๋์๋  ํธ์ถํด๋ ์๊ด ์๋ค.

<img src="https://velog.velcdn.com/images/jujube0/post/22f35e0e-23dc-4643-9d9a-699ae47b370a/image.png">

```swift
// Transformable ์ด์ฉํ๊ธฐ -> ๊ทธ๋ฅ ๊ทธ๋๋ก ์ ์ฅํ๋ฉด ๋จ
bowtie.tintColor = UIColor(named: "red")
```

## subclassing NSManagedObject

<aside>
๐ก ์ด์ ์๋ key-value coding์ ์ด์ฉํด์ NSManagedObject์ ์ง์  ์ ๊ทผํ๋ ๋ฐฉ๋ฒ์ ์ด์ฉํ๋ค. ์ง์  key๋ฅผ ๋งค๋ฒ ์์ฑํ๊ธฐ ๋๋ฌธ์ ์๋ฌ์ ์ทจ์ฝํ๋ค๋ ๋จ์ ์ด ์๋ค.

</aside>

key-value coding ๋์ , NSManagedObject๋ฅผ ์๋ธํด๋์ฑํ  ์ ์๋ค. ํ๋์ data model์ entity๋ฅผ ์ํ ํด๋์ค(์ด ๊ฒฝ์ฐ `BowTie` ํด๋์ค)๋ฅผ ๋ง๋๋ ๊ฒ์ด๋ค!

xcdatamodeld ํ์ผ์์ entity์ inspector๋ฅผ ์ด์ฉํด์ xcode๊ฐ manually/automatically ํ๊ฒ ์๋ธํด๋์ค๋ฅผ ์์ฑํ๊ฒ ์ค์ ํ  ์ ์๋ค.

ํด๋น ์ค์ ์ BowTie entity๋ฅผ ๋ชจ๋ธ์ ์ถ๊ฐํ ํ, ์ฒซ ์ปดํ์ผ ์ ์ ์ค์ ํด์ผํ๋ค. ์ปดํ์ผ ํ์ ์ค์ ์ ํ๊ฒ ๋๋ฉด ๋ ๊ฐ์ง ๋ฒ์ ์ managed object subclass ๋ฒ์ ์ด ์์ฑ๋๋ค. ใ 

**Editor\Create NSManagedObject Subclass**๋ฅผ ์ ํํ ํ ์ํ๋ data model์ ์ ํํ์. create์ ๋๋ฅด๋ฉด ๋ ๊ฐ์ swift file์ด ์ถ๊ฐ๋๋ค. 

- BowTie+CoreDataProperties.swift
    - the properties that correspond to the BowTie attributes in your data model
- BowTie+CoreDataClass.swift
    - operations (empty first)

์ํ! custom code๋ค์ CoreDataClass์๋ง ์ถ๊ฐ๋๋ค. ์ฆ, entity๊ฐ ์์ ๋์ด๋ ์ฒซ๋ฒ์งธ ํ์ผ๋ง ์์ ๋๊ธฐ ๋๋ฌธ์ custom code๋ค์ ์์ ํ๊ฒ ์ ์ง๋๋ ๊ฒ~ 

## Data validation in core data

xcdatamodeld์์ attribute์ ๋๋ฅธ ํ validation ํ๋ ์ฑ์ฐ๋ฉด ๋จ.

managed object context์ `save` ๋ฅผ ํธ์ถํ๋ฉด context๊ฐ validation์ ์งํํจ.

validation์ ์คํจํ๋ฉด error๊ฐ ๋ฐ์

์๋ฅผ ๋ค์ด maximum 5๋ก ์ค์ ํ rating(double)์ ๋ ํฐ ๊ฐใใน ๋ฃ์ผ๋ฉด NSValidationNumberTooLargeError๊ฐ ๋ฐธ์ฑํ๋ค.
