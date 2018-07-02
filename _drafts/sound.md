Slowing down audio from 48048 to 48000 to fix 23.97 vs 24 fps in fcpx


```bash
afconvert recording.m4a -o temp.wav --file WAVE --data LEI16
sox temp.wav output.wav tempo 1.00192
afconvert output.wav -o output.aac --file mp4f --data 'aac '
```

