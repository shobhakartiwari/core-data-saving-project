# core-data-saving-project

Certainly! Here's the complete code for an iOS UIKit project in Swift that fetches data from a server, saves images to Core Data, and displays them in a table view. This includes the `CoreDataManager` class as well.

### Step 1: Set up your project

1. Open Xcode and create a new iOS project:
   - Product Name: ImageTableViewDemo
   - Interface: Storyboard
   - Language: Swift

2. Replace the contents of `ViewController.swift` with the following code.

### ViewController.swift

```swift
import UIKit

class ViewController: UIViewController, UITableViewDataSource, UITableViewDelegate {
    
    @IBOutlet weak var tableView: UITableView!
    
    var photos: [PhotoEntity] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        tableView.dataSource = self
        tableView.delegate = self
        fetchPhotos()
    }
    
    func fetchPhotos() {
        photos = CoreDataManager.shared.fetchAllPhotos()
        tableView.reloadData()
        
        // Fetch from server if needed
        fetchPhotosFromServer()
    }
    
    func fetchPhotosFromServer() {
        let url = URL(string: "https://jsonplaceholder.typicode.com/photos")!
        let task = URLSession.shared.dataTask(with: url) { data, response, error in
            guard let data = data, error == nil else {
                print("Failed to fetch data: \(error?.localizedDescription ?? "Unknown error")")
                return
            }
            do {
                let decoder = JSONDecoder()
                let fetchedPhotos = try decoder.decode([Photo].self, from: data)
                
                for photo in fetchedPhotos {
                    if let imageData = try? Data(contentsOf: photo.thumbnailUrl),
                       let image = UIImage(data: imageData) {
                        CoreDataManager.shared.createPhotoEntity(id: photo.id, title: photo.title, thumbnailUrl: photo.thumbnailUrl.absoluteString, imageData: imageData)
                    }
                }
                
                DispatchQueue.main.async {
                    self.fetchPhotos() // Reload data after saving to Core Data
                }
            } catch {
                print("Failed to decode JSON: \(error.localizedDescription)")
            }
        }
        task.resume()
    }
    
    // MARK: - UITableViewDataSource
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return photos.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        guard let cell = tableView.dequeueReusableCell(withIdentifier: "PhotoCell", for: indexPath) as? PhotoTableViewCell else {
            fatalError("Unable to dequeue PhotoTableViewCell")
        }
        let photo = photos[indexPath.row]
        if let imageData = photo.imageData, let image = UIImage(data: imageData) {
            cell.configure(with: image, title: photo.title ?? "")
        } else {
            cell.configure(with: UIImage(named: "placeholder")!, title: photo.title ?? "")
        }
        return cell
    }
}

```

### CoreDataManager.swift

Create a new Swift file named `CoreDataManager.swift` and add the following code:

```swift
import Foundation
import CoreData

class CoreDataManager {
    static let shared = CoreDataManager() // Singleton instance
    
    private init() {} // Private initializer to ensure singleton
    
    // MARK: - Core Data stack
    
    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "ImageTableViewDemo")
        container.loadPersistentStores(completionHandler: { (_, error) in
            if let error = error as NSError? {
                fatalError("Unresolved error \(error), \(error.userInfo)")
            }
        })
        return container
    }()
    
    var managedObjectContext: NSManagedObjectContext {
        return persistentContainer.viewContext
    }
    
    // MARK: - Core Data Saving support
    
    func saveContext () {
        let context = persistentContainer.viewContext
        if context.hasChanges {
            do {
                try context.save()
            } catch {
                let nserror = error as NSError
                fatalError("Unresolved error \(nserror), \(nserror.userInfo)")
            }
        }
    }
    
    // MARK: - PhotoEntity CRUD operations
    
    func createPhotoEntity(id: Int, title: String, thumbnailUrl: String, imageData: Data?) {
        let photoEntity = PhotoEntity(context: managedObjectContext)
        photoEntity.id = Int32(id)
        photoEntity.title = title
        photoEntity.thumbnailUrl = thumbnailUrl
        photoEntity.imageData = imageData
        
        saveContext()
    }
    
    func fetchAllPhotos() -> [PhotoEntity] {
        let fetchRequest: NSFetchRequest<PhotoEntity> = PhotoEntity.fetchRequest()
        do {
            let photos = try managedObjectContext.fetch(fetchRequest)
            return photos
        } catch {
            print("Failed to fetch photos: \(error.localizedDescription)")
            return []
        }
    }
}
```

### Photo.swift

Create a new Swift file named `Photo.swift` and add the following code:

```swift
import Foundation

struct Photo: Codable {
    let id: Int
    let title: String
    let thumbnailUrl: URL
}
```

### PhotoEntity+CoreDataProperties.swift

Make sure you have the `PhotoEntity` class generated automatically from your Core Data model. This should include properties for `id`, `title`, `thumbnailUrl`, and `imageData`.

### PhotoTableViewCell.swift

Ensure you have a `PhotoTableViewCell` class set up in your project to display the photos in the table view cells. Here's a basic implementation:

```swift
import UIKit

class PhotoTableViewCell: UITableViewCell {
    
    @IBOutlet weak var photoImageView: UIImageView!
    @IBOutlet weak var titleLabel: UILabel!
    
    func configure(with image: UIImage, title: String) {
        titleLabel.text = title
        photoImageView.image = image
    }
}
```

### Storyboard Setup

1. Open your `Main.storyboard`.
2. Drag and drop a `UITableView` onto your view controller.
3. Create a prototype cell for the table view and set its style to Custom.
4. Add an `UIImageView` and a `UILabel` to the prototype cell.
5. Assign the `PhotoTableViewCell` class to the prototype cell in the Identity Inspector and set a reuse identifier (e.g., "PhotoCell").
6. Connect the `tableView` outlet in your `ViewController` to the actual `UITableView` in the storyboard.

### Running the Project

Run your project on a simulator or device. It should fetch photos from the server, save them to Core Data using `CoreDataManager`, and display them in the table view using `ViewController`. Adjust the code as per your specific Core Data model and requirements.
