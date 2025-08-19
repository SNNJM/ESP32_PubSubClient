# Solution Architecture WebCam Project 

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            App Bootstrap                                 │
│──────────────────────────────────────────────────────────────────────────│
│ • Imports: OpenCV (cv2), NumPy, datetime, os                             │
│ • Entry: main(cam_index=0, width=1280, height=720)                       │
│ • Create resizable window: "Webcam Mini (press 0..6 to change mode)"     │
│ • Ensure ./captures/ folder exists                                       │
│ • Print controls (0..6, c, q) to console                                 │
└──────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                         Webcam Initialization                            │
│──────────────────────────────────────────────────────────────────────────│
│ • cap = cv2.VideoCapture(cam_index)                                      │
│ • cap.set(WIDTH=1280, HEIGHT=720) (driver may pick nearest supported)    │
│ • If camera not opened → print error and exit                            │
└──────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                             Main Runtime Loop                            │
│──────────────────────────────────────────────────────────────────────────│
│ while True:                                                              │
│   • ret, frame = cap.read()                                              │
│     – If ret == False → warn & continue                                  │
│   • processed, save_img = apply_transform(mode, frame)                   │
│   • disp = hud(processed, "<mode> | keys: 0..6, c=capture, q=quit")      │
│     – hud(): ensure BGR; draw top bar + text                             │
│   • If disp is grayscale → convert to BGR for consistent imshow          │
│   • cv2.imshow(window, disp_bgr)                                         │
│   • k = cv2.waitKey(1) & 0xFF                                            │
│     – '0'..'6' → mode switch                                             │
│     – 'c' → save save_img to ./captures/{timestamp()}_<Mode>.png         │
│     – 'q' → break loop (quit)                                            │
└──────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                          Image Transform Pipeline                        │
│──────────────────────────────────────────────────────────────────────────│
│ apply_transform(mode, frame_bgr):                                        │
│   0) Original          → return frame_bgr                                 │
│   1) Grayscale         → cvtColor(BGR2GRAY)                               │
│   2) Gaussian Blur     → Gray → GaussianBlur(7×7, σ=1.4)                  │
│   3) Median Blur       → Gray → medianBlur(ksize=5)                       │
│   4) Canny Edges       → Gray → Canny(80, 160)                            │
│   5) Sobel Magnitude   → Gray → Sobel x/y → magnitude → 0..255 uint8      │
│   6) Sharpen (color)   → filter2D(kernel=[[0,-1,0],[-1,5,-1],[0,-1,0]])   │
│ • Returns (image_for_display, image_for_saving)                           │
│   – Gray/edge modes save 1-channel images; others save color              │
└──────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                           File Capture & Naming                          │
│──────────────────────────────────────────────────────────────────────────│
│ • On 'c' key:                                                            │
│   – fname = ./captures/{YYYYMMDD_HHMMSS}_{Mode}.png                      │
│   – cv2.imwrite(fname, save_img)                                         │
│ • timestamp(): datetime.now().strftime("%Y%m%d_%H%M%S")                  │
└──────────────────────────────────────────────────────────────────────────┘
                                   ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                          Graceful Shutdown                               │
│──────────────────────────────────────────────────────────────────────────│
│ • On 'q' or KeyboardInterrupt:                                           │
│   – cap.release()                                                        │
│   – cv2.destroyAllWindows()                                              │
└──────────────────────────────────────────────────────────────────────────┘

'''
