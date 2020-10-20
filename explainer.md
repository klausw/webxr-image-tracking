# WebXR Image Tracking API - explainer

## Introduction

Augmented reality platforms that are based on camera mapping often have the capability to recognize user-specified images in the environment and provide tracking information for them. The Image Tracking API makes this feature available to WebXR applications.

For an example of this functionality in a native application, see:

  https://developers.google.com/ar/develop/java/augmented-images

Since detecting and tracking these images happens locally on the device, this functionality can be implemented without providing camera images to the application.

## Use Cases

- augmenting a physical tabletop game by recognizing components and adding 3D models to them when viewed through a smartphone WebXR application

- creating shared anchor points for a multi-user AR experience by placing trackable images in the environment

- using a poster as a portal into an immersive experience for an art installation

## Using the API

1. Request image tracking as a required or optional feature using its feature descriptor and a list of images to track:

```js
const session = await navigator.xr.requestSession('immersive-ar', {
  requiredFeatures: ['image-tracking'],
  trackedImages: [
    {
      image: document.getElementById('img'),
      widthInMeters: 0.2
    }
  ]
});
```

The `image` is a reference to a HTMLImage element. For effective tracking, images must contain sufficient detail, must not have rotational or mirror symmetry, and should avoid repeating patterns or flat single-colored areas.

The image must be from the same origin as the WebXR application, or accessible cross-origin according to a valid CORS policy. Tracking images is only permitted if their contents could be read by the application. (The same restrictions apply when using an image element as a `gl.texImage2D` texture source.) If a tracked image is inaccessible, the `requestSession` promise is rejected with a security exception.

The `widthInMeters` specifies the expected measured width of the image in the real world. This is required, but can be an estimate. If the actual size doesn't match the expected size, the initial reported pose when the image is first recognized is likely to be inaccurate, and may remain inaccurate if the system can't reliably detect its true 3D position. When viewed from a fixed camera position, a half-sized image at half the distance looks identical to a full-sized image, and the tracking system can't differentiate these cases without additional context about the environment.

2. in requestAnimationFrame, query XRFrame for the current state of tracked images:

```js
const results = frame.getImageTrackingResults();
for (const result of results) {
  const element = result.image;
  const pose = result.getPose(referenceSpace);
  if (pose) {
    // tracking information is available in pose.transform
    if (pose.emulatedPosition) {
      // the object's position is being inferred, it may be partially visible
    } else {
      // image is being actively tracked
  } else {
    // image is trackable, but tracking information is currently unavailable
  }
}
```

There are four possible result types for the image tracking results:

1. The tracked image has a pose with `emulatedPosition: false`. This means the image was recognized and is currently being actively tracked in 3D space.
1. The tracked image has a pose with `emulatedPosition: true`. This means that the image was recognized and tracked recently, but may currently be out of camera view or obscured, and the reported pose is extrapolated from recent tracking results. 
1. The tracked image has an entry in results, but the pose is `null`. This means that the image is trackable, but there is no current pose information, for example because the image hasn't been recognized yet, or because its last recognized position is too old to be useful.
1. The tracked image does not appear as an entry in results. This means that the source image is not trackable at all, for example because it had insufficient distinctive features to be recognized, and won't be detected in future frames either.

The pose position corresponds to the center point of the tracked image. The pose orientation has +x pointing toward the right edge of the image and +y toward the top of the image. The +z axis is orthogonal to the picture plane, pointing toward the viewer when the image is in front.

The returned image tracking data also includes a `widthInMeters` value as measured by the tracking system. This is initially equal to the application-provided width, then updated on a best-effort basis when the image is being actively tracked. If the tracking result has a pose with `emulatedPosition: false`, drawing a rectangle of this width at the provided pose in 3D space should ideally be a close match to the image as seen by the tracking camera. If the actual size differs from the initially specified size and hasn't been accurately measured yet, this rectangle should still be at the expected 2D position when drawn on a 2D screen, but may be at the wrong depth.

## Appendix: Proposed Web IDL

```webidl

partial dictionary XRSessionInit {
  sequence<XRTrackedImageInit> trackedImages;
};

dictionary XRTrackedImageInit {
  HTMLImageElement image;
  float widthInMeters;
};

partial interface XRFrame {
  FrozenArray<XRImageTrackingResult> getImageTrackingResults();
}

interface XRImageTrackingResult {
  XRPose? getPose(XRSpace relative_to);
  float widthInMeters;

  [SameObject]
  readonly attribute HTMLImageElement image;
};


```
