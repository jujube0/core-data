# core-data

# 1. First Core Data App

- **on-disk persistence**
    - core data 는 on-disk persistence를 제공한다.
    - 앱을 종료한 후에 재진입하더라도 데이터가 사라지지 않는 것
    - (in memory persistence의 경우 앱이 메모리에 존재하는 경우에만 데이터를 유지시킨다.)
- **NSManagedObject**
    - Single object stored in core data

**name(String)을 받아 people([NSManagedObject])에 저장하는 코드**

```swift
func save(name: String) {
    guard let appDelegate = UIApplication.shared.delegate as? AppDelegate else {
        return
    }
    
    // 1. managed context를 가져오기
    let managedContext = appDelegate.persistentContainer.viewContext
    
    // 2. 새로운 managed object를 생성하여 manged context에 저장하기
    let entity = NSEntityDescription.entity(forEntityName: "Person", in: managedContext)!
    
    let person = NSManagedObject(entity: entity, insertInto: managedContext)
    
    // 3
    person.setValue(name, forKey: "name")
    
    // 4
    do {
        try managedContext.save()
        people.append(person)
    } catch let error as NSError {
        print("Could not save. \(error), \(error.userInfo)")
    }
}
```

1. **managed context를 가져오기**
Core Data Store에 접근하기 위해서는 일단 NSManagedObjectContext를 가져와야한다. 
    - im-memory scratchpad (a small, fast memory for the temporary storage of data)
    - core data에 새로운 managed object를 저장하기 위해서는
        1. 일단 새로운 managed object를 managed object context에 추가하고
        2. 실제 disk에 commit 하는 과정을 거쳐야한다.
    - 처음 프로젝트를 생성할 때 **Use Core Data**를 체크하면, xcode가 자동으로 AppDelegate.swift 파일에서 managed object를 생성해준다. app delegate에 접근한 후, app delegate의 NSPersistentContainer의 프로퍼티에 접근해서 위와 같이 managed object context에 접근할 수 있다.
2. **새로운 managed object를 생성하여 manged context에 저장하기**
→ 위는 NSManagedObject의 static method인 `entity(forEntityName:in:)` 을 이용하여 한 번에 처리 가능하다. 
NSManagedObject는 어떤 entity도 표현할 수 있는데, NSEntityDescription은 data model의 entity definition과 NSManagedObject 인스턴스를 연결해주는 piece linking이다.
3. attribute를 key-value coding을 통해 설정해준다. data model에 매치되지 않는 잘못된 key를 이용할 경우 앱이 크래시남
4. **commit**
    
    managed object context의 `save()` 를 이용해서 저장할 수 있다. `save` 는 에러를 던질 수 있기 때문에 try 키워드와 함께 이용해야한다. 
    
- managed object context를 가져오거나 entity를 가져오는 부분은 init() 또는 viewDidLoad() 에서 한 번만 진행될 수도 있다.

**core data에 저장된 데이터를 가져오기**

```swift
override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        
        // 1
        guard let appDelegate = UIApplication.shared.delegate as? AppDelegate else {
            return
        }
        
        let managedContext = appDelegate.persistentContainer.viewContext
        
        // 2
        let fetchRequest = NSFetchRequest<NSManagedObject>(entityName: "Person")
        
        // 3
        do {
            people = try managedContext.fetch(fetchRequest)
        } catch let error as NSError {
            print("Could not Fetch. \(error), \(error.userInfo)")
        }
    
    }
```

1. 역시 core data 를 이용하기 전에는 일단 managed object context가 필요하다. 
2. `NSFetchRequest` 는 core data에서 fetch 를 담당하는 클래스이다.

# 2. NSManagedObject Subclasses

<aside>
💡 - subclassing of NSManagedObject to make your own classes for each data entity, creating a direct one-to-one mapping between entities in the data model editor and classes in the code.
- data types available in Core Data entities 
- validating data automatically before saving

</aside>

- data types available in Core Data entities
    - String, Boolean, UUID, URI…
    - Integer 16, Integer 32, Integer 64
        - 16-bit integer: -32768 to 32767
        - 32-bit integer: -21억 to 21djr
        - 64-bit integer: 920경
    - Binary Data (image, pdf files…)
        - **Allows External Storage** 옵션을 이용해서, data를 db에 직접 저장하거나 URI 만을 저장하는 것 중 선택 가능하게 할 수 있다. binary data type에만 가능한 옵션으로, 켜져있다면 해당 attribute를 이용해서 쿼라흔 넋이 불가하다.

