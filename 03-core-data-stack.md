# 3. The Core Data Stack

# 4 core data classes

-   **NSManagedObjectModel**
-   **NSPersistentStore**
-   **NSPersistentStoreCoordinator**
-   **NSManagedObjectContext**

지금까지는 초기 세팅시 _use core data_ 를 선택해주는 것으로, xcode의 도움을 받아 core data 초기 세팅을 진행했다.

하지만 persistent store migration 등의 특별한 작업들을 위해서는 core data stack에 대해 아는 것이 필수적이다. 일단 네가지 core data stack에 존재하는 네가지 클래스들에 대해 먼저 알아보자.

## The managed object model

> **NSManagedObjectModel** represents each object type in yout app’s data model, the properties they can have, and the relationships between them.

Other parts of the core data stack use the model to create objects, store properties and save data.

## The persistent store

> **NSPersistentStore** _reads and writes data_ to whichever storage method you’ve decided to use

4가지 타입이 있음 (그 중 세개는 atomic presistent store로, deserialized되어 메모리에 로드된 후에서야 읽기 또는 쓰기 작업이 가능하다.)

**four built-in Core Data store types**

1.  **NSSQLiteStoreType** : SQLite databse. 유일한 non-atomic store type. lightweight & efficient
    -   Xcode가 기본으로 이용하는 타입
2.  **NSXMLStoreType** : XML file 을 이용하기 때문에 가장 human-readable하다.
    -   atomic → large memory footprint
    -   OS X에서만 이용 가능하다.
3.  **NSBinaryStoreType** : binary data file을 이용한다.
    -   역시 atomic. 잘 이용 안함
4.  **NSInMemoryStoreType**: in-memory persistent store type
    -   메모리를 이용하는 거기 때문에 사실 persistent하지 않다.(앱을 끄면 사라진다.)
    -   unit-test나 caching에 주로 이용된다.

## The persistent store coordinator

> bridge between the managed object model and the persistent store. managed object model을 이해하고, persistent store로부터 값을 보내고 받는 것을 모두 담당한다.

추가로 persistent store의 configuration을 숨기는 역할도 담당한다.

→ managed object context는 어떤 형식(SQLite database, XML…)으로 데이터를 저장하는지 모르며, 복수개의 persistent store를 가지고 있을 경우에도 store coordinator가 하나의 interface로 작용하여 managed context가 이를 신경쓰지 않아도 되게한다.

## The managed object context

> 나머지들은 advance한 동작을 원할 때만 이용하는 반면, 실제로 주로 이용하는 것은 managed object context이다.

-   context는 managed object를 다루는 in-memory scratchpad이다.
-   Core Data object과 관련된 모든 작업들은 managed object context 내에서 이루어진다.
-   context의 `save()` 를 호출해야 context에 작업한 결과물들이 실제 persistent store에 반영된다.
-   context는 object들의 lifecycle(faulting, inverse relationship handling and validation)도 함께 관리한다.
-   managed object는 항상! 연관된 context를 가지고 있다. (reference를 항상 가지고 있기 때문에 `object.managedObjectContext` 를 통해 접근할 수 있다.) 추가로 한 번 연결된 managed object와는 life cycle 내에서 계속 연결을 유지한다.
-   하나의 Application은 하나 이상의 context를 가질 수 있다. context는 in-memory scratch 임을 기억하자. 같은 Core Data object를 각기 다른 context에 동시에 로드하는 것도 가능하다.
-   context와 managed object는 thread-safe가 아니다.

## The persistent store container

iOS10부터 생긴, 새로운 클래스.

나머지 네 개의 core data stack class들을 가지고 있는 컨테이너.

container를 초기화하고 persistent store 를 로드하기만 하면 나머지 작업들을 대신 해준다.

# Practice

## Creating stack object

-   CoreDataStack.swift 파일 생성

```swift
import Foundation
import CoreData

class CoreDataStack {
  private let modelName: String
  
  lazy var managedContext: NSManagedObjectContext = {
    self.storeContainer.viewContext
  }()
  
  init(modelName: String) {
    self.modelName = modelName
  }
  
  private lazy var storeContainer: NSPersistentContainer = {
    
    let container = NSPersistentContainer(name: self.modelName)
    container.loadPersistentStores { _, error in
      if let error = error as NSError? {
        print("Unresolved error \\(error), \\(error.userInfo)")
      }
    }
    return container
  }()
  
  func saveContext() {
    guard managedContext.hasChanges else { return }
    
    do {
      try managedContext.save()
    } catch let error as NSError {
      print("Unresolved error \\(error), \\(error.userInfo)")
    }
  }
}
```

-   이용할 때는,

```swift
lazy var coreDataStack = CoreDataStack(modelName: "DogWalk")
```

## Modeling data

-   modelName과 동일하게 xcdatamodeld 파일을 생성해야한다.

**Relationships**

model간의 relationship을 설정할 수 있다.

<img src="https://velog.velcdn.com/images/jujube0/post/69394aac-c43f-49d2-aaa4-878758049116/image.png">


기본으로 one-to-one relationship으로 설정되기 때문에 inspector에서 Type을 바꿔줘야한다.

-   destination: receiving end of a relationship
-   inverse: how to find its way back

우측 하단의 Editor Style을 바꿔서 ERD를 볼 수 있다.

## Adding managed object subclasses

Dog, Walk 엔티티의 Codegen 속성을 Manual/None으로 바꿔준다.

Editor > Create NSManagedObject Subclass 를 누르고 Dog, Walk entity를 모두 클릭해서 Create!

-   fetch

```swift
let dogName = "Fido"
let dogFetch = Dog.fetchRequest()
dogFetch.predicate = NSPredicate(format: "%K = %@", #keyPath(Dog.name), dogName)

do {
  let results = try coreDataStack.managedContext.fetch(dogFetch)
  if results.isEmpty {
    currentDog = Dog(context: coreDataStack.managedContext)
    currentDog?.name = dogName
    coreDataStack.saveContext()
  } else {
    currentDog = results.first
  }
} catch let error as NSError {
  print("Fetch error: \\(error) description: \\(error.userInfo)")
}
```

-   save

```swift
let walk = Walk(context: coreDataStack.managedContext)
walk.date = Date()

if let dog = currentDog,
   let walks = dog.walks?.mutableCopy() as? NSMutableOrderedSet {
  walks.add(walk)
  dog.walks = walks
}

coreDataStack.saveContext()
```

-   delete
```swift
coreDataStack.managedContext.delete(walkToRemove) coreDataStack.saveContext()
```