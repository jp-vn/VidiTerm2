import cv2
import numpy as np
import os
import sys
import argparse
import subprocess
import signal
import fcntl
import termios
import struct
import time
import threading
import shutil
import base64
import io
from queue import Queue
from PIL import Image

class TerminalPlayer:
    def __init__(self, video_path, native_resolution=True, ultra_high_resolution=False):
        self.video_path = video_path
        self.native_resolution = native_resolution
        self.ultra_high_resolution = ultra_high_resolution
        self.cap = cv2.VideoCapture(video_path)
        self.fps = self.cap.get(cv2.CAP_PROP_FPS)
        self.frame_time = 1/self.fps
        self.orig_width = int(self.cap.get(cv2.CAP_PROP_FRAME_WIDTH))
        self.orig_height = int(self.cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
        self.audio_process = None
        
        # Increase queue size for ultra high resolution mode to handle speed fluctuations
        queue_size = 30 if ultra_high_resolution else 10
        self.frame_queue = Queue(maxsize=queue_size)
        self.stop_event = threading.Event()
        
        # Detect terminal capabilities
        self.term_width, self.term_height = shutil.get_terminal_size()

    def get_terminal_size(self):
        # Get terminal size in characters
        h, w = struct.unpack('HHHH', 
            fcntl.ioctl(sys.stdout.fileno(), termios.TIOCGWINSZ, 
            struct.pack('HHHH', 0, 0, 0, 0)))[:2]
            
        # Use half-blocks for all non-ultra modes
        h_multiplier = 2  # For half-blocks
            
        return w, h * h_multiplier

    def resize_frame(self, frame):
        term_w, term_h = self.get_terminal_size()
        
        # For ultra high resolution mode, use higher quality processing
        if self.ultra_high_resolution:
            # For the ultra high resolution mode, we don't want to downscale the video
            # We'll just return the original frame to maintain the highest quality
            # The iTerm2 image protocol will handle proper scaling
            return frame
            
        # Calculate aspect ratio preserving dimensions
        video_ratio = self.orig_width / self.orig_height
        term_ratio = term_w / term_h
        
        if term_ratio > video_ratio:
            new_height = term_h
            new_width = int(term_h * video_ratio)
        else:
            new_width = term_w
            new_height = int(term_w / video_ratio)
        
        # Use high-quality interpolation for better results
        # INTER_LANCZOS4 provides higher quality than the default INTER_LINEAR
        resized = cv2.resize(frame, (new_width, new_height), 
                            interpolation=cv2.INTER_LANCZOS4)
                            

            
        return resized
        


    def frame_to_terminal(self, frame):
        height, width = frame.shape[:2]
        
        if self.ultra_high_resolution:
            # Use iTerm2's image protocol for ultra-high resolution
            return self.ultra_high_res_frame_to_terminal(frame)
        else:
            # Process two rows at a time using half-block characters
            output = []
            for y in range(0, height-1, 2):
                line = ""
                for x in range(width):
                    # Get colors for upper and lower pixels
                    upper_color = frame[y, x]
                    lower_color = frame[y+1, x] if y+1 < height else np.zeros(3, dtype=np.uint8)
                    
                    # Create ANSI color codes
                    upper_ansi = f"\x1b[38;2;{upper_color[2]};{upper_color[1]};{upper_color[0]}m"
                    lower_ansi = f"\x1b[48;2;{lower_color[2]};{lower_color[1]};{lower_color[0]}m"
                    
                    # Combine using half-block character
                    line += f"{upper_ansi}{lower_ansi}▀"
                
                output.append(line + "\x1b[0m")
            
            return "\n".join(output)
        

        
    def ultra_high_res_frame_to_terminal(self, frame):
        """Generate ultra-high resolution output using iTerm2's image protocol"""
        
        # Convert to RGB color space
        pil_img = Image.fromarray(cv2.cvtColor(frame, cv2.COLOR_BGR2RGB))
        
        # For smoother playback, significantly reduce the image size
        # This is a compromise for better performance
        max_dimension = 1000 # Lower resolution for smoother playback
        
        # Always resize to improve performance, even if the original is smaller
        current_width, current_height = pil_img.size
        if current_width > current_height:
            new_size = (max_dimension, int(current_height * max_dimension / current_width))
        else:
            new_size = (int(current_width * max_dimension / current_height), max_dimension)
            
        # Use bicubic instead of Lanczos - it's faster with only slightly lower quality
        pil_img = pil_img.resize(new_size, Image.Resampling.BICUBIC)
        
        # Use JPEG with lower quality setting for much faster encoding
        with io.BytesIO() as bio:
            pil_img.save(bio, format='JPEG', quality=80)  # Lower quality for better performance
            img_bytes = bio.getvalue()
            
        # Encode the image directly to base64
        base64_data = base64.b64encode(img_bytes).decode('ascii')
        
        # Get terminal dimensions
        term_width, term_height = struct.unpack('HHHH', 
            fcntl.ioctl(sys.stdout.fileno(), termios.TIOCGWINSZ, 
            struct.pack('HHHH', 0, 0, 0, 0)))[:2]
        
        # Use even smaller width for smoother performance
        pixel_width = min(term_width * 6, 1000)  # Lower resolution cap
        
        # Use a simple width specification for better performance
        spec = f"width={pixel_width}"
        
        # Use the iTerm2 protocol with simpler specifications
        image_protocol = f"\x1b]1337;File=inline=1;{spec};preserveAspectRatio=1:{base64_data}\x07"
        
        return image_protocol

    def play_audio(self):
        command = [
            "ffplay", "-nodisp", "-autoexit", "-loglevel", "quiet",
            self.video_path
        ]
        self.audio_process = subprocess.Popen(command, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)

    def frame_producer(self):
        # For ultra high resolution mode, we might need to skip frames to maintain sync
        # when playback falls behind
        frame_count = 0
        frames_to_skip = 0
        
        # For ultra-high resolution mode, implement frame dropping right from the start
        # to ensure smooth playback
        frame_drop_interval = 1
        if self.ultra_high_resolution:
            # For ultra high res, process every other frame (effectively halving the frame rate)
            # for smoother playback
            frame_drop_interval = 2
        
        while not self.stop_event.is_set():
            ret, frame = self.cap.read()
            if not ret:
                break
                
            frame_count += 1
            
            # In ultra-high resolution mode, systematically drop frames to reduce processing load
            if self.ultra_high_resolution:
                # Only process every Nth frame (where N is frame_drop_interval)
                if frame_count % frame_drop_interval != 0:
                    continue
                    
                # Additional adaptive frame dropping when queue gets full
                if self.frame_queue.qsize() > 15:  # More aggressive queue management
                    # Skip more frames when queue is filling up
                    continue
            
            # Process the frame
            frame = self.resize_frame(frame)
            
            # Only put the frame if the queue isn't full
            # This prevents excessive memory usage
            try:
                self.frame_queue.put(frame, block=False)
            except:
                # If the queue is full, just continue with the next frame
                pass
            
        # Signal the end of the video
        self.frame_queue.put(None)

    def play(self):
        print(f"\x1b[?25l")  # Hide cursor
        print("\x1b[2J")  # Clear screen completely
        print("\x1b[H")   # Move cursor to top-left
        
        # Display video information and rendering mode
        mode = "Standard (Half-block)" 
        if self.ultra_high_resolution:
            mode = "Ultra High Resolution (iTerm2 Image Protocol)"
            # Only skip the text output, but don't return
            pass
        else:
            print(f"\nVideo Info: {self.orig_width}x{self.orig_height} @ {self.fps:.2f} FPS")
            print(f"Rendering Mode: {mode}\n")
        
        # For ultra high resolution mode, adjust the effective frame rate for smoother playback
        effective_fps = self.fps
        if self.ultra_high_resolution:
            # Use a lower effective frame rate for smoother playback
            effective_fps = min(self.fps, 24)  # Cap at 24 FPS for smoother playback
            self.frame_time = 1/effective_fps  # Recalculate frame time
        
        # Start audio playback
        audio_thread = threading.Thread(target=self.play_audio)
        audio_thread.start()
        
        # Start frame producer
        producer_thread = threading.Thread(target=self.frame_producer)
        producer_thread.start()
        
        try:
            start_time = time.time()
            frame_count = 0
            frames_behind = 0  # Track how many frames we're behind
            
            # For ultra high resolution mode, use a more aggressive frame skipping threshold
            skip_threshold = 2
            if self.ultra_high_resolution:
                skip_threshold = 1  # More aggressive frame skipping
            
            while True:
                frame = self.frame_queue.get()
                if frame is None:
                    break
                    
                # Calculate timing
                frame_count += 1
                target_time = start_time + frame_count * self.frame_time
                current_time = time.time()
                
                # Check if we're behind schedule
                if current_time > target_time:
                    # We're running behind
                    frames_behind += 1
                    
                    # If we're significantly behind, start skipping frames to catch up with the audio
                    if self.ultra_high_resolution and frames_behind > skip_threshold:
                        # Skip frames more aggressively in ultra high resolution mode
                        if self.frame_queue.qsize() > 0:
                            continue
                            
                    # Reset target time to avoid falling further behind
                    target_time = current_time
                else:
                    # We're on schedule or ahead
                    frames_behind = 0  # Reset the behind counter
                    time.sleep(target_time - current_time)
                
                # Clear screen and display frame
                sys.stdout.write("\x1b[H")  # Move to top-left
                sys.stdout.write(self.frame_to_terminal(frame))
                sys.stdout.flush()
                
        except KeyboardInterrupt:
            pass
        finally:
            self.cleanup()

    def cleanup(self):
        self.stop_event.set()
        if self.audio_process:
            self.audio_process.terminate()
        self.cap.release()
        print("\x1b[?25h")  # Show cursor
        print("\x1b[0m")    # Reset colors
        
        # iTerm2 specific: Clear scrollback buffer to free memory from large images
        if self.ultra_high_resolution:
            print("\x1b[3J", end="")  # Clear scrollback buffer
            
        os.system('clear')  # Clear screen

def main():
    parser = argparse.ArgumentParser(description='Terminal Video Player')
    parser.add_argument('video_path', help='Path to the video file')
    parser.add_argument('-n', '--native', action='store_true',
                      help='Use native resolution mode')
    parser.add_argument('-uhr', '--ultra-high-resolution', action='store_true',
                      help='Enable ultra high resolution mode using iTerm2 image protocol')
    args = parser.parse_args()

    if not os.path.exists(args.video_path):
        print(f"Error: Video file '{args.video_path}' not found.")
        sys.exit(1)

    # Clear the screen and move to top
    os.system('clear')
    print("\x1b[H")
    
    # Ultra high resolution mode automatically uses optimized playback
    if args.ultra_high_resolution:
        print("Ultra high resolution mode uses optimized playback settings")
        print("(Lower resolution but higher frame rate)")
        time.sleep(1)  # Brief pause to show message
        os.system('clear')  # Clear the message
    
    player = TerminalPlayer(
        args.video_path, 
        args.native,
        args.ultra_high_resolution
    )
    player.play()

if __name__ == "__main__":
    main()