## Storing non-standard data types in Core Data

<aside>
💡 UIColor 처럼 attributes 타입에 존재하지 않는 타입을 저장할 때에는 어떻게 해아할까?

</aside>

- Data로 해당 인스턴스를 시리얼라이즈한 후 binary data 타입으로 저장할 수 있다. 이 경우 Data를 가져올 때 다시 UIColor로 바꿔주는 작업이 필요하다.
    
    ```swift
    // Data 로 직접 변환하기
    let entity = NSEntityDescription.entity(forEntityName: "BowTie", in: managedContext)!
    let bowtie = BowTie(entity: entity, insertInto: managedContext)
    
    let image = UIImage(named: imageName!)
    bowtie.photoData = image?.pngData()
    ```
    
- Transformable 을 이용할 수도 있다. 타입과 Data간 변환할 때에 대한 정보를 iOS 에 제공한다.
    - attribute를 transformable하게 만들기 위해서는 세가지 조건이 필요하다.
        1. Add `NSSecureCoding` protocol conformance to the backing data type.
        2. Create and Register an `NSSecureUnarchiveFromDataTransformer` subclass.
        3. Associate the custom data transformer subclass with the Transformable attribute in the Data Model Editor.
    - UIColor를 포함한, Apple framework에 존재하는 대부분의 data type들은 모두 이미 NSSecureCoding을 만족하고 있다.
    

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

1. 해당 data transformer가 decode 가능한 class list를 리턴한다. ( 이 경우 UIColor 만 디코드 가능함)
2. `register` 는 transformer class의 이름 (key)과 transformer instance를 key-value로 매핑해서 저장한다.
3. AppDelegate 의 `application(_:didFinishLaunchingWithOptions:)` 내에서 `register` 를 호출한다. 사실 application이 core data stack을 구성하기 전이라면 어디서든 호출해도 상관 없다.

<img src="https://velog.velcdn.com/images/jujube0/post/22f35e0e-23dc-4643-9d9a-699ae47b370a/image.png">

```swift
// Transformable 이용하기 -> 그냥 그대로 저장하면 됨
bowtie.tintColor = UIColor(named: "red")
```

## subclassing NSManagedObject

<aside>
💡 이전에는 key-value coding을 이용해서 NSManagedObject에 직접 접근하는 방법을 이용했다. 직접 key를 매번 작성하기 때문에 에러에 취약하다는 단점이 있다.

</aside>

key-value coding 대신, NSManagedObject를 서브클래싱할 수 있다. 하나의 data model의 entity를 위한 클래스(이 경우 `BowTie` 클래스)를 만드는 것이다!

xcdatamodeld 파일에서 entity의 inspector를 이용해서 xcode가 manually/automatically 하게 서브클래스를 생성하게 설정할 수 있다.

해당 설정은 BowTie entity를 모델에 추가한 후, 첫 컴파일 전에 설정해야한다. 컴파일 후에 설정을 하게 되면 두 가지 버전의 managed object subclass 버전이 생성된다. ㅠ

**Editor\Create NSManagedObject Subclass**를 선택한 후 원하는 data model을 선택하자. create을 누르면 두 개의 swift file이 추가된다. 

- BowTie+CoreDataProperties.swift
    - the properties that correspond to the BowTie attributes in your data model
- BowTie+CoreDataClass.swift
    - operations (empty first)

아하! custom code들은 CoreDataClass에만 추가된다. 즉, entity가 수정되어도 첫번째 파일만 수정되기 때문에 custom code들은 안전하게 유지되는 것~ 

## Data validation in core data

xcdatamodeld에서 attribute을 누른 후 validation 필드 채우면 됨.

managed object context의 `save` 를 호출하면 context가 validation을 진행함.

validation에 실패하면 error가 발생

예를 들어 maximum 5로 설정한 rating(double)에 더 큰 값ㅇㄹ 넣으면 NSValidationNumberTooLargeError가 밸싱한다.
