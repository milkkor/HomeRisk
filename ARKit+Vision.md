
# Using Vision Framework Object Detection in ARKit  
**Dennis Ippel**

> In this short tutorial we’ll use Vision Framework to add object detection and classification capabilities to a bare-bones ARKit project. We’ll use an open source Core ML model to detect a remote control, get its bounding box center, transform its 2D image coordinates to 3D and then create an anchor which can be used for placing objects in an AR scene.

---

## Step 1: Create a New AR Project

To get started you’ll need to create a new **Augmented Reality App** in Xcode:  
**File > New > Project …** and then choose **“Augmented Reality App”**.

Replace the code in `ViewController.swift` with the code below so we can get a clean start:

```swift
import UIKit
import ARKit

class ViewController: UIViewController, ARSCNViewDelegate {

    @IBOutlet var sceneView: ARSCNView!
    
    private var viewportSize: CGSize!
    
    override var shouldAutorotate: Bool { return false }

    override func viewDidLoad() {
        super.viewDidLoad()
        sceneView.delegate = self
        
        viewportSize = sceneView.frame.size
    }
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        let configuration = ARWorldTrackingConfiguration()
        configuration.planeDetection = []
        sceneView.session.run(configuration)
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        sceneView.session.pause()
    }
}
```



---

## Step 2: Add the YOLO Model

> We’ll use a freely available open source Core ML model called **YOLO** which stands for “You Only Look Once”. It is a state-of-the-art, real-time object detection system which can locate and classify 80 different types of objects.

- The Core ML model can be downloaded from Apple’s Developer website:  
  [https://developer.apple.com/machine-learning/models/](https://developer.apple.com/machine-learning/models/)

- Scroll down to **“YOLOv3-Tiny”**, click **“View Models”** and then download the file:  
  `YOLOv3TinyInt8LUT.mlmodel`

- Once downloaded, **drag the `.mlmodel` file from Finder into Xcode** and make sure it is **added to the target**.

---

## Step 3: Capture Camera Image for Detection

Object detection needs a camera image, so we’ll hook into `SCNSceneRendererDelegate`’s  
`renderer(_:willRenderScene:atTime:)` method to query for an image and start the object detection process if the image is available.

---

## Step 4: Create and Perform the Object Detection Request

Using ARKit’s captured image we’ll create an image request and make it perform an object detection request:

```swift
func renderer(_ renderer: SCNSceneRenderer, willRenderScene scene: SCNScene, atTime time: TimeInterval) {
    // Get the capture image (which is a cvPixelBuffer) from the current ARFrame
    guard let capturedImage = sceneView.session.currentFrame?.capturedImage else { return }
    
    let imageRequestHandler = VNImageRequestHandler(cvPixelBuffer: capturedImage, 
                                                    orientation: .leftMirrored, 
                                                    options: [:])
    
    do {
        try imageRequestHandler.perform([objectDetectionRequest])
    } catch {
        print("Failed to perform image request.")
    }
}
```


Here the `imageRequestHandler` performs an object detection request called `objectDetectionRequest`.  
This request needs to be created only once and can be defined in a lazy variable.  
Here, we create an instance of the YOLO model and create a CoreML request.

```swift
lazy var objectDetectionRequest: VNCoreMLRequest = {
    do {
        let model = try VNCoreMLModel(for: YOLOv3TinyInt8LUT().model)
        let request = VNCoreMLRequest(model: model) { [weak self] request, error in
            self?.processDetections(for: request, error: error)
        }
        return request
    } catch {
        fatalError("Failed to load Vision ML model.")
    }
}()
```

---

## Step 5: Handle the Detection Results

`processDetections` is `VNCoreMLRequest`’s completion handler.  
This is where we’ll get the recognized **remote control** object and its **bounding box**, then do all the necessary conversions to get **3D world coordinates** which we can use to create an **ARKit anchor**.

First we need to:

1. Go through all the observations.
2. Check the classification string to see if a remote control is detected.
3. Filter out low confidence observations.

```swift
for observation in results where observation is VNRecognizedObjectObservation {
    guard let objectObservation = observation as? VNRecognizedObjectObservation,
        let topLabelObservation = objectObservation.labels.first,
        topLabelObservation.identifier == "remote", // check if the classified object is a remote control
        topLabelObservation.confidence > 0.9
        else { continue }
  
        ....
```
---

## Step 6: Convert Bounding Box to View Coordinates

> Now that we are confident we’ve detected a remote control we can get its bounding box.  
The bounding box’s coordinates are **normalized image coordinates**.  
A few conversions have to be done to get the **view coordinates**.

```swift
// Get the affine transform to convert between normalized image coordinates and view coordinates
let fromCameraImageToViewTransform = currentFrame.displayTransform(for: .portrait, viewportSize: viewportSize)
// The observation's bounding box in normalized image coordinates
let boundingBox = objectObservation.boundingBox
// Transform the latter into normalized view coordinates
let viewNormalizedBoundingBox = boundingBox.applying(fromCameraImageToViewTransform)
// The affine transform for view coordinates
let t = CGAffineTransform(scaleX: viewportSize.width, y: viewportSize.height)
// Scale up to view coordinates
let viewBoundingBox = viewNormalizedBoundingBox.applying(t)
```

We’ll now use the **bounding box center** as the coordinate we’ll want to convert to 3D space.  
(It might not be the actual center of the detected real-world object but that goes beyond the scope of this tutorial.)

---

## Step 7: Convert 2D to 3D Using Hit Test

To get the 3D world coordinate we can use the **view-space center point** to perform a hit test.  
If we specify `featurePoint` as the hit test result type, ARKit finds the **feature point nearest to the hit-test ray**.  
If we get a result, we can use its `worldTransform` property to create an **ARKit anchor** and add it to the session:

```swift
let midPoint = CGPoint(x: viewBoundingBox.midX,
                       y: viewBoundingBox.midY)

let results = sceneView.hitTest(midPoint, 
                                types: .featurePoint)

guard let result = results.first else { continue }

let anchor = ARAnchor(name: "remoteObjectAnchor", 
                      transform: result.worldTransform)

sceneView.session.add(anchor: anchor)
```

---

## Step 8: Add 3D Object on Detection

Adding this anchor to the session will invoke `ARSCNViewDelegate`’s `renderer(_, didAdd:, for:)` function,  
in which we can add **3D content** to the scene.  
In this case we’ll add a **simple red sphere** and attach it to the anchor:

```swift
func renderer(_ renderer: SCNSceneRenderer, didAdd node: SCNNode, for anchor: ARAnchor) {
    guard anchor.name == "remoteObjectAnchor" else { return }
    let sphereNode = SCNNode(geometry: SCNSphere(radius: 0.01))
    sphereNode.geometry?.firstMaterial?.diffuse.contents = UIColor.red
    node.addChildNode(sphereNode)
}
```

---

## Conclusion

> The red sphere will now be placed on the detected remote control and will persist like any other ARKit object.  

Again, this might not be the most accurate solution to place virtual content over real-world content.  
However, it shows a few **really useful techniques** that might come in handy in your own ARKit projects.

---
