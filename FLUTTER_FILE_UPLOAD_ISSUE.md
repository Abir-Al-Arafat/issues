# Issue Resolution: Flutter File Upload MIME Type Mismatch

## Problem Statement

During the development of our ride-sharing application, I encountered a critical issue where file uploads from our Flutter mobile app were consistently failing, while the same uploads worked perfectly when tested via Postman. The error message indicated "Invalid file type" despite sending valid `.jpg` image files.

## Error Details

**Console Output:**

```
📎 File Upload Details:
  Field name: image
  Original name: scaled_35.jpg
  MIME type: application/octet-stream
  Size: undefined bytes
---
Global Error Error: Invalid file type
    at DiskStorage.destination [as getDestination]
```

**Key Observations:**

- Flutter was sending `.jpg` files with MIME type: `application/octet-stream`
- Postman was sending the same file type with MIME type: `image/jpeg`
- The file validation logic only checked MIME types, not file extensions

## Root Cause Analysis

The backend Multer file upload middleware was configured to validate uploaded files based solely on their MIME type headers. However, Flutter's HTTP client (particularly when using packages like `http` or `dio`) often sends files with a generic `application/octet-stream` MIME type instead of the specific image MIME type.

This happened in two critical places in the code:

1. **Storage Destination Function**: Determined upload folder (images/videos/audios/pdfs) based on MIME type
2. **File Filter Function**: Validated allowed file types based on MIME type

Both functions rejected files with `application/octet-stream` MIME type, causing all Flutter uploads to fail.

## Solution Implemented

I implemented a **dual-validation approach** that checks both MIME type AND file extension, making the upload system compatible with both web/Postman and mobile/Flutter clients.

### Changes Made

**1. Updated Storage Destination Logic:**

```typescript
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    let uploadPath = "";

    // Get file extension
    const ext = path.extname(file.originalname).toLowerCase();

    // Define allowed extensions
    const imageExtensions = [".jpg", ".jpeg", ".png", ".gif", ".webp", ".bmp"];
    const videoExtensions = [".mp4", ".avi", ".mov", ".mkv", ".webm"];
    const audioExtensions = [".mp3", ".wav", ".ogg", ".m4a"];
    const pdfExtensions = [".pdf"];

    // Determine upload path based on MIME type OR file extension
    if (file.mimetype.startsWith("image/") || imageExtensions.includes(ext)) {
      uploadPath = path.join(__dirname, "../../public/uploads/images");
    } else if (
      file.mimetype.startsWith("video/") ||
      videoExtensions.includes(ext)
    ) {
      uploadPath = path.join(__dirname, "../../public/uploads/videos");
    }
    // ... similar logic for audio and PDF
  },
});
```

**2. Updated File Filter Validation:**

```typescript
const fileFilter = (req: Request, file: Express.Multer.File, cb: any) => {
  const ext = path.extname(file.originalname).toLowerCase();

  const imageExtensions = [".jpg", ".jpeg", ".png", ".gif", ".webp", ".bmp"];
  // ... other extension arrays

  // Check MIME type OR file extension (for Flutter compatibility)
  const isImage =
    file.mimetype.startsWith("image/") || imageExtensions.includes(ext);
  const isOctetStream = file.mimetype === "application/octet-stream";

  // Allow if valid MIME type OR if octet-stream with valid extension
  if (
    isImage ||
    isVideo ||
    isAudio ||
    isPdf ||
    (isOctetStream &&
      [
        ...imageExtensions,
        ...videoExtensions,
        ...audioExtensions,
        ...pdfExtensions,
      ].includes(ext))
  ) {
    cb(null, true);
  } else {
    cb(new Error("Invalid file type"));
  }
};
```

## Technical Benefits

1. **Cross-Platform Compatibility**: Works seamlessly with web browsers, API testing tools (Postman), and mobile apps (Flutter, React Native)
2. **Security Maintained**: Still validates file types, just using extension as a fallback
3. **User Experience**: Mobile users can now upload images without errors
4. **Debugging Enhanced**: Added detailed console logging to track file upload attempts

## Testing & Validation

**Before Fix:**

- ❌ Flutter uploads: Failed with "Invalid file type"
- ✅ Postman uploads: Succeeded

**After Fix:**

- ✅ Flutter uploads: Succeeded (validates by extension)
- ✅ Postman uploads: Succeeded (validates by MIME type)
- ✅ Files correctly routed to appropriate folders (images/videos/audios/pdfs)

## Lessons Learned

1. **Platform Differences Matter**: Different HTTP clients handle file metadata differently
2. **Defensive Programming**: Always implement fallback validation strategies
3. **Extension-Based Validation**: File extensions are more reliable across platforms than MIME types
4. **Comprehensive Testing**: Always test file uploads from actual mobile devices, not just API tools
5. **Logging is Critical**: Detailed logging helped identify the root cause quickly

## Impact

This fix unblocked the mobile development team and allowed users to:

- Upload profile images from the Flutter app
- Upload passenger images during booking creation
- Upload driver verification documents
- Upload package delivery photos

The resolution time was under 30 minutes once proper logging was implemented to identify the MIME type mismatch.
