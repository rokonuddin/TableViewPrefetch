# TableViewPrefetch

Apple has introduced API for prefetching `UITableView` and `UICollectionView` in iOS 10. This README is a short story of the working example of  `UITableViewDataSourcePrefetching` protocol.

## Overview

To implement prefetching, we confirmed `UITableViewDataSourcePrefetching` protocol to our view controller just like `UITableViewDataSource` and `UITableViewDelegate`. That enables table view's data source to begin loading data for cells before `tableView(_:cellForRowAt:)` data source method is called. 

### Getting Started

Set view contrpller to table view prefetch datasource

```swift
tableView.prefetchDataSource = self
```

Initiate asynchronous loading of the data required for the cells at the specified index paths in your implementation of `tableView(_:prefetchRowsAt:)`

```swift
func tableView(_ tableView: UITableView, prefetchRowsAt indexPaths: [IndexPath]) {
    for indexPath in indexPaths {
    	guard let _ = loadingOperations[indexPath] else { return }
    		if let dataLoader = dataStore.loadPhoto(at: indexPath.row) {
		  		loadingQueue.addOperation(dataLoader)
			  	loadingOperations[indexPath] = dataLoader
		  }
    }
}
```

Cancel pending data load operations when the table view informs you that the data is no longer required in the `tableView(_:cancelPrefetchingForRowsAt:)` method


```swift
func tableView(_ tableView: UITableView, cancelPrefetchingForRowsAt indexPaths: [IndexPath]) {
	for indexPath in indexPaths {
		if let dataLoader = loadingOperations[indexPath] {
			dataLoader.cancel()
			loadingOperations.removeValue(forKey: indexPath)
		}
	}
}
```

### Loading Data Asynchronously

Unlike `tableView(_:cellForRowAt:)`, the `tableView(_:prefetchRowsAt:)` method is not necessarily called for every cell in the table view. It is called for cells that are not visible in the screen. Implementation of `tableView(_:cellForRowAt:)` therefore must be able to handle the following potential situations

* Data has been loaded via the prefetch request, and is ready to be displayed.

```swift
// Has the data already been loaded?
if let image = dataLoader.image {
    cell.updateAppearanceFor(image)
    loadingOperations.removeValue(forKey: indexPath)
} 
```
* Data is currently being prefetched, but is not yet available.

```swift
else {
   // No data loaded yet, so add the completion closure to update the cell once the data arrives
   dataLoader.loadingCompleteHandler = updateCellClosure
}
```
* Data has not yet been requested. 

```swift 
// Need to create a data loaded for this index path
if let dataLoader = dataStore.loadImage(at: indexPath.row) {
	// Provide the completion closure, and kick off the loading operation
	dataLoader.loadingCompleteHandler = updateCellClosure
	loadingQueue.addOperation(dataLoader)
	loadingOperations[indexPath] = dataLoader
}   
```
To handles all of these situations Operation is used to load the data for each row. We create the Operation object and store it in the prefetch method. The data source method can then either retrieve the operation and the result, or create it if it doesn’t exist.

```swift
class DataLoadOperation: Operation {
    var image: UIImage?
    var loadingCompleteHandler: ((UIImage?) -> ())?
    private var _image: ImageModel
    
    init(_ image: ImageModel) {
        _image = image
    }
    
    override func main() {
        if isCancelled { return }
        guard let url = _image.url else { return }
        downloadImageFrom(url) { (image) in
            DispatchQueue.main.async() { [weak self] in
                guard let `self` = self else { return }
                if self.isCancelled { return }
                self.image = image
                self.loadingCompleteHandler?(self.image)
            }
        }
    }
}
```
### Reference
[developer.apple.com](https://developer.apple.com/documentation/uikit/uitableviewdatasourceprefetching)

[www.raywenderlich.com](https://www.raywenderlich.com/5000-ios-10-collection-view-data-prefetching)

[andreygordeev.com](https://andreygordeev.com/2017/02/20/uitableview-prefetching/)
