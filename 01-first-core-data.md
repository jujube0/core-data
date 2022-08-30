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