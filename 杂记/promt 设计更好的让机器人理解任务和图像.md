![[Pasted image 20260118185040.png]]

```
{
  "model": "gemini-robotics-er-1.5-preview",
  "contents": {
    "parts": [
      {
        "inlineData": {
          "data": "<BASE64_IMAGE_DATA_REDACTED>",
          "mimeType": "image/png"
        }
      },
      {
        "text": "Detect object should be noticed during in task \"Place the cylinder into the circle hole in the first row\", with no more than 20 items. Output a json list where each entry contains the 2D bounding box in \"box_2d\" and a text label in \"label\"."
      }
    ]
  },
  "config": {
    "temperature": 0.5,
    "responseMimeType": "application/json"
  }
}
```

prompt描述为：Detect object should be noticed during in task "Place the cylinder into the circle hole in the first row"
![[Pasted image 20260118185221.png]]