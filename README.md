# ONLY ~~WORKS~~ ON iTERM2

![image](https://github.com/user-attachments/assets/cca6b299-8705-4db8-adcd-6409df244a10)
![image](https://github.com/user-attachments/assets/bf933911-d9d3-412f-b671-461aa4a7ce0d)
![image](https://github.com/user-attachments/assets/047683dd-9938-4c26-ba81-64d561bc6272)

https://github.com/user-attachments/assets/67d06439-8b45-4395-82a3-1f585aa7c23a

### Longer Demo [https://www.youtube.com/watch?v=AANogTvjWbY]

[![YouTube](http://i.ytimg.com/vi/AANogTvjWbY/hqdefault.jpg)](https://www.youtube.com/watch?v=AANogTvjWbY)

MAKE SURE TO ADD -uhr flag in the end, otherwise it plays like mpv tct video output BUT WORSE

## Idea

lol was playing around with mpv --vo=tct and thought to myself "hmm what IS the 'highest quality' video player that plays/renders the video directly in the terminal?" dumb idea ik but i needed to scratch that itch

Requirements:
    - Definition of video: has to reflect closely to what the source looks like (colors, framerate) and also has to be synched to audio like source
    - Video HAS TO play inside the terminal (in this case iTerm2), user should not have to leave the CLI

Need improvement:
        
        - rewrite in c++ more optimized opencv
        
        - get it working with short videos
        
        - DOES NOT WORK PROPERLY ON RETINA SCREENS, or shorter videos
    

#### dependencies
    - Python 3.6+
    - OpenCV (cv2)
    - NumPy
    - Pillow (PIL)
    - FFmpeg/FFplay for audio playback

    ## Installation

    1. Clone this repository:
       ```
       git clone https://github.com/jp-vn/VidiTerm2.git
       cd VidiTerm-main
       ```
    
    2. Install the required Python packages:
       ```
       pip install opencv-python numpy pillow
       ```
    
    3. Install FFmpeg (includes FFplay for audio):
       
       On macOS (using Homebrew):
       ```
       brew install ffmpeg
       ```

    run in iTerm2:   
    python3 VidiTerm.py path/to/your/video.mp4 -uhr

    make sure to include the -uhr flag!
