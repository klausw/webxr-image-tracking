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

Request image tracking as a required or optional feature using its feature descriptor and a list of images to track:

```js
const img = document.getElementById('img);
// Ensure the image is loaded and ready for use
// FIXME: does createImageBitmap do this implicitly?
await img.decode();
const imgBitmap = createImageBitmap(img);

const session = await navigator.xr.requestSession('immersive-ar', {
  requiredFeatures: ['image-tracking'],
  trackedImages: [
    {
      image: imgBitmap,
      widthInMeters: 0.2
    }
  ]
});
```

The `image` attribute must be an ImageBitmap, it can be created from various image sources such as HTMLImageElements using [createImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/createImageBitmap). The image content is a snapshot at the time the `requestSession` call is made, and later changes to the image such as redrawing a canvas source image have no effect on tracking.

For effective tracking, images must contain sufficient detail, must not have rotational symmetry, and should avoid repeating patterns or flat single-colored areas.

If multiple images are being tracked, results can be unpredictable if the images share common features such as a logo appearing on multiple images. In that case, the system may return tracking results for the wrong image.

The `widthInMeters` attribute specifies the expected measured width of the image in the real world. This is required, but can be an estimate. If the actual size doesn't match the expected size, the initial reported pose when the image is first recognized is likely to be inaccurate, and may remain inaccurate if the system can't reliably detect its true 3D position. When viewed from a fixed camera position, a half-sized image at half the distance looks identical to a full-sized image, and the tracking system can't differentiate these cases without additional context about the environment.

Once the session is active, in requestAnimationFrame, query XRFrame for the current state of tracked images:

```js
const results = frame.getImageTrackingResults();
for (const result of results) {
  // The result's index is the image's position in the trackedImages array specified at session creation
  const image_index = result.index;

  // Get the current pose if available.
  const pose = result.getPose(referenceSpace);

  if (state == "tracked") {
    HighlightImage(image_index, pose);
  } else if (state == "emulated") {
    FadeImage(image_index, pose);
  } else if (state == "lost") {
    HideImage(image_index);
  } else if (state == "untrackable") {
    MarkImageUntrackable(image_index);
    if (RemainingTrackableImageCount() == 0) {
      WarnUser("No trackable images");
    }
  }
}
```

The result list only contains entries for tracked images whose information has changed. The order of results is arbitrary, but each result's `index` attribute provides its location in the `trackedImages` array used with the initial session request.

The `trackingState` attribute provides information about the tracked image:

* `tracked` means the image was recognized and is currently being actively tracked in 3D space, and is at least partially visible to a tracking camera. (This does not necessarily mean that it's visible in the user's viewport in case that differs from the tracking camera field of view.)
* `emulated` means that the image was recognized and tracked recently, but may currently be out of camera view or obscured, and the reported pose is based on assuming that the object remains at the same position and orientation as when it was last seen. This pose is likely to be adequate for a poster attached to a wall, but may be unhelpful for an image attached to a moving object.
* `lost` means that the image is no longer being tracked, for example because its most recent known pose is too old and/or the current environment doesn't have sufficient reference points to calculate a location.
* `untrackable` means that the source image is not trackable at all, for example because it had insufficient distinctive features to be recognized. This state is only reported once for affected images, typically shortly after session start, and the affected images won't ever appear in results for the remainder of the session.

The pose position corresponds to the center point of the tracked image. The pose orientation has +x pointing toward the right edge of the image and +y toward the top of the image. The +z axis is orthogonal to the picture plane, pointing toward the viewer when the image is in front.

The returned image tracking data also includes a `measuredWidthInMeters` value as measured by the tracking system. This is zero if this is unknown, for example due to the image not having been detected yet, then updated on a best-effort basis when the image is being actively tracked. If the tracking state is `tracked`, drawing a rectangle of this width at the provided pose in 3D space should ideally be a close match to the image as seen by the tracking camera. If the actual size differs from the initially specified size and hasn't been accurately measured yet, drawing this rectangle on a 2D screen should still visually appear at the expected screen position and size, but may be at the wrong depth, leading to incorrect occlusion compared to other scene objects.

## Appendix: Proposed Web IDL

```webidl

partial dictionary XRSessionInit {
  sequence<XRTrackedImageInit> trackedImages;
};

dictionary XRTrackedImageInit {
  ImageBitmap image;
  float widthInMeters;
};

partial interface XRFrame {
  FrozenArray<XRImageTrackingResult> getImageTrackingResults();
};

enum XRImageTrackingState {
  "untrackable",
  "tracked",
  "emulated",
  "lost"
};

interface XRImageTrackingResult {
  XRPose? getPose(XRSpace relativeTo);

  readonly attribute unsigned long index;
  readonly attribute XRImageTrackingState trackingState;
  readonly attribute float measuredWidthInMeters;
};
```
